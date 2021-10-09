---
layout: post
title: QEMU_virtio 源码分析
date: 2021-10-09
tags: jekyll    
---

## 概述

在传统的设备全模拟中，虚拟机完全不知道自己处于虚拟化环境。virtio 是一种前后端架构，包括 kernel 中的前端驱动和后端设备，以及自身的传输协议。

1. 前端驱动为虚拟机内部的 virtio 模拟设备对应的驱动，每种设备都有对应的驱动。主要作用是接收用户态请求，然后按照请求，进行封装。最后通过写 IO 端口，通知 qemu 后端设备
2. 后端设备在 qemu 中，用来接收 IO 请求，然后从请求数据中按传输协议进行解析。根据物理设备的不同，操作实际的物理设备，例如网络数据的传输，将转发给 tap 设备。完成请求后，会通过中断机制通知前端驱动

 virtio 前后端的数据传输通过 virtio 队列（virtio queue，virtqueue）完成 。一个设备会注册多个 virtio 队列，每个队列负责不同的数据传输。有的是控制面的队列，有的是数据传输队列。virtqueue 是通过 vring 实现，是虚拟机和 qemu 之间的一段共享内存，用环形缓冲区表示。

虚拟机将要发送请求准备好，放在共享内存，然后把数据描述放在vring,写对应的 IO 端口。qemu 通过 epoll，接收到虚拟机的通知，从 vring 中读取数据描述信息。然后去对应的共享内存区域读取数据。qemu 在完成请求后，将数据结构放在vring，通过中断的方式通知虚拟机。虚拟机就可以直接从 vring 获取数据。

