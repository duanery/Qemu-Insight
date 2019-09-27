# virtio简介

status

features

配置空间

virtqueues

设备初始化流程

# virtio

https://github.com/zhangjaycee/real_tech/wiki/virtual_008

# virtio-pci

每一个virtio相关的设备，都会通过pci设备来呈现。杂糅起来，一个设备内会嵌入三个对象：

```
VirtIOPCIProxy
	VirtioBusState
VirtIODevice
```

VirtIOPCIProxy对外暴露pci接口，虚拟机对virtio设备的探测及读写都通过pci总线来进行。

同时这个设备也是一个virtio总线桥设备，负责把pci总线转换成virtio总线。总线由内嵌在VirtIOPCIProxy中的VirtioBusState来表示。一条virtio总线上只能挂一个virtio设备，每个virtio设备都派生自VirtIODevice。

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

"virtio-net-pci" 这个设备，对应的实例是VirtIONetPCI，其是由VirtIOPCIProxy和VirtIONet组成，VirtIOPCIProxy内嵌了VirtioBusState，VirtIONet派生自VirtIODevice。

## realized

### pci对外接口

#### legacy



#### modern

- Common configuration structure layout

- Notification structure layout
- ISR status capability
- Device-specific configuration

### virtio设备realized



## 驱动初始化

### 协商feature

### 发现及配置virtqueue

### DRIVER_OK

- virtio_set_status

## 正常读写设备

### 在virtqueue中分配elem



### notify

- legacy
- ioeventfd
- vhost



# virtio-net

## 控制面

## 数据面

# virtio-blk

## 控制面

## 数据面
