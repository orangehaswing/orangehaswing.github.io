---
layout: post
title: kernel_VFIO源码分析
date: 2021-07-24
tags: jekyll   
---

# kernel_VFIO源码分析

## 设备与 vfio-pci 驱动绑定

1. 将设备与原来的驱动解除绑定，重新与 vfio-pci 驱动绑定操作：

   ```
   echo 0000:1a:00.3>/sys/bus/pci/devices/0000\:1a\:00.3/driver/unbind 
   echo'1spci-ns 0000:1a:00.3 |awk -F':| ' '(print $5” "$6)'` > /sys/bus/pci/drivers/vfio-pci/new_id 
   ```

   该过程会调用 vfio 内核接口 vfio_pci_probe。这个函数会在 vfio_pci_init 驱动初始化时，注册 vfio_pci_driver。内部有个probe函数，具体实现是 vfio_pci_probe。

   ```
   module_init(vfio_pci_init)
   	＃注册vfio pci驱动
   	pci_register_driver(&vfio_pci_driver)
   	
   ＃注册驱动的函数如下
   vfio_pci_driver {
   	.probe      = vfio_pci_probe,
   	.remove     = vfio_pci_remove,
   }
   
   drivers\vfio\pci\vfio_pci.c
   ＃驱动绑定调用
   vfio_pci_probe
   	＃ 创建iommu group结构体，并从给定的物理设备获取iommu_group。主要是校验设备是否支持IOMMU
   	struct iommu_group *group
   	group = vfio_iommu_group_get(&pdev->dev)
   		iommu_group_get
   
   	＃添加vfio设备的具体实现
   	vfio_add_group_dev
   		＃返回iommu结构体，表示iommu驱动层group
   		iommu_group=iommu_group_get
   		＃ 根据iommu层group生成vfio的group
   		group=vfio_group_get_from_iommu
   		if(group){
   			＃一个vfio group会对应多个vfio device， 如果第一次创建vfio device，就需要创建vfio group
   			group=vfio_create_group(iommu_group)
   		}
   		device = vfio_group_get_device
   		＃ 最后创建vfio device
   		device=vfio_group_create_device
   ```

   主要工作：

   a. 得到 iommu_group 结构体，表示 iommu 驱动层 group；
   b. 根据 iommu 层的 group 生成一个 vfio 层的 group。group 只会在第一个设备进行直通时候创建；
   c. 判断物理设备 dev 是否已经创建，如果没有，创建一个 vfio_device，并且绑定到 vfio_group上；

2. 在上述 vfio_pci_probe 过程中，会创建及初始化 group。

   ```
   vfio_pci_probe
      vfio_add_group_dev
   		# drivers\vfio\vfio.c
   		vfio_create_group
   			＃ 获取iommu_group结构体
   			group->iommu_group =iommu_group;
   			
   			＃创建vfio group设备，设备对应/vfio/group/＄group_id
   			device_create 
   			＃ 将vfio group设备保存在group结构的dev上
   			group->dev=dev
   
   			＃把vfio group设备添加到链表中
   			list_add(&group->vfio_next, &vfio.group_list)
   ```

   主要工作：该函数会创建一个设备，就是 /dev/vfio/groupid。vfio_group 的成员 dev 保存这个设备。用户态通过该设备控制vfio_group。vfio_group 的 container 成员存放了该 vfio_group 链接到的 container。 

3. vfio_pci_probe 期间，也会创建 vfio device

   ````
   vfio_pci_probe
      vfio_add_group_dev
      		# drivers\vfio\vfio.c
   		vfio_group_create_device
   			＃ 保存vfio device层相关信息：dev表示物理设备，group表示vfio group，ops表示vfio_pci_ops
   			device->dev = dev;
   			device->group #group;
   			device->ops = ops;
   
   			＃ device_data是设备的私有数据，
   			device->device_data = device_data;
   			dev_set_drvdata(dev, device);
   			vfio_group_get
   ````

   主要工作：vfio_device 表示 VFIO 层面的设备，其中 dev 成员表示物理设备。当用户态获取到 /vfio/group/$group_id 后，就分配了一个fd。可以使用iocl系统调用 VFIO_GROUP_GET_DEVICE_FD 获取到这里的 vfio device 的 fd，接首就可以使用ioctl系统调用获取vfio device 的私有数据。vfio_pci_ops 定义了对 vfio 设备操作的回调函数。

## vfio-pci接口

```
static const struct vfio_device_ops vfio_pci_ops={
	.name       = "vfio-pci",
	.open       = vfio_pci_open，＃使能设备，根据物理设备的配置信息生成vfio_pci_device配置信息
	.release    = vfio_pci_release,
	.ioctl      = vfio_pci_ioctl，＃ioctl处理函数
	.read       = vfio_pci_read,
	.write      = vfio_pci_write,
	.mmap     	= vfio_pci_mmap,
	.request    = vfio_pci_request,
};
```

主要工作：VFIO 驱动模块调用 vfio_group_get_device_fd 处理ioctl请求，找到对用 vfio_device，获得空闲fd，并设置文件结构的操作接口为 vfio_pci_ops，将文件结构与空闲fd关联起来。

1. 重要的ioctl

   ```
   vfio_pci_ioctl
   	if(cmd==VFIO_DEVICE_GET_INFO)
   		vfio_device_info
   		info.flags
   		info.num_regions
   		info.num_irqs
   	} else if (cmd == VFI0_DEVICE_GET_EIGION_IMO)(
   		case VFIO_PCI_CONF1G_REGION_INDDX
   		...
   		case VFIO_PCI_BARO_REGION_INDEX...VFIO_PCI_BARS_REGION_INDEX
   		...
   		case VFIO_PCI_ROM_REGION_INDEX
   	} else if (cmd == VFIO_DEVICE_GET_IRQ_INFO)
   		vfio_irq_info
   		info.flags
   		info.count
   		info.index
   	} else if (cmd ==VFIO_DEVICE_SET_IRQS){
   		vfio_set_irqs_validate_and_prepare
   		vfio_pci_set_irqs_ioctl
   		...
   	}
   ```

   主要工作：对iocti做处理。这里以 vfio_region_info 结构体为例进行分析：

   ```
   struct vfio_region_info {
   	_u32  argsz; 
   	_u32  flags;
   	_u32   index;    /* Region index */
   	_u32  cap_offset;/* Offset within info struct of first cap */
   	_u64   size;     /* Region size (bytes) */
   	_u64   offset;   /* Region offset from start of device fd */
   };
   ```

   argsz 表示参数大小，是输入参数；flags 表明该内存区域允许的操作，是输出参数；index 表示ioctl 调用时的内存区域索引，输入参数；size 表示 region 大小，输出参数；offset 表示内存区域在 VFIO 设备文件对应的偏移，输出参数；

   VFIO_PCI_CONFIG_REGION_INDEX:

   ```
   switch (info.index) {
   	case VFIO_PCI_CONFIG_REGION_INDEX:
   		info.offset=VFIO_PCI_INDEX_TO_OFFSET(info.index);
   		info.size=pdev->cfg_size;
   		info.flags =VFIO_REGION_INFO_FLAG_READ | VFIO_REGION_INFO_FLAG_WRITE;
   		break;
   }
   ```

   VFIO_PCI_INDEX_TO_OFFSET 将内存区域索引转换成一个偏移。pdev-＞cfg_size 中保存物理设备的配置空间大小，flags 设置为可读可写。用户态获取 PCI 配置空间信息后，可以使用 write/read 系统调用读写该区域。offset 作为读写的位置。VFIO 读写函数分别是上述注册的 vfio_pci_ops 内的 vfio_pci_read 和 vfio_pci_write。

## vfio驱动

VFIO 模块在初始化函数 vfio_init 会注册一个 misc 设备 vfio_dev。

```
static struct miscdevice vfio_dev={	.minor=VFIO_MINOR,	.name=“vfio”,	.fops =&vfio_fops,	.nodename = "vfio/vfio",	.mode=S_IRUGO | S_IWUGO,};
```

该 misc 设备的文件操作接口保存在 vfio_fops，并且会创建一个设备 /dev/vfio/vfio 全局变量 vfio。

```
static struct vfio {	struct class            *class;	struct list_head      	fommu_drivers_list;	struct mutex         	iommu_drivers_lock;	struct list_head      	group_list;	struct idr       		group_idr;	struct mutex         	group_lock;	struct cdev      		group_cdev;	dev_t           		group_devt;	wait_queue_head_t      	release_q;} vfio;
```

1. 创建 container

   ```
   vfio_fops_open	container = kzalloc(sizeof(*container), GFP_KERNEL)
   ```

   主要工作：分配一个 vfio_container 结构，并初始化。将 container 复制到打开fd的私有结构中。这样，每一个用户态在打开 /dev/vfio/vfio 时内核就为起分配一个 vfio_container 结构体作为该进程所有 VFIO 设备的载体。

2. group 附加到 container

   ```
   vfio_group_set_container	...	ret=driver->ops->attach_group(container->iommu_data, group->iommu_group);	...	vfio_container_get(container);
   ```

   通过打开 /dev/vfio/vfio 获得 container 的fd。通过打开 /dev/vfio/groupid 获得 groupid。内核提供 ioctl，将 group 附加到 container上。

## vfio lOMMU驱动

vfio iommu 驱动是 VFIO 接口和底层 iommu 驱动之间的通信桥梁。向上接收来自 VFIO 接口请求，向下利用 iommu 驱动完成 DMA 重定向。以 vfio_iommu_type1 驱动介绍：

vfio_iommu_type1:

```
static const struct vfio_iommu_driver_ops vfio_iommu_driver_ops_type1 = {	.name      		= "vfio-iommu-type1",	.owner          = THIS_MODULE,	.open       	= vfio_iommu_type1_open,	.release     	= vfio_iommu_type1_release,	.ioct1    		= vfio_iommu_type1_ioctl,	.attach_group     	= vfio_iommu_type1_attach_group,	.detach_group    	= vfio_iommu_type1_detach_group,	.pin_pages      	= vfio_iommu_type1_pin_pages,	.unpin_pages     	= vfio_iommu_type1_unpin_pages,	.register_notifier  = vfio_1ommu_typei_register_notifier, 	.unregister_notifier = vflo_iommu_type1_unregister_notifier, }; 
```

1. 初始化函数是 vfio_iommu_type1_init

   ```
   static int _init vfio_iommu_type1_init(void) {	return vfio_register_iommu_driver(&vfio_iommu_driver_ops_type1);}
   ```

   主要工作：向 vfio 注册一个操作接口，接口如上 vfio_iommu_driver_ops_type1。

2. open 函数

   ```
   vfio_iommu_type1_open	iommu->dma_list =RB_ROOT;
   ```

   主要工作：每个 container 打开一个 vfio_iommu_driver，分配一个 vfio_iommu，然后进行一些初始化。dma_list 成员用来表示该container 中 DMA 重定向的映射表，也就是 GPA 到 HPA 的转换。

3. set iommu

   ```
   vfio_ioctl_set_iommu	try_module_get	driver->ops->ioct1	data = driver->ops->open	_vfio_container_attach_groups
   ```

   主要工作：该接口用来设置 container 使用 iommu 类型。
   a. 参数 args 是用户态进程指定的 vfio_iommu 驱动类型；
   b. 操作 open 函数返回一个不透明结构体，将 container 上所有的 group 都附加到 vfio_iommu 驱动上；
   c. 设置 container 的 iommu_driver 成员为 vfio_iommu_driver，iommu_data 成员为具体 vfio iommu 驱动返回的结构体；

4. dma 映射

   ```
   vfio_fops_unl_ioct1	case VFIO_CHECK_EXTENSION:		vfio_loctl_check_extension	case VFIO_SET_IOMMU:		ret = vfio_loctl_set_iommu	default:		if(driver)	/* passthrough all unrecognized ioctls */			ret =  drier->ops->ioctl(data, cmd, arg);
   ```

   主要工作：通过 ioctl 创建一个设备 I/O 地址（iova）到宿主机物理地址的映射，这样设备在进行 DMA 操作时使用的地址都是 IOVA，经过 IOMMU 的 DMA 重映射进行地址转换。将 IOVA 转换成宿主机物理地址，container 上的所有设备都直通给虚拟机。这些设备使用同一份 IOVA 到宿主机物理地址的映射表。

   ```
   vfio_iommu_type1_ioct1	if(cmd VFIO_CHECK_EXTENSION){		vfio_domains_have_iommu_cache	}else if (cmd == VFIO_IOWMU_GET_INFO){		vfio_pgsize_bitmap	}else if (cmd==VFIO_I0MU_MAP_DMA){		vfio_dma_do_map	}else if(cmd==VFI0_104MU_UNMAP_DMA){		...	}
   ```

   用户态进程调用 ioctl(VFIO_IOMMU_MAP_DMA)接口，从用户态传入 vfio_iommu_type1_dma_map 参数

   ```
   struct vfio_iommu_type1_dma_map{	_u32  	argsz;	_u32  	flags;	_u64   	vaddr;         /*Process virtual address */	_u64  	iova;         /*I0 virtual address */	_u64   	size;         /*Size of mapping (bytes) */};
   ```

   指定了用户态进程的虚拟地址和设备的 IOVA 地址的映射关系。其中，vaddr 表示用户态进程的虚拟地址，iova 表示设备使用的IO地址，size 表示大小。接着调用 vfio_dma_do_map 完成地址映射。

   ```
   ＃获取到用户态洋细信息map以及iommu fddeu op eub ot	＃创建vfio dma结构体	struct vfio_dma*dma	＃参数校验	...	# 找到需要mmap的节点，判断是否已经被映射过	vfio_find_dma(iommu,iova,size)	＃/dev/vfio/vfio的可用dma数量减少1	iommu->d-a_avail--;	# iova是device使用的dma地址	dma->iova-iova;	# vaddr是CPU使用的地址	dma->vaddr ·vaddr;	＃内存读写保护标记位	dma->prot·prot;	# 将dma链接到红黑树中，以确保下次查找该dma（iova space mapping）是否被映射过时能快速找到	vfio_link_dma(iommu, dma) 	# 锁住内有页	vfio_pin_map_dma(iommu, dma, size)		vfio_pin_map_dma_chunk		＃将从vaddr开始，长度为size内存区域的内存空间中的连续页锁定，避免其他应用使用这些页，同时获得这些页的物理地址HPA，		vfio_pin_pages_remote		＃ 利用iommu driver提供的iommu_map函数，将qemu地址空间的一部分（HVA）映射到iova空间，等到将该段IOVA空间和device绑定后，以后deviceBDMA主动发起的动作最终会落到qemu地址空间的一段内存上，也就是落在GPA空间上		vfio_iommu_map
   ```

   























































