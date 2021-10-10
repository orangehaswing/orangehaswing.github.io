---
layout: post
title: kernel_virtio驱动 源码分析
date: 2021-10-10
tags: jekyll
---

## 驱动加载过程

每个 virtio 驱动都有对应的 virtio 设备，例如 virtio-net，virtio-blk 等。当 kernel 启动，会调用 PCI 枚举过程，并且会调用相应的驱动 probe 完成设备初始化加载过程。

```c
drivers\virtio\virtio_pci_common.c 
static struct pci_driver virtio_pci_driver = { 
	.name     	= 	"virtio-pci", 
	.id_table  	= 	virtio_pci_id_table, 
	.probe    	= 	virtio_pci_probe, 
	.remove   	= 	virtio_pci_remove, 
	.sriov_configure	=	virtio_pci_sriov_comfigure, 
}; 
```

virtio_pci_driver 结构体定义了 virtio_pci_probe 函数

```c
drivers\virtio\vírtio_pci_common.c 
virtio_pci_probe 
	// 分配virtio_pci_device结构体
	vp_dev = kzalloc(sízeof(struct virtie_pci_device), GFP_XERNEL) 
	
	// 设置vde相关结构
	pci_set_drvdata(pcí_dev, vp_dev) 
	vp_dev->vdev.dev.parent = &pc1_dev.dev 
	vp_dev->vdev.dev.release « virtio_pc:_nelease_dev: 
	vp_dev->pcí_dev = pci_dev; 
	
	//使能PCI设备
	pci_enable_device(pci_dev) 
	
	//初始化PCI设备对应的virtio设备
	virtio_pci_modern_probe(vp_dev) 
	
	// 将virtio设备注册到系统
	register_virtio_device(&vp_dev->vdev) 
```

## 初始化PCI设备

```c
drivers\pci\pci.c 
pci_enable_device 
	pci_enable_device_flags(dev, IORESOURCE_MEM|IORESOURCE_IO) 
```

### 初始化virtio设备

```c
drivers\virtio\virtio_pci_modern.c 
virtio_pci_modern_probe 
	// 设置设备的device id和vendor id，这里的base是0x1040，就是qemu里面的base id
	vp_dev->vdev.id.device = pci_dev->device - 0x1040;
    vp_dev->vdev.id.vendor = pci_dev->subsystem_vendor; 
     
	// 调用find函数，从配置空间找到pci capability数据，这部分在qemu里面通过注册对应memory region信息保存
	common = virtio_pci_find_capability 
	isr = virtio_pci_find_capability 
	notify = virtio_pci_find_capability 
	device = virtio_pci_find_capability 
	
	// 调用map_capability，将virtio设备的bar空间映射到内核地址空间
	vp_dev->common = map_capability 
	vp_dev->isr = map_capability 
	vp_dev->notify_base = map_capability 
	vp_dev->device = map_capability 
	
	// 设置pci config 和 MSIX 
	vp_dev->vdev.config = &virtio_pci_config_ops; 
	vp_dev->config_vector = vp_config_vector; 
	
	// 用来设置和删除virtio设备的virtiqueue
	vp_dev->setup_vq = setup_vq; 
	vp_dev->del_vq = del_vq; 
```

virtio_pci_modern_probe 函数首先设置 virtio 设备的 vendor ID 和 device ID。 

然后多次调用 virtio_pci_find_capability 函数，发现 qemu 后端设备的 pci capability 配置空间数据。接着找到所属的 PCI BAR，把数据写入到对应的 bar region 中。将这些 bar 地址空间和类型保存起来，等待 kernel 启动阶段激活设备。

接着调用 map_capability，映射 virtio 设备对应的 capability 在 PCI 设备的 bar 空间映射到内核地址空间。后续内核 CPU 可以直接访问这段内核地址，达到访问virtio设备配置和控制设备输出传输的目的。

最后将 PCI config 数据保存，并保存回调函数和 MSIX 相关的配置， virtio queue 数据。


## virtio pci device结构 

