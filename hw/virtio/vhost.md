# vring

## 控制面

### open /dev/vhost-net



### vhost_dev_init

VHOST_SET_OWNER

内核会创建vhost-[pid]工作线程。

### vhost_ack_features

VHOST_SET_FEATURES

### vhost_dev_enable_notifiers

禁用ioeventfd，禁止qemu中监听host_notifier。

### vhost_dev_start

VHOST_SET_MEM_TABLE

对每一个VirtQueue：

- VHOST_SET_VRING_NUM
- VHOST_SET_VRING_BASE
- VHOST_SET_VRING_ADDR
- VHOST_SET_VRING_KICK：guest写数据到VirtQueue之后，写Notify寄存器，kick，vhost开始读取数据。
- VHOST_SET_VRING_CALL：vhost处理完数据后，写入used ring，call，发送中断通知驱动。

### vhost_net_set_backend

VHOST_NET_SET_BACKEND

## 数据面

qemu中无代码，完全在内核实现。

qemu中只有控制面代码。