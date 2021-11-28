---
layout: post
title: QEMU_VFIO_MSIX 中断源码分析
date: 2021-07-24
tags: jekyll
---

## 概述

Guest OS 在启动阶段(包括 BIOS 和 kernel 两个阶段)，会对 vfio 设备的配置空间读写。当写设备配置空间时，在 qemu 内部的回调函数是 vfio_pci_write_config。该函数内部会对写操作进行判断。如果是配置 msi、msix 时，会使能/禁止设备终端。这个操作具体发生在，向 msi、msi-x 的 capability structure 中写入 msi/msi-x enable bit。

如果Guest OS发送到数据能正确配置msi、msi-x功能，就会调用 vfio_msi_enable 或 vfio_msix_enable

## 使能中断

本文只针对 MSI-X 中断方式代码进行分析。

在 vfio_realize 过程中，调用 vfio_add_capabllities 函数，内部将初始化 msix。在 qemu 中，对 MSI-X table area 的模拟由 qemu 来模拟。所以会创建一个 MSI-X 结构。

```java
vfio_add_capabilities
	vfio_add_std_cap
		vfio_msix_setup
			msix_init
```

当 guest OS 向 qemu msix 寄存器 MSIX enbale(message control bit 15) 写1时，就会使能 MSIX。

![msixtable](https://gitee.com/orangehaswing/blog_images/raw/master/images/MSIX_cap.png)

使能 msix 过程，主要是申请gsi，创建中断需要的eventfd，并将这些资源向kvm注册。由kvm管理中断。

```java
vfio_pci_write_config()
	// 如果写的offset地址落在msix cap ～cap＋12区间内，就进入使能阶段
	else if (pdev->cap_present&QEMU_PCI_CAP_MSIX && 
			 ranges_overlap(addr, len, pdev->msix_cap, MSIX_CAP_LENGTH)) { 
		// 首先判断msix msg ctl标志位，记录上次msix是否已经使能
		was_enabled = msix_enabled 
		// 将这次写的数据，向msix msg ctl写入
		pci_default_write_config 
		// 再次读取msix msg ctl标志位
		is_enabled = msix_enabled 
		//  分别要处理两种情况：
		// 1．上次状态没使能，这次状态使能，那就enable
		// 2．上次状态使能，这次关闭使能，那就disable
		if(!was_enabled && is_enabled) { 
			vfio_msix_enable 
		} else if (was_enabled && !is_enabled) { 
			vfio_msix_disable 
		}
	}
	
//  以enable msix， 分析函数调用过程
vfio_msix_enable()
	// 关闭中断，需要关闭的是msi和INTx，msix在这时肯定没有使能
	vfio_disable_interrupts
	// 有多少个msix table entry，就会分配多少vector，将作为标记，记录irq eventfd，gsi是否被使用。
	vdev->msi_vectors = g_new0(VFIOMSIVector, vdev->msix->entries) 

	//  分配gsi， írq eventfd， 向kvm注册中断路由。
	vfio_msix_vector_do_use

	vfio_msix_vector_release
		// 第一次将不会进入，因为这时virq＝-1
		if (vector->virg >=e){
			...
		}
		
	// 注册回调函数，当向msix bar table写，虚拟机就会陷出（没有建EPT页表），回调函数就会重新调用vfio_msix_vector_use和	vfio_msix_vector_release 
	msix_set_vector_notifiers(vfio_msix_vector_use,vfio_msix_vector_release) 
		dev->msix_vector_use_notifier =use_notifier 
		dev->msix_vector_release_notifier = release_notifier; 
	
// 分别分析三个函数
vfio_msix_vector_do_use(&vdev->pdev,e,NULL,NULL)
	入参vector是0，这时的vector一定没有被使用，会进入下面条件
	if(!vector->use){ 
		// 负数，表示没有分配gsí，这里的virq就是gsi，只是变量名不同
		vector->vírq = -1; 
		// 初始化irq eventfd
		if (event_notifier_init(&vector->interrupt, e)) { 
			...
		} 
		vector->use-true; 
		msix_vector_use(pdev, nr); 
	}
	
	// 第一次调用，不满足所有条件，将不进入下面的任何判断条件。
	if (vector->virq >= 0) { 
		if(!msg){ 
			vfio_remove_kvm_msí_vírq(vector); 
		} else { 
			vfio_update_kvm_msi_virq(vector, *esg, pdev); 
		}
	} else { 
		if (msg) { 
			vfio_add_kvm_msi_virq(vdev, vector, nr,true); 
		}
	}
	
	if (vdev->nr_vectors < nr +1) { 
		vdev->nr_vectors = nr+1 
		// 构造irq_set，设备irq数量1，类型msix，保存irq eventfd的raw fd，向kvm注册绑定
		vfio_enable_vectors 
			irq_set->index=VFIO_PCI_MSIX_IRQ_INDEX 
			irq_set->count=vdev->nr_vectors 
			fds =(int32_t *)&irq_set->data 

			// 向kvm注册，表示中断对应的irqfd
			ioctl(vdev->vbasedev. fd, VFIO_DEVICE_SET_IRQS,irq_set) 
	}

	// 禁用PBA模拟
	clear_bit(nr,vdev->msix->penmsix_msgding)
```

a. vfio_add_kvm_msi_virq 向 KVM 中注入监听事件，当虚机写入 MSI 配置空间时，可以获取到 MSI message（包括 address 和 data，data 中包含了设备在虚机内部使用的中断号），设备在向 KVM 注入事件时需要传入虚机内部使用的中断号，这个中断号在 Posted Interrupt 时会使用。

b. vfio_enable_vectors 利用 vfio 提供的 VFIO_DEVICE_SET_IRQS 命令为透传设备申请真正的中断回调函数，因为中断相关的配置都不允许虚机直接访问，必须借助 VFIO 提供的 IO 接口实现。此时，会在物理机内申请中断，中断服务程序在 VFIO 内实现，服务程序主要激活步骤 a 中的监听事件，然后再调用 Posted Interrupt。

小结：当硬件设备产生中断时，首先物理机会执行服务程序，服务程序主要工作是发送 eventfd_signal，激活 kvm 中的 irqfd_inject，最终调用 deliver_posted_interrupt 向虚机注入中断。

## 注册中断回调

接着上述的 vfio_enable_vectors 函数深入分析，入参是 irqfd，系统调用会在内核注册中断回调函数。调用的关键函数是 request_irq()，参考[链接](https://www.kernel.org/doc/html/v4.12/core-api/genericirq.html)。

该函数向内核注册了一个中断回调函数：vfio_msihandler。这个回调函数的作用是，当有中断来时，会向 eventfd 写1。这个 eventfd 会提前注册到 kvm。所以 kvm 一直在 epoll 对应的 eventfd。当 eventfd 被写1后，kvm 接收到中断通知，调用 kvm posted interrupt，向对应的 CPU 发送中断。

```java
vfio_enable_vectors
	ioctl(vdev- >vbasedev. fd,VFIO_DEVICE_SET_IRQS,irq_set)

kernel: drivers\vfio\pci\vfio_pcí. c
// 进入ioctl系统调用
if (cmd ==VFIO_DEVICE_SET_IRQS) 
	vfio_pci_set_irqs_ioct1 
		case VFIO_PCI_MSIX_IRQ_INDEX 
			vfio_pci_set_msi_trigger 
				vfio_msi_set_block 
				// 对输入的每一个irqfd，都会注册中断回调函数
				for (i = 0, j = start; i < count && !ret; i++, j++) { 
					int fd = fds ? fds[i] : -1;
					ret = vfio_msi_set_vector_signal(vdev, j, fd, msix); 
				}
				
// 注册中断回调函数过程
vfio_msi_set_vector_signal
	// 根据irqfd，在内核创建对应的eventfd
	trigger = eventfd_ctx_fdget(fd)
	// 在内核申请注册中断回调函数
	request_irq(irq, vfio_msihandler, Ø, vdev->ctx[vector].name, trigger)

// 实现回调函数
vfio_msihandler
	// 当中断来时，该回调函数会被调用，向eventfd写1
	eventfd_signal(trigger,1)
```

## 中断处理

按照上述所说的，中断来的时候，会触发中断回调函数 vfio_msihandler，相应的 eventfd 会被写。kvm 通过 epoll 方式一直监听 eventfd，当它发现对应的 eventfd 被写了以后，会调用 irqfd_inject 完整中断注入虚拟机的过程。

```c
virt\kvm\eventfd.c
irqfd_inject 
	// 根据触发方式不同，调用一次或者两次（水平触发／上升沿下降沿触发）
	// 注意，这里用到了irqfd对应的gsi，后续会详细分析irqfd绑定gsi的过程
		kvm_set_irq(kvm,KVM_IRQFD_RESAMPLE_IRO_SOURCE_ID,irqfd->gsi,1,false) 
			// 获取中断数量
			i=kvm_irq_map_gsi(kvm,irq_set,irq) 
			while (i--) {
				// kvm中断回调函数，如果是msi／msix，就会调用kvm_set_msi
				irq_set[i].set 
			}

// kvm中断回调kvm_set_msi
kvm_set_msi
	kvm_irq_delivery_to_apic
		kvm_apic_set_irq
		_apic_accept_irq
		// posted interrupt回调函数，向虚拟机注入中断
		kvm_x86_ops->deliver_posted_interrupt
		// 如果不支持 posted interrupt 模式，就会创建中断请求，然后将vcpu kick出来
		kvm_make_request
			set_bit(req & KVM_REQUEST_MASK, (void *)&vcpu->requests)
		kvm_vcpu_kick
```

## irqfd 和 gsi 绑定过程

当向 irqfd 写入数据时，qemu 会调用 irqfd_inject 进行中断注入，将会根据水平触发还是上下沿触发，调用 kvm_set_irq。调用函数如下

```c
irqfd_inject
	kvm_set_irq(kvm,KVM_USERSPACE_IRQ_SOURCE_ID, irqfd->gsi, 1, false);
```

其中的一个参数是 irqfd-＞gsi，这个参数是全局中断号，将通过 gsi 找到对应的 kvm irq route 路由表。

关键数据结构如下

```c
struct kvm_kernel_irqfd {
	...
	/* Update side is protected by irqfds.lock */
	struct kvm_kernel_irq_routing_entry irq_entry;
	/* Used for level IRQ fast-path */
	int gsi;
	struct eventfd_ctx *eventfd;
	...
};
```

在调用函数 kvm_set_irq 时，找到对应的 irqset 内容，也就是找到对应的msi回调函数 kvm_set_msi。

```c
i =kvm_irq_map_gsi(kvm, irq_set, irq);
irq_set[i].set -> 回调 kvm_set_msi
```

将 irqfd 和 gsi 绑定过程在 vfio 代码的 vfio_pci_write_config -> vfio_msix_enable 执行

```c
vfio_add_kvm_msi_virq
	kvm_irqchip_add_irqfd_notifier_gsi
		kvm_irqchip_assign_irqfd
			int fd = event_notifier_get_fd(event)
			struct kvm_irqfd irqfd={
				.fd=fd,
				.gsi =vírq,
				.flags=0,
			};
			kvm_vm_ioctl(s, KVM_IRQFD, &irqfd)
```

系统调用 KVM_IRQFD 在 kvm 代码如下

```c
kvm_vm_ioctl
	case KVM_IRQFD
		virt\kvm\eventfd.c
		kvm_irqfd
		// 判断入参flag，是绑定irqfd还是解除绑定
		if (args->flags & KVM_IRQFD_FLAG_DEASSIGN) {
			kvm_irqfd_deassign
		}
		kvm_irqfd_assign

kvm_irqfd_assign
	// 申请一个结构kvm_kernel_irqfd类型的irqfd
	struct kvm_kernel_irqfd *irqfd
	// 对irqfd各个参数赋值
	irqfd->kvm = kvm;
	irqfd->gsi = args->gsi;
```

## 注入中断路由表

上述中断处理过程中 irq_set[i].set 的回调函数是 kvm_set_msi，是如何通过 gsi 找到对应的中断回调函数的呢？接着分析注入中断路由表的过程。

```c
qemu
vfio_add_kvm_msi_virq
	kvm_irqchip_add_msi_route
		// 定义kvm irq route路由表结构体
		struct kvm_irq_routing_entry kroute
		// 获取中断向量对应的msg
		msg = pci_get_msi_message(dev, vector)
		// 在bit map获取一个空闲的gsi全局中断号
		virq = kvm_irqchip_get_virq(s)
		
		// 填充路由表内容
		// 复制gsi全局中断号信息
		kroute.gsi =virq;
		kroute.type=KVM_IRQ_ROUTING_MSI;
		kroute.flags =0;
		kroute.u.msi.address_lo =(uint32_t)msg.address;
		kroute.u.msi.address_hi=msg.address>> 32;
		kroute.u.msi.data=le32_to_cpu(msg.data);
		kvm_irqchip_commit_routes 

		// 系统调用，注入中断路由表
		kvm_vm_ioct1(s,KVM_SET_GSI_ROUTING,s->irq_routes) 
```

系统调用将进入内核态，下面是 kvm 内容

```c
kvm
virt\kvm\kvm_main.c
kvm_vm_ioctl
	case KVM_SET_GSI_ROUTING
		kvm_set_irq_routing
		// 定义kvm kernel路由表结构体
		struct kvm_kernel_irq_routing_entry
		for (i=0;i < nr; ++i){
			// 复制gsi
			nr_rt_entries = max(nr_rt_entries,ue[i].gsi)
		}

		// 创建一个中断芯片，KVM_NR_IRQCHIPS＝1， KVM_IRQCHIP_NUM_PINS＝256
		// 模拟msi的中断方式
		for (i =Ø; i < KVM_NR_IRQCHIPS; i++) {
			for (j = Ø; j < KVM_IRQCHIP_NUM_PINS;j++) {
				new->chip[i][j] = -1;
			}
		}

		for (i = 0; i < nr; ++i) {
			// 把用户态传入的路由表拷贝到内核态结构体kvm_kernel_irq_routing_entry
			setup_routing_entry
				// 设置两个参数
				e->gsi = gsi;
				e->type = ue->type;
				r = kvm_set_routing_entry(kvm, e, ue)
				// 设置si芯片二维数组每项为全局中断号（实际是一维数组）
				if (e->type == KVM_IRQ_ROUTING_IRQCHIP) {
					rt->chip[e->irqchip.irqchip][e->irqchip.pin] = e->gsi
				}

		// 更新kvm中断路由表
		kvm_irq_routing_update
```

## 总结

整个 msix 中断初始化过程总结如下

1. vfio realize 阶段，初始化一个模拟的 msix 结构。
2. 虚拟机内核在 pci 设备枚举之后，会尝试使能 msix。qemu 进入 vfio_pci_write_config 使能 msix。
3. 根据 msix cap 信息，决定 enable 或 disable。
4. 执行 enable，创建一个 irqfd，将 irqfd 向 kvm 注册。在内核态，会为这个 irqfd 生成一个中断回调函数，当硬件
   设备有中断来时，中断回调函数会向 irqfd 写1，接着完成irqfd中断注入流程。
5. enable 过程中，为 write msix region 注册回调函数。
6. 虚拟机内核接着会 mask 所有的中断 entry，然后使能需要 enable 的 entry．这时的操作是向 msix table area 写
   msix msg。
7. 调用回调函数，分配一个 irqfd，分配一个空闲的gsi全局中断向量号，向 kvm 注册中断路由表。将 irqfd 和 gsi 绑
   定，当 irqfd 中断注入时，可以找到对应的中断回调函数。将 irqfd 向 kvm 注册，生成对应的中断回调函数。

## 参考链接

1.https://www.kernel.org/doc/html/v4.12/core-api/genericirq.html

2.https://www.cnblogs.com/haiyonghao/p/14709880.html

3.https://github.com/qemu/qemu