```c
struct virtio_pci_device {
	// 描述设备virtio属性
	virtio_device 
		id 
		config 
		vqs 
		features 

	// 描述PCI 设备结构
	pci_dev 
		bus 
		slot 
		devfn 
		vendor 

	// bar region结构，内容存放在mmio
	isr 
	common 
	device 
	notify_base 
	ioaddr 

	// virtio队列
	vqs 

	// misx中断标记位和中断向量
	msix_enabled 
	msix_vectors 

	// 添加或删除virtiqueue函数
	setup_vq 
	del_vq 
}
```

## 注册virtio设备

```c
drivers\virtio\virtio.c 
register_virtio_device 
	// 设置virtio设备BUS为virtio_bus，并初始化virtio_bus 
	dev->dev.bus =&virtio_bus 
	device_initialize(&dev->dev) 

	// 设置virtio设备名
	dev_set_name(&dev->dev, "virtio%u", dev->index) 

	// 重置设备
	dev->config->reset(dev) 
	
	// 设置virtio设备状态
	virtio_add_status(dev, VIRTIO_CONFIG_S_ACKNOWLEDGE) 

	// 将设备注册到系统
	device_add(&dev->dev) 
		virtio_dev_probe 
		virtnet_probe 
```

register_virtio_device 函数设置了 virtio 设备的 BUS，并在初始化过程注册到系统中。调用回调函数重置 virtio 设备。

最后调用 device_add，将设备加载到系统。这个过程会发送一个 uevent 消息到用户空间，这个 uevent 消息包含
virtio 设备的 vendor id， device id。udev 接收到这个消息，会加载 virtio 设备对应的驱动。然后调用
bus_probe_device 函数，最终调用 BUS 的 probe 和设备的 probe，也就是 virtio_dev_probe 函数和 virtnet_probe 
 函数。

设备初始化过程过程如下：

1. 重置设备，调用 dev-＞config-＞reset 回调函数

2. 设置 VIRTIO_CONFIG_S_ACKNOWLEDGE 状态标记位，后端 virtio 设备会收到该标记位。表示 virtio 驱动已经知道该设备

3. 设置 DRIVER 状态位，表示 virtio 设备使用什么样的驱动，该过程在 virtio bus probe 过程中完成。调用函数是 virtio_dev_probe 函数 add_status 

   ```c
   virtio_dev_probe 
   	virtio_add_status(dev, VIRTIO_CONFIG_S_DRIVER) 
   ```

4. 读取 virtio 设备 feature 状态位，求出 virtio 驱动 feature，取两者的子集，然后向后端写入子集。

   ```c
   virtnet_probe 
   	dev->features = NETIF_F_HIGHDMA; 
   	if (virtio_has_feature(vdev, VIRTIO_NET_F_CSUM) { 
   		...
   	} 
   	if (virtio_has_feature(vdev, VIRTIO_NET_F_GUEST_CSUM))
   	...
   	if (virtio_has_feature(vdev, VIRTIO_NET_F_MRG_RXBUF))
   	...
   ```

5. 设置 FEATURES_OK 特性位。在这之后，virtio 驱动就不会再接收到新的特性了。

   ```c
   drivers\virtio\virtio.c 
   virtio_finalize_features 
   	virtio_add_status(dev, VIRTIO_CONFIG_S_FEATURES_OK) 
   ```

6. 重新读取设备 feature 位，确保设置 FEEATURES_OK 标记位。

   ```c
   virtio_finalize_features 
   	status = dev->config->get_status(dev) 
   	if (!(status & VIRTIO_CONFIG_S_FEATURES_OK)) { 
   		...
   	}
   ```

## virtio 驱动初始化

### 读写 mmio region 

上述 virtio_pci_modern_probe 过程中，完成了变量赋值 vp_dev-＞vdev.config ＝ ＆virtio_pci_config_ops；

virtio_pci_config_ops 结构体如下：

