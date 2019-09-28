# 1. virtio简介

详细请参考[virtio-v1.0.pdf](<http://docs.oasis-open.org/virtio/virtio/v1.0/virtio-v1.0.pdf>)

术语：

- 驱动：指虚拟机内的virtio驱动程序。
- 设备：只qemu中实现的virtio设备。

所有的virtio设备都包含4部分：

- Device status field
- Feature bits
- Device Configuration space
- One or more virtqueues

## 1.1 Device Status Field

status字段用于指示驱动初始化设备的进度。

- **ACKNOWLEDGE (1)** 指示guest os发现了设备，并且是一个有效的virtio设备。
- **DRIVER (2)** 指示guest os知道如何驱动这个设备。
- **FEATURES_OK (8)** 指示驱动完成了feature的协商。
- **DRIVER_OK (4)** 指示驱动已经完成，设备可以正常工作了。

## 1.2 Feature Bits

特性协商，用于兼容。新的设备可以适配旧的驱动，新的驱动也可以适配旧的设备。

Feature bits are allocated as follows:
**0 to 23** Feature bits for the specific device type
**24 to 32** Feature bits reserved for extensions to the queue and feature negotiation mechanisms
**33 and above** Feature bits reserved for future extensions.

## 1.3 Device Configuration Space

每一类virtio设备都有不同大小的配置空间，这个配置空间跟pci设备的配置空间有一定的差别。

pci设备配置空间是通过0xcf8和0xcfc端口来访问，只有256字节大小。

而virtio设备的配置空间则是以pci设备的BAR空间存在的，可以是IO BAR空间，也可以是MEM BAR空间，大小不固定。

virtio设备的配置空间布局包含几个部分：

- Common configuration
- Notifications
- ISR Status
- Device-specific configuration (optional)
- PCI configuration access

每一部分都可以独立存在，每一部分存在与否需要通过vendor-specific pci capability来发现，这个capability存在pci的配置空间中。

```c
struct virtio_pci_cap {
    u8 cap_vndr; /* Generic PCI field: PCI_CAP_ID_VNDR */
    u8 cap_next; /* Generic PCI field: next ptr. */
    u8 cap_len; /* Generic PCI field: capability length */
    u8 cfg_type; /* Identifies the structure. */
    u8 bar; /* Where to find it. */
    u8 padding[3]; /* Pad to full dword. */
    le32 offset; /* Offset within bar. */
    le32 length; /* Length of the structure, in bytes. */
};
```

***cap_vndr*** 0x09; Identifies a vendor-specific capability.

***cap_next*** Link to next capability in the capability list in the PCI configuration space.

***cap_len*** Length of this capability structure, including the whole of struct virtio_pci_cap, and extra data if
any. This length MAY include padding, or fields unused by the driver.

***cfg_type*** identifies the structure, according to the following table:

- ```c
  /* Common configuration */
  #define VIRTIO_PCI_CAP_COMMON_CFG 1
  /* Notifications */
  #define VIRTIO_PCI_CAP_NOTIFY_CFG 2
  /* ISR Status */
  #define VIRTIO_PCI_CAP_ISR_CFG 3
  /* Device specific configuration */
  #define VIRTIO_PCI_CAP_DEVICE_CFG 4
  /* PCI configuration access */
  #define VIRTIO_PCI_CAP_PCI_CFG 5
  ```

- 标记每一部分

***bar*** values 0x0 to 0x5 specify a Base Address register (BAR) belonging to the function located beginning
at 10h in PCI Configuration Space and used to map the structure into Memory or I/O Space. The BAR
is permitted to be either 32-bit or 64-bit, it can map Memory Space or I/O Space.

***offset*** indicates where the structure begins relative to the base address associated with the BAR. The
alignment requirements of offset are indicated in each structure-specific section below.

***length*** indicates the length of the structure.

virtio设备的配置空间的每一部分存在在BAR空间中，可以存放到同一个bar空间，每一部分的存在及位置则由pci capability指定。在分析qemu的实现时，不要搞混淆了。

### 1.3.1 Common configuration structure layout

```c
struct virtio_pci_common_cfg {
    /* About the whole device. */
    le32 device_feature_select; /* read-write */
    le32 device_feature; /* read-only for driver */
    le32 driver_feature_select; /* read-write */
    le32 driver_feature; /* read-write */
    le16 msix_config; /* read-write */
    le16 num_queues; /* read-only for driver */
    u8 device_status; /* read-write */
    u8 config_generation; /* read-only for driver */
    /* About a specific virtqueue. */
    le16 queue_select; /* read-write */
    le16 queue_size; /* read-write, power of 2, or 0. */
    le16 queue_msix_vector; /* read-write */
    le16 queue_enable; /* read-write */
    le16 queue_notify_off; /* read-only for driver */
    le64 queue_desc; /* read-write */
    le64 queue_avail; /* read-write */
    le64 queue_used; /* read-write */
};
```

***device_feature_select*** The driver uses this to select which feature bits *device_feature* shows. Value 0x0 selects Feature Bits 0 to 31, 0x1 selects Feature Bits 32 to 63, etc.

***device_feature*** The device uses this to report which feature bits it is offering to the driver: the driver writes to *device_feature_select* to select which feature bits are presented.

***driver_feature_select*** The driver uses this to select which feature bits *driver_feature* shows. Value 0x0 selects Feature Bits 0 to 31, 0x1 selects Feature Bits 32 to 63, etc.

***driver_feature*** The driver writes this to accept feature bits offered by the device. Driver Feature Bits selected by *driver_feature_select*.

***config_msix_vector*** The driver sets the Configuration Vector for MSI-X.

***num_queues*** The device specifies the maximum number of virtqueues supported here.

***device_status*** The driver writes the device status here (see 2.1). Writing 0 into this field resets the device.

***config_generation*** Configuration atomicity value. The device changes this every time the configuration
noticeably changes.

***queue_select*** Queue Select. The driver selects which virtqueue the following fields refer to.

***queue_size*** Queue Size. On reset, specifies the maximum queue size supported by the hypervisor. This
can be modified by driver to reduce memory requirements. A 0 means the queue is unavailable.

***queue_msix_vector*** The driver uses this to specify the queue vector for MSI-X.

***queue_enable*** The driver uses this to selectively prevent the device from executing requests from this
virtqueue. 1 - enabled; 0 - disabled.

***queue_notify_off*** The driver reads this to calculate the offset from start of Notification structure at which
this virtqueue is located.

***queue_desc*** The driver writes the physical address of Descriptor Table here.

***queue_avail*** The driver writes the physical address of Available Ring here.

***queue_used*** The driver writes the physical address of Used Ring here.

这一部分，主要是virtio设备的通用配置，包含特性协商，status，virtqueue的配置。

### 1.3.2 Notification structure layout

The notification location is found using the VIRTIO_PCI_CAP_NOTIFY_CFG capability. This capability is
immediately followed by an additional field, like so:

```c
struct virtio_pci_notify_cap {
    struct virtio_pci_cap cap;
    le32 notify_off_multiplier; /* Multiplier for queue_notify_off. */
};
```
*notify_off_multiplier* is combined with the *queue_notify_off* to derive the Queue Notify address within a BAR
for a virtqueue:

```c
cap.offset + queue_notify_off * notify_off_multiplier
```
The *cap.offset* and *notify_off_multiplier* are taken from the notification capability structure above, and the
*queue_notify_off* is taken from the common configuration structure.

通知结构布局，主要是virtio设备在virtqueue写入数据后，来通知qemu可以读了。每个virtqueue都会有一个通知寄存器，其位置通过上面的公式计算，如果`notify_off_multiplier = 0`则所有virtqueue共享一个通知寄存器。

### 1.3.3 ISR status capability

The VIRTIO_PCI_CAP_ISR_CFG capability refers to at least a single byte, which contains the 8-bit ISR
status field to be used for INT#x interrupt handling.

The ISR bits allow the device to distinguish between device-specific configuration change interrupts and
normal virtqueue interrupts:

| Bits        | 0               | 1                              | 2 to 31  |
| ----------- | --------------- | ------------------------------ | -------- |
| **Purpose** | Queue Interrupt | Device Configuration Interrupt | Reserved |

virtio设备发出中断来通知virtqueue已经有数据，需要驱动读取。

在启用MSI-x时，ISR未使用。

### 1.3.4 Device-specific configuration

VIRTIO_PCI_CAP_DEVICE_CFG

设备指定的配置，每个设备都有自己特殊的参数。

### 1.3.5 PCI configuration access capability

The VIRTIO_PCI_CAP_PCI_CFG capability creates an alternative (and likely suboptimal) access method
to the common configuration, notification, ISR and device-specific configuration regions.
The capability is immediately followed by an additional field like so:

```c
struct virtio_pci_cfg_cap {
    struct virtio_pci_cap cap;
    u8 pci_cfg_data[4]; /* Data for BAR access. */
};
```

The fields cap.bar, cap.length, cap.offset and pci_cfg_data are read-write (RW) for the driver.
To access a device region, the driver writes into the capability structure (ie. within the PCI configuration
space) as follows:

- The driver sets the BAR to access by writing to cap.bar.
- The driver sets the size of the access by writing 1, 2 or 4 to cap.length.
- The driver sets the offset within the BAR by writing to cap.offset.

At that point, pci_cfg_data will provide a window of size cap.length into the given cap.bar at offset cap.offset.

### 1.3.6 Legacy Interfaces

| Bits                | 32                      | 32                      | 32           | 16          | 16            | 16          | 8            | 8         |
| ------------------- | ----------------------- | ----------------------- | ------------ | ----------- | ------------- | ----------- | ------------ | --------- |
| **Read/Write** | R                       | R+W                     | R+W          | R           | R+W           | R+W         | R+W          | R         |
| **Purpose**         | Device Features bits 0:31 | Driver Features bits 0:31 | Queue Address | queue_size | queue_select | Queue Notify | Device Status | ISR Status |

大部分字段同之前介绍的。

## 1.4 virtqueues

virtqueue用于传递数据，在虚拟机和宿主机之间。每个设备可以包含0个或多个virtqueue.

Each virtqueue consists of three parts:

- Descriptor Table

- Available Ring

- Used Ring

The memory aligment and size requirements, in bytes, of each part of the virtqueue are summarized in the
following table:

| Virtqueue Part | Alignment | Size |
| --- | --- | --- |
| Descriptor Table | 16 | 16\*(Queue Size) |
| Available Ring | 2 | 6 + 2\*(Queue Size) |
| Used Ring | 4 | 6 + 8\*(Queue Size) |

virtqueue的整体数据结构：

```c
#define ALIGN(x) (((x) + qalign) & ~qalign)
static inline unsigned virtq_size(unsigned int qsz)
{
	return ALIGN(sizeof(struct virtq_desc)*qsz + sizeof(u16)*(3 + qsz))
		+ ALIGN(sizeof(u16)*3 + sizeof(struct virtq_used_elem)*qsz);
}
struct virtq {
    // The actual descriptors (16 bytes each)
    struct virtq_desc desc[ Queue Size ];
    // A ring of available descriptor heads with free-running index.
    struct virtq_avail avail;
    // Padding to the next Queue Align boundary.
    u8 pad[ Padding ];
    // A ring of used descriptor heads with free-running index.
    struct virtq_used used;
};
struct virtq_desc {
    /* Address (guest-physical). */
    le64 addr;
    /* Length. */
    le32 len;
    /* This marks a buffer as continuing via the next field. */
    #define VIRTQ_DESC_F_NEXT 1
    /* This marks a buffer as device write-only (otherwise device read-only). */
    #define VIRTQ_DESC_F_WRITE 2
    /* This means the buffer contains a list of buffer descriptors. */
    #define VIRTQ_DESC_F_INDIRECT 4
    /* The flags as indicated above. */
    le16 flags;
    /* Next field if flags & NEXT */
    le16 next;
};
struct indirect_descriptor_table {
	/* The actual descriptors (16 bytes each) */
	struct virtq_desc desc[len / 16];
};
struct virtq_avail {
    #define VIRTQ_AVAIL_F_NO_INTERRUPT 1
    le16 flags;
    le16 idx;
    le16 ring[ /* Queue Size */ ];
    le16 used_event; /* Only if VIRTIO_F_EVENT_IDX */
};
struct virtq_used {
    #define VIRTQ_USED_F_NO_NOTIFY 1
    le16 flags;
    le16 idx;
    struct virtq_used_elem ring[ /* Queue Size */];
    le16 avail_event; /* Only if VIRTIO_F_EVENT_IDX */
};
/* le32 is used here for ids for padding reasons. */
struct virtq_used_elem {
    /* Index of start of used descriptor chain. */
    le32 id;
    /* Total length of the descriptor chain which was used (written to) */
    le32 len;
};
```

每个virtqueue都有一个队列大小，由`Queue Size`表示，循环队列。

## 1.5 设备初始化流程

设备的配置空间，主要完成设备的控制，属于控制面，包含feature协商，配置空间发现，virtqueue的初始化，中断配置，status配置。

virtqueues，主要完成数据的传递，属于数据面。

### 1.5.1 初始化流程

1. 复位设备
2. 设置 ACKNOWLEDGE status bit。
3. 设置 DRIVER status bit: guest OS 知道如何驱动设备.
4. feature协商，完成后设置FEATURES_OK status bit.
5. 执行设备特定的配置：MSI-x配置，virtqueue初始化。
6. 设置 DRIVER_OK status bit。

### 1.5.2 virtqueue初始化

一个设备可以有0个或多个virtqueue，而且virtqueue不是在pci设备的BAR空间中，而且存在于guest的内核地址空间中。

驱动从Common configuration(1.3.1)中读取*num_queues*字段，确定设备拥有的virtqueue的数量，每一个virtqueue都有一个队列大小。

对每一个virtqueue：

1. 写 virtqueue index 到 *queue_select* 寄存器。从0开始。
2. 读取队列大小 *queue_size* 寄存器 。
3. 可选的，驱动可以选择一个较小的queue_size，写入queue_size。
4. 在驱动中分配virtq_desc结构体，把其物理地址写入 *queue_desc* 寄存器。这里的物理地址，是guest物理地址*gpa*。
5. 在驱动中分配virtq_avail结构体，把其物理地址写入 *queue_avail* 寄存器。这里的物理地址，是guest物理地址*gpa*。
6. 在驱动中分配virtq_used结构体，把其物理地址写入 *queue_used* 寄存器。这里的物理地址，是guest物理地址*gpa*。
7. 如果启用MSI-x，则选择一个vector写入 *queue_msix_vector* 寄存器。
8. 确定virtqueue notify的地址：读取 *queue_notify_off* 寄存器，根据`virtio_pci_notify_cap.offset + queue_notify_off * notify_off_multiplier`计算出此virtqueue的通知地址。
9. 启动virtqueue：向 *queue_enable* 寄存器写入1。至此virtqueue已经配置好。

### 1.5.3 Notifying The Device

驱动写 virtqueue index 到 Queue Notify 寄存器，以通知设备一个virtqueue已经存入数据。

# virtio

https://github.com/zhangjaycee/real_tech/wiki/virtual_008

# 2. virtio-pci(qemu源码分析)

每一个virtio相关的设备，都会通过pci设备来呈现。在qemu中，一个设备内会嵌入三个对象：

```
VirtIOPCIProxy
	VirtioBusState
VirtIODevice
```

VirtIOPCIProxy对外暴露pci接口，guest对virtio设备的探测及读写都通过pci总线来进行。

同时这个设备也是一个virtio总线桥设备，负责把pci总线转换成virtio总线。总线对象由内嵌在VirtIOPCIProxy中的VirtioBusState来表示。一条virtio总线上只能挂一个virtio设备，每个virtio设备都派生自VirtIODevice。

读写virtio-pci设备的配置空间，BAR空间，都先到达VirtIOPCIProxy设备，再由这个设备转发到virtio总线上，最终到达VirtIODevice进行处理。

看一个例子：

```c
struct VirtIOPCIProxy {
    PCIDevice pci_dev;
	......
    VirtioBusState bus;
};
typedef struct VirtIONet {
    VirtIODevice parent_obj;
	......
} VirtIONet;
struct VirtIONetPCI {
    VirtIOPCIProxy parent_obj;
    VirtIONet vdev;
};
static const TypeInfo virtio_net_pci_info = {
    .name          = TYPE_VIRTIO_NET_PCI,  //"virtio-net-pci"
    .parent        = TYPE_VIRTIO_PCI,
    .instance_size = sizeof(VirtIONetPCI),
    .instance_init = virtio_net_pci_instance_init,
    .class_init    = virtio_net_pci_class_init,
};
```

"virtio-net-pci" 这个设备，对应的实例是VirtIONetPCI，其是由VirtIOPCIProxy和VirtIONet组成，VirtIOPCIProxy内嵌了VirtioBusState，VirtIONet派生自VirtIODevice。由3个对象共同对guest呈现一个virtio设备。

## 2.1 整体架构

`hw/virtio/virtio-pci.c`中包含了VirtIOPCIProxy和VirtioBusState 2个类型。

`hw/virtio/virtio.c`中包含了VirtIODevice的实现。

一个核心原则：对VirtIOPCIProxy设备的读写，都会转换成对VirtIODevice的读写。包括控制面和数据面。

## 2.2 realized

### 2.2.1 pci设备realized

1. 调用virtio_pci_bus_new()完成对virtio总线的初始化和realized。
2. 调用`object_property_set_bool(OBJECT(vdev), true, "realized", errp);`完成对内嵌的VirtIODevice对象realized。

### 2.2.2 virtio设备realized

1. `virtio_init`完成对VirtIODevice的初始化，分配Device-specific配置空间(1.3.4)，VirtQueue初始化(一个设备最多可以有1024个VirtQueue)。

2. `virtio_add_queue`

   ```c
   VirtQueue *virtio_add_queue(VirtIODevice *vdev, int queue_size,
                               VirtIOHandleOutput handle_output)
   ```

   给VirtIODevice添加一个VirtQueue，*queue_size*指定队列大小，*handle_output*指定virtqueue有数据时如何处理。主要是驱动向virtqueue写入数据后，会写 virtqueue index 到 Queue Notify 寄存器(1.5.3)，然后qemu中就会收到通知，然后调用handle_output读取virtqueue上的数据。

3. 其他设备指定的实现。

### 2.2.3 device_plugged

在virtio设备realized之后，会通知virtio总线，设备plugged。在virtio_device_realize()中会调用virtio_bus_device_plugged()，最终会调用到virtio_pci_device_plugged()函数，该函数完成对pci设备的legacy和modern BAR空间的注册。

在virtio设备plugged设备之后，通过virtio总线来通知PCI设备，PCI设备才真正完成初始化。

### 2.3.4 pci设备对外接口

#### legacy

当驱动读写PCI设备的IO BAR空间时，guest会退出到kvm模块，kvm模块再把io读写请求转发到qemu中，qemu调用下面2个函数完成io请求。

- virtio_pci_config_read：完成对virtio配置空间的读，virtio_ioport_write()和virtio_config_readl()。
- virtio_pci_config_write：完成对virtio配置空间的写，virtio_ioport_write()和virtio_config_writel()。

#### modern

当驱动读写PCI设备的MEM BAR空间时，guest会退出到kvm模块，kvm模块再把MMIO读写请求转发到qemu中，qemu区分MMIO属于哪一部分，分别调用不同的函数完成MMIO请求。

- Common configuration structure layout
  - virtio_pci_common_read：读。
  - virtio_pci_common_write：写。
- Notification structure layout
  - virtio_pci_notify_read：读，空实现。
  - virtio_pci_notify_write：写，调用virtio_queue_notify，最终调用VirtQueue的*handle_output*来处理。
- ISR status capability
  - virtio_pci_isr_read：读取isr寄存器。
  - virtio_pci_isr_write：空实现。
- Device-specific configuration
  - virtio_pci_device_read：读config
  - virtio_pci_device_write：写config。

## 2.3 驱动初始化

### 2.3.1 协商feature

- virtio_set_features

### 2.3.2 发现及配置virtqueue

- virtio_queue_set_num
- virtio_queue_set_rings

### 2.3.3 DRIVER_OK

- virtio_set_status
- virtio_pci_start_ioeventfd

## 2.4 正常读写设备

### 2.4.1 在virtqueue中分配elem



### 2.4.2 guest向host的notify

guest向virtqueue中写入数据后，会写 virtqueue index 到 Queue Notify 寄存器(1.5.3)，此就是notify。

数据的处理有多种方式：

- legacy：数据读取完全在qemu的用户态实现。

  1. guest写 Queue Notify 寄存器。

  2. kvm因为io写退出到qemu中。

  3. ```c
         case VIRTIO_PCI_QUEUE_NOTIFY:
             if (val < VIRTIO_QUEUE_MAX) {
                 virtio_queue_notify(vdev, val);
             }
             break;
     ```

  4. qemu调用virtio_queue_notify，最终调用VirtQueue的*handle_output*来处理。

- ioeventfd：

- vhost：

# virtio-net

## 控制面

## 数据面

# virtio-blk

## 控制面

## 数据面