vring 包含三个部分：descriptor table (用来表示IO请求的传输数据信息，包括长度，内存地址），available vring (前端驱动设置的表示后端设备可用的描述符表的索引），usedvring (后端设备在使用完描述符表后设置的索引）。

## virtio 设备初始化


virtio 设备在初始化时，会创建一个 PCI 设备，叫 virtio PCI 代理设备。代理设备会挂载 PCI 总线上，然后 virtio 代理设备会创建一条 virtio 总线，最终 virtio 设备挂载到 virtio 总线上。

```
virtio_pci_register_types 
	type_register_static(&virtio_pci_bus_info); 
	type_register_static(&virtio_pci_info); 

static const TypeInfo virtio_pci_bus_info ={ 
	.name     =	TYPE_VIRTIO_PCI_BUS, 
	.parent   =	TYPE_VIRTIO_BUS, 
	.instance_size	=	sizeof(VirtioPCIBusState), 
	.class_size  =	sizeof(VirtioPCIBusClass), 
	.class_init  =	virtio_pci_bus_class_init, 
}; 

static const TypeInfo virtio_pci_info = { 
	.name     =	TYPE_VIRTIO_PCI, 
	.parent   =	TYPE_PCI_DEVICE, 
	.instance_size = sizeof(VirtIOPCIProxy), 
	.class_init  =	virtio_pci_class_init, 
	.class_size  =	sizeof(VirtioPCIClass), 
	.abstract   = true, 
}; 
```

上述 virtio pci 注册了两种类型：VirtioPCIBusClass 和 VirtioPCIClass。virtio PCI 代理的父设备是一个 PCI 设备，类型是 VirtioPCIClass，实例是 VirtlOPCIProxy。创建方式如下：

```
VirtIOPCIProxy *proxy	=	VIRTIO_PCI(obj); 
```

proxy 设备是一个抽象设备，不能创建实例，只能由子类去创建。qemu 中定义 virtio 设备的 pci 设备，例如 virtioballon PCI，virtio scsi PCI 等 

```
static const VirtioPCIDeviceTypeInfo virtio_balloon_pci_info=( 
	.base_name      	= 	TYPE_VIRTI0_BALLOON_PCI, 
	.generic_name       = 	"virtio-balloon-pci", 
	.transirional_name  = 	"vIrtio-balloon-pci-transitional", 
	.non_transitional_name = "virtio-balloon-pci-non-transitional", 
	.instance_size 		= 	sizeof(VirtI0BalloonPCI), 
	.instance_init 		= 	virtio_balloon_pci_instance_init, 
	·Class_init   		= 	virtio_balloon_pci_class_init, 
}; 
```

virtio 设备在系统设备树的结构如下图

```
----------- main system bus ---------------
				|
			 pci host
				|
-------------- pci.0 -------------
				|
		virtio balloon pci
				|
------------- virtio bus ---------
				|
			virtio balloon device
```

所有 virtio 设备的父类是 TYPE_VIRTIO_DEVICE，实例对象是 VirtlOBalloon 

```
static const TypeInfo virtio_balloon_info = { 
	.name	=	TYPE_VIRTIO_BALLOON, 
	.parent =	TYPE_VIRTIO_DEVICE, 
	.instance_síze = sizeof(VirtIOBalloon), 
	.instance_init = virtio_balloon_instance_init, 
	.class_init = virtio_balloon_class_init, 
}; 
```

整体的对应关系:

```
VirtIOBalloonPCI 
	# proxy
	VirtIOPCIProxy 
		PCIDevice 
		MemoryRegion 
		VirtIOPCIQueue 
		VirtioBusState 
		
	# device
	VirtIOBalloon 
		VirtIODevice 
		VirtQueue 
		VirtQueueElement 
```

VirtlOBalloonPCI 是 virtio ballon PCI 代理设备的实例对象，包括 VirtlOPCIProx 代理设备和 VirtlOBalloon 设备。 代理设备放与 PCI 相关的成员和数据，ballon 设备放具体的设备数据。

创建 virtio 设备过程：qemu 命令行输入

```
-device virtio-ballon-pci 
```

具体的实例化过程：

```
virtio_balloon_pci_info { 
	.instance_init 	=	virtio_balloon_pci_instance_init, 
}

virtio_balloon_pci_instance_init 
	VirtIOBalloonPCI *dev =VIRTIO_BALLOON_PCI(obj); 
	virtio_instance_init_common(obj, &dev->vdev, sizeof(dev->vdev), TYPE_VIRTIO_BALLOON); 
		object_initialize_child_with_props 
			object_initíalize_child_with_propsv 
				object_initíalize(childobj, size, type) 
				object_property_add_child(parentobj, propname, obj) 
		qdev_alias_all_properties(vdev, proxy_obj) 
```

在这个实例化过程中，会调用 virtio_instance_init_common，并将 vdev 成员地址，类型为 TYPE_VIRTIO_BALLOON 传给初始化函数 object_initialize。该过程只是初始化了 QOM。

## virtio设备具现化

qemu 在 main 函数对所有 -device 参数进行解析，然后调用具现化统一函数 device_set_realized。该函数内部会调用
设备类的 realize 函数。

virtio 设备的继承关系：DeviceClass-＞PCIDeviceClass-＞VirtioPCIClass。类型初始化函数如下： 

```
virtio_pci_class_init 	DeviceClass *dc = DEVICE_CLASS(klass); 	PCIDeviceClass *k = PCI_DEVICE_CLASS(klass); 	VirtioPCIClass *vpciklass = VIRTIO_PCI_CLASS(klass); 		k->realize = virtio_pci_realize; 	device_class_set_parent_realize(dc, virtio_pci_dc_realize, &vpciklass->parent_dc_realize); 		*parent_realize = dc->realize; 		dc->realize = virtio_pci_dc_realize; 
```

这一步，将 PCIDeviceClass 的 realize 回调函数设置为 virtio_pci_realize。这里 dc-＞realize 被调换了顺序。最终，VirtioPCIClass parent_dc_realize 变成了 dc-＞realize（pci_qdev_realize）， dc-＞realize 的回调函数变成了 
virtio_pci_dc_realize。 

其中 dc-＞realize 的初始化位置如下：

```
hw\pci\pci. c pci_device_class_init 	DeviceClass *k = DEVICE_CLASS(klass); 	k->realize = pci_qdev_realize; 
```

在 virtio ballon PCI 代理设备初始化函数 virtio_balloon_pci_class_init 中，VirtioPCIClass realize 设置成了 virtio_balloon_pci_realize 

```
vírtio_balloon_pci_class_init 	DeviceClass *dc = DEVICE_CLASS(klass); 	VirtioPCIClass *k = VIRTIO_PCI_CLASS(klass); 	PCIDeviceClass *pcidev_k = PCI_DEVICE_CLASS(klass); 	k->realize = vírtio_balloon_pci_realize; 
```

综上，virtio 设备的 realize 函数关系如下：

```
device_set_realized 	dc-＞realize ＃DeviceClass-＞realize，就是virtio_pci_dc_realize 		virtio_pci_dc_realize 			vpciklass-＞parent_dc_realize ＃ VirtioPCIClass-＞parent_dc_realize，就是 pci_qdev_realize 				pcí_qdev_realize 					pc-＞realize ＃PCIDeviceClass-＞realize，就是virtio_pci_realize virtio_pcí_realize 						k-＞realize ＃VirtioPCIClass-＞realize，就是virtio_balloon_pci_realize 							virtio_balloon_pci_realize 								qdev_realize 
```

## PCI realize 过程 

pci 设备的 realize 过程，调用的函数是 pci_qdev_realize。主要工作分为三个方面：

1. 调用 do_pci_register_device 注册，完成设备及对应 PCI 总线上的一些初始化。
2. 调用所属的 PCI realize 函数。
3. 最后调用 pci_add_option_rom 加载 PCI 设备的 ROM 空间

```
hw\pci\pci.c pci_qdev_realize 	do_pci_register_device 	if(pc->realize) { 		pc->realize(pci_dev, &local_err)	}	pci_add_option_rom(pci_dev, is_default_rom, &local_err) 
```

### PCI 注册

```
do_pci_register_device 	＃获取PCI设备和bus总线	DeviceState *dev = DEVICE(pci_dev) 	PCIBus *bus = pci_get_bus(pci_dev) 		＃ 如果设置devfn ＝-1，由总线自己选择插槽，得到插槽保存devfn。	if (devfn < 0) {		...	}	pci_dev->devfn = devfn; 		＃ 获取dev id，这部分内容将在寻找msi／msix中断控制器用到。	＃ 找寻规律：从设备开始向上查找，一直找到root bus。根据设备挂载在root bus，pcie桥，pci桥不同，设置不同的cache type和dev 	pci_dev->requester_id_cache = pci_req_id_cache_get(pci_dev) 		＃ 初始化一段region，类型是bus_master_container_region 	memory_region_init 	＃ 如果是bus master类型，即PCI主设备，设置相应的region	pci_init_bus_master(pci_dev); 		＃ 设置PCI配置空间和irq中断相关信息	pci_dev->irq_state = 0;    pci_config_alloc(pci_dev);     pci_config_set_vendor_id(pci_dev->config, pc->vendor_id); 		    pci_config_set_device_id(pci_dev->config, pc->device_id);     pci_config_set_revision(pci_dev->config, pc->revision);    pci_config_set_class(pcí_dev->config, pc->class_id);    ...    	＃ 设置pci设备是多功能设备，如果是，在header type第7bit置1，表示开启多功能	pcí_init_multifunction(bus, pci_dev, &local_err) 		＃ 设置读写pci 配置空间回调函数	config_read = pci_default_read_config; 	config_write = pci_default_write_config; 	pci_dev->config_read = config_read; 	pci_dev->config_write = config_write; 
```

### virtio PCI realize 过程 

初始化 virtio PCI 代理设备过程

```
virtio_pci_realize 	*	region 0   --	virtio legacy io bar 	*	region 1   --  	msi-x bar 	*	region 2   --  	virtio modern io bar (off by default) 	*	region 4+5 -- 	virtio modern memory (64bit) bar 	memory_region_init(&proxy->modern_bar,. . .) 	＃创建virtio bus总线	virtio_pci_bus_new(&proxy->bus, szeof(proxy->bus), proxy) 	k->realize(proxy,errp) 
```

在 virtio pci realize 过程中，初始化了多个 BAR 空间信息 proxy 的 common， isr ， device ， notify ， notify_pio 设置 bar 索引号。从上述注释看出： bar 0 是 legacy io bar， bar 1 是 msix 区域，bar 2 是 modern IO，bar 4 是 MMIO 区域。

### 设置 rom 区域

```
pci_add_option_rom ＃ 如果设备没有rom文件或文件为空，直接返回if (lpdev->romfile) return; if (strlen(pdev->romfile)==0) return; ＃ 这里判断是否使用fw_cfg中的文件，还是自己创建一个。if(lpdev->rom_bar){ int class - pci_get_word(pdev->config + PCI_CLASS_DEVICE) ＃ 有的设备有自己的rom，就从设备指定开始加载) if (class == exe3ee) { rom_add_vga(pdev->romfile); } else { rom_add_option(pdev->romfile,-1); ) return; ＃查找目录对应的rom文件和大小path = qemu_find_file(QEMU_FILE_TYPE_BIOS, pdev->romfile); size=get_image_size(path); ＃创建一段rom region，并和rom文件建立映射关系，向pci注册rom barmemory_region_init_rom memory_region_get_ram_ptr pcí_regíster_bar(pdev, PCI_ROM_SLOT,O,&pdev->rom) 
```

## virtio 设备 realize 

### 使能设备

```
qdev_realize 	object_property_set_bool(OBJECT(dev), "realized", true, errp) 
```

这里调用 object_property_set_bool，将 virtio 设备具现化，调用 virtio_device_realize。 

```
virtio_device_realize 	vdc->realize(dev,&err) 	virtio_bus_device_plugged 
```

vdc-＞realize 会调用具体的函数具现化，例如 net 设备的具现化函数：virtio_net_device_realize

```
hw\net\virtio-net.c virtio_net_device_realize 	＃设置配置空间大小	virtio_net_set_config_size 		＃virtio初始化	virtio_init 		＃分配virtio队列内存	n->vqs = g_malloc0(sizeof(VirtIONetQueue) * n->max_queues) 	＃添加virtio队列	for (i = Ø; i < n->max_queues; i++) { 		virtio_net_add_queue(n, i); 	}		n->nic = qemu_new_nic(&net_virtio_info, &n->nic_conf,n->netclient_type,n->netclient_name, 	...
```

当设备设备挂载到 virtio bus 总线，将调用 virtio_pci_device_plugged，为设备注册 bar 空间数据。

```
virtio_pci_device_plugged 	＃设置pci配置空间	pci_set_word(config+PCI_VENDOR_ID,PCI_VENDOR_ID_REDHAT_QUMRANET); 	pci_set_word(config +PCI_DEVICE_ID,øx1040 + virtio_bus_get_vdev_id(bus)); 		pci_config_set_revision(config, 1); 	config[PCI_INTERRUPT_PIN] = 1; 		＃创建cap信息，包括notify， cfg，notify_pio	...	＃设置映射关系，proxy-＞xxx是在virtio_pci_realize阶段赋值内容，到这里，capacity内容建立完成	virtio_pci_modern_mem_region_map(proxy,&proxy->common,&cap); 	virtio_pci_modern_mem_region_map(proxy,&proxy->isr,&cap); 	virtio_pci_modern_mem_region_map(proxy, &proxy->device, &cap); 	virtio_pci_modern_mem_region_map(proxy, &proxy->notify, &notify.cap); 		＃向pci注册bar空间	pci_register_bar 	＃设置pci读写回调函数	proxy->pci_dev.config_write = virtio_write_config; 	proxy->pci_dev.config_read = virtio_read_config; 
```

主要工作是设置设备的配置空间 vendor id 和 device id。在 device id 的 base 基础上（0x1040），加上设备的 id。接下来讲 virtio 设备的寄存器配置信息作为 pci capability 写入到配置空间。

pci capability 用来表示设备的功能。virtio device 把多个 Memory Region 作为 MMIO 对应的子 region。将这些信息写入 virtio pci 设备的配置空间。结构体如下：

```
include\standard-headers\linux\virtio_pci.h struct virtio_pci_cap{ 	uint8_t cap_vndr;   /* Generic PCI field: PCI_CAP_ID_VNDR */ 	uint8_t cap_next;   /*Generic PCI field: next ptr. */ 	uint8_t cap_len;   /*Generic PCI field:capability length */ 	uint8_t cfg_type;   /* Identifies the structure.*/ 	uint8_t bar;   /*Where to find it.*/ 	uint8_t id;  /* Multiple capabilities of the same type */ 	uint8_t padding[2];/* Pad to full dword.*/ 	uint32_t offset;   /* Offset within bar.*/ 	uint32_t length;   /*Length of the structure,in bytes.*/ }; 
```

其中，cap_vndr 表示 cap id。cap_next 指向下一个 cap pci 配置空间位置。bar 表示使用哪个 bar 空间。offset 表示MemoryRegion 在设备 bar 的偏移位置。length 表示结构体的长度。

## Memory region 

virtio region init 过程中，会初始化5个 memory region。分别是

1. virtio-pci-common 
2. virtio-pci-isr 
3. virtio-pci-device 
4. virtio-pci-notify 
5. virtio-pci-notify-pio 

这些 memory region 的信息放在 VritioPCIRegion 类型的成员中。

```
virtio_pci_device_plugged 	virtio_pci_modern_mem_region_map(proxy, &proxy->common, &cap); 	virtio_pci_modern_mem_region_map(proxy, &proxy->isr, &cap); 	vírtio_pci_modern_mem_region_map(proxy, &proxy->device, &cap); 	virtio_pcí_modern_mem_region_map(proxy, &proxy->notify, &notify.cap); 	if(modern_pio) {		virtio_pci_modern_io_region_map(proxy, &proxy->notify_pio, &notify_pio.cap); 	}
```

virtio_pci_modern_mem_region_map 主要完成两个功能：

1. 将 VirtioPCIRegion 的 mr 成员 virtio-pci-xx 作为子 region，加入到 bar region。当虚拟机内部读写这段 memory region 时，会陷入到这几段 region 的回调函数
2. 调用 virtio_pci_add_mem_cap 函数，将这些寄存器加入到 virtio pci 设备的 pci capability 上

```
virtio_pci_modern_mem_region_map 	virtio_pci_modern_region_map 		memory_region_add_subregion 		virtio_pci_add_mem_cap
```

## 注册 PCI bar

```
pci_register_bar ＃判断如果是桥设备，只能分配两个bar空间hdr_type = pci_dev->config[PCI_HEADER_TYPE] & ~PCI_HEADER_TYPE_MULTI_FUNCTION; assert(hdr_type!=PCI_HEADER_TYPE_BRIDGE || region_num < 2); ＃保存region信息，在最初注册的时候，region 地址是PCI_BAR_UNMAPPED，也就是gpa还没有分配。在kernel＃启动过程中，才会分配具体的gpa addrr->addr = PCI_BAR_UNMAPPED; r->size = size; r->type = type; r->memory =memory; r->address_space 
```

## 小结

virtio 设备在初始化和具现化过程中涉及的函数调用
初始化过程

1. virtio_net_pci_instance_init 
2. virtio_instance_init_common 
3. virtio_net_instance_init 

具现化过程

TYPE_VIRTIO_PCI realize 阶段 

1. virtio_pci_dc_realize 
2. pci_qdev_realize 
3. virtio_pci_realize 

TYPE_VIRTIO_NET_PCI 

1. virtio_pci_bus_new 
2. virtio_net_pci_realize 

TYPE_VIRTIO_DEVICE 

virtio_device_realize 

TYPE_VIRTIO_NET 

1. virtio_net_device_realize 
2. virtio_init 
3. virtio_add_queue 