```c
drivers\virtio\virtio_pci_modern.c 
static const struct virtio_config_ops virtio_pci_config_ops = { 
	·get    =	vp_get, 
	.set    =	vp_set, 
	.generation = vp_generation, 
	.get_status = vp_get_status, 
	.set_status =	vp_set_status, 
	.reset   =	vp_reset, 
	.find_vqs  =	vp_modern_find_vqs, 
	.del_vqs   =	vp_del_vqs, 
	.get_features  =	vp_get_features, 
	.finalize_features = vp_finalize_features, 
	.bus_name = vp_bus_name, 
	.set_vq_affinity = vp_set_vq_affinity, 
	.get_vq_affinity = vp_get_vq_affinity, 
};
```

virtio_pci_config_ops 主要作用是完成 virtio 设备的 IO 操作，包括读写 virtio pci 设备的 PIO 和 MMIO，如
vp_get_status， vp_set_status。

函数如下：

```c
static u8 vp_get_status(struct virtio_device *vdev) {
	struct virtio_pci_device *vp_dev = to_vp_device(vdev); 
	return ioread8(vp_dev->ioaddr +VIRTIO_PCI_STATUS);
}

static voíd vp_set_status(struct virtio_device *vdev, u8 status) {
	struct virtio_pci_devíce *vp_dev = to_vp_device(vdev); 
	iowrite8(status, vp_dev->ioaddr + VIRTIO_PCI_STATUS); 
}
```

上述函数调用 io read 和 write 函数，直接写地址 vp_dev-＞ioaddr＋VIRTIO_PCI_STATUS。这里读取的是 io addr mmio 地址空间。读写这些数据将会陷入到 QEMU。该 memory region 在 QEMU 有对应的回调函数。

例如，读写的是 common 地址数据

```c
kernel include\uapi\linux\virtio_pci.h 
struct virtio_pci_common_cfg { 
	/* About the whole device. */ 
	_1e32 	device_feature_select; 	/* read-write */ 
	_1e32 	device_feature;  		/* read-only */ 
	_le32 	guest_feature_select;  	/* read-write */ 
	_1e32 	guest_feature;   		/* read-write */ 
	_le16 	msix_config;  			/* read-write */ 
	_le16 	num_queues;    			/* read-only */ 
	_u8 	device_status;   		/* read-write */ 
	_u8 	config_generation;  	/* read-only */ 
	
	/* About a specific virtqueue. */ 
	_le16 	queue_select;   		/* read-write */ 
	_le16 	queue_size;  			/* read-write, power of 2. */ 
	_le16 	queue_msix_vector; 		/* read-write */ 
	_le16 	queue_enable;   		/* read-write */ 
	_le16 	queue_notify_off;  		/* read-only */ 
	_le32 	queue_desc_lo;   		/*read-write */ 
	_le32 	queue_desc_hi;   		/* read-write */ 
	_le32 	queue_avail_lo;  		/* read-write*/ 
	_le32 	queue_avail_hi;  		/* read-write */ 
	_le32 	queue_used_lo;   		/* read-write */ 
	_le32 	queue_used_hi;   		/* read-write */ 
}; 

```

读取上述的 device_feature 字段，陷入到 QEMU

```c
qemu hw\virtio\virtio-pcí.c 
static const MemoryRegionops common_ops = { 
	.read = virtio_pci_common_read, 
	.write = virtio_pci_common_write, 
	.impl = { 
		.min_access_size = 1, 
		.max_access_size = 4, 
	}, 
	.endianness = DEVICE_LITTLE_ENDIAN, 
}; 
```

内部的 virtio_pci_common_read 和 virtio_pci_common_write 封装了对这些 IO 操作。 

## virtio驱动初始

```c
drivers\net\virtio_net.c 
virtnet_probe 
	virtio_cread_feature 
	alloc_etherdev_mq 
	dev->features =NETIF_F_HIGHDMA; 
	init_vqs(vi) 
	virtnet_init_settings(dev) 
	register_netdev(dev) 
	virtio_device_ready(vdev) 
```

virtnet_probe 函数创建了 net_device 结构体，该数据结构存放了与 virtio net 相关的数据成员。然后分配 virtio net feature 标记位。初始化 virt queue，最后设置 virtio 设备标记 DRIVER_OK 特征位。



























