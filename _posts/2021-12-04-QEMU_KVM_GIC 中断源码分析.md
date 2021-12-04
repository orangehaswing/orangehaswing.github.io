---
layout: post
title: QEMU_KVM_GIC 中断控制器源码分析
date: 2021-12-04
tags: jekyll
---

## 简介

GIC，Generic Interrupt Controller。是ARM公司提供的一个通用的中断控制器。主要作用是接受硬件中断信号，并经过一定处理后，分发给对应的CPU进行处理。

当前GIC由四个版本：GICv1 ~ GICv4：

![gic-version](https://gitee.com/orangehaswing/blog_images/raw/master/images/gic-version.png)

## GICv2原理

### 基本模块

gicv2由两大模块组成：

1. distributor：实现中断分发，对于0~31号终端，即SGI，PPI作为每个core的独有中断，不参与core仲裁。只有SPI终端方式是所有core共享。根据配置，由distributor实现中断分发，最终选择最高优先级中断发给CPU。
2. CPU interface / virtual cpu interface：分别用在中断和虚拟中断场景。用于将GICD发送到中断信息或虚拟中断信息，传输给CPU core。其中virtual cpu interface又分为两个组件
   1. virtual interface control：寄存器使用GICH_ 作为前缀
   2. virtual cpu interface：寄存器使用GICV_ 作为前缀

![gicv2-structure](https://gitee.com/orangehaswing/blog_images/raw/master/images/gicv2-structure.png)

上述显示，SGI中断号0 - 15，PPI为16 - 31，SPI为 32 - 1019。GIC包括distributor 和 CPU interface。为了区分中断和虚拟中断投递方式，又细分两个模块。

其中gicv2定义了一些自己的寄存器，使用memory-mapped方式访问，cpu可以通过访问这部分空间对gic操作。

寄存器分为以下几个部分：

1. GICD_*: distributor的寄存器
2. GICH_*: 虚拟interface的控制寄存器
3. GICV_*: 虚拟interface的控制寄存器
4. GICC_*: 虚拟cpu interface的寄存器

中断处理流程主要分为以下几步：

1. GCI决定每个中断的使能状态，不使能的中断，是不能发送到
2. 如果某个中断的中断源有效，GIC将该中断的状态设置位pending，然后判断该中断的目标core
3. 对于每个core，GIC将当前处于pending状态的优先级最高度中断，发送给该core的cpu interface
4. cpu interface 接受 GIC 发送到中断请求，判断优先级是否满足要求。如果满足，就将中断通过nFIQ或nIRQ管脚，发送给core
5. core响应中断，通过读取GICC_IAR寄存器，来认可该中断，如果是软中断，返回处理器ID，否则返回中断号
6. 当core认可该中断后，GIC将该中断的状态，修改为activate状态
7. 当core完成该中断后，通过写EOIR(end of interrupt register)来实现优先级重置，写GICC_DIR寄存器，来无效该中断

### GICv2m

GICv2也支持msi/msix中断方式，但是需要额外支持GICv2m组建，GICv2m组件包括多个msi frame。每个msi frame都单独链接到一组SPI

![gicv2](https://gitee.com/orangehaswing/blog_images/raw/master/images/gicv2.jpg)

每个 msi frame 容量是 4KB，其中包含一些寄存器信息。当向 MSI_SETSPI_NS 寄存器写入数据时，就会触发 SPI 通知
GIC 的过程，数据传输的内容是 SPIID 号，msix 数据格式如下：

```
MsiVector{
	msg_addr_1o
	msg_addr_hi
	msg_data
	masked: false,
}
```

其中 msg_data 包含了 SPIID 号。则 msg_data 格式如下

```
kernel\Documentation\virtual\kvm\api.tt
bits: 	| 	31...28   |   27...24  |   23...16   |  15...0
field:  | vcpu2_index |  irq_type  |  vcpu_index |  irq_id |
```

irq_type 为KVM_ARM_IRQ_TYPE_SPI，irq_id 为 SPI id 号。如果使用 0 号 cpu，msg_data＝1＜＜24｜spi id。
每个单独的 msi frame 都对应单独的 spi id 窗口，架构如下：

![gicv2-frame](https://gitee.com/orangehaswing/blog_images/raw/master/images/gicv2-frame.jpg)

### GICv3原理

GICV3 相比 v2 版本，支持 8 个 PE，msi/msix 中断。超过 1020 个中断 ID 等特性。

定义了如下几种中断类型：

1. SPI（Shared Peripheral Interrupt）: 公用外部设备断，多个CPU或core处理，不限特定CPU。例如按键，触摸屏。
2. PPI（Private Peripheral Interrupt）: 私有外设中断，是每个核心独占，PPI会送到指定CPU，例如CPU本地时钟
3. SGI（Software Generated Interrupt）: 软中断，通过写GICD_SGIR寄存器触发中断事件，一般用于核间通
4. LPI（Locality-specific Peripheral Interrupt）: 是GICv3新特性，LPI始终基于消息的中断，配置保存在表中而不是寄存器，比如msi/msix。

中断类型与中断号对应关系

| 硬件中断号  |          中断类型          |
| :---------: | :------------------------: |
|    0-15     |            SGI             |
|   16 - 31   |            PPI             |
|  32 - 1019  |            SPI             |
| 1020 - 1023 | 用于指示特殊情况的特殊中断 |
| 1024 - 8191 |          Reservd           |
| 8192 - MAX  |            LPI             |

GICv3由三个组件组成：

- distributor: SPI 中断的管理，将中断发送给 redistributor
- redistributor: PPI, SGI, LPI 中断的管理，将中断发送给 cpu interface
- cpu interface: 传输中断给 core

![gicv3](https://gitee.com/orangehaswing/blog_images/raw/master/images/gic-v3.png)

distributor 主要作用是检测各个中断源状态，控制中断源行为，并分发产生的中断事件到指定的一个或多个 CPU interface。

redistributor 主要作用是对 SGI 和 PPI 设置，启用或禁用中断，设置优先级，设置电平触发或边缘触发等

cpu interface：用于和 CPU 接口，使能或禁用CP的中断事件，包括 nIRQ 和 nFIQ 两种中断信号线。通知中断等操作。

### LPI与ITS

额外组件：ITS

LPI 的中断方式基于内存（msi/msix 中断方式），GIC提供一个寄存器（GITS_TRANSLATER寄存器），当外设往这个寄存器写数据，就向 GIC 发送中断。基于内存的方式，减少中断线数量。当 ITS 接收到 LPI 形式的中断，ITS 就进行解析，发送中断给 redistributor。通常使用 ITS 完成 msi/msix 方式的中断路由。

当外设通过写 GITS_TRANSLATER，发起 LPI 中断时，提供两个信息：eventlD 和 devicelD。eventID 表示外设发送的中断类型, devicelD 表示哪一个外设发起 LPI 中断

整体流程是: ITS 接收到 LPI 中断号，发送信息到 redistributor，最后将中断发送给 CPU。

说明：cpu interface 在 CPU 的 core 内部实现功能，distributor, redistributor, ITS 在 GIC 内部实现。

中断处理流程：

1. 外设设备发起中断，分发给 distributor
2. distributor 将中断，分发给合适的 redistributor
3. redistributor 将中断信息，发送给 CPU interface
4. CPU interface 产生合适的中断异常给 CPU
5. CPU 处理该异常，并调用软件处理中断

![flow](https://gitee.com/orangehaswing/blog_images/raw/master/images/interrupt-flow.jpeg)

### 中断路由

gicv3 使用 hierarchy 来标识一个具体的 core

![gic-route](https://gitee.com/orangehaswing/blog_images/raw/master/images/gic-route.png)

上述描述了一个PE的中断路由。每一个core的affnity值可以通过MPDIR_EL1寄存器获取，每个affnity占用8bit。配置对应core的MPIDR值，可以将中断路由到该core上。

## GIC v2/v3 源码分析

### 创建中断芯片

arm gic 中断控制器的调用在 hw\armlvirt.c machvirt_init 函数，主要工作创建中断芯片，并在 fdt 创建相应的 node

```
hw\arm\virt.c
machvirt_init
	// 如果版本是v3，需要获取最大CPU数量，v2版本最大8个
	if(vms->gic_version == 3){
		virt_max_cpus = vms->memmap[VIRT_GIC_REDIST].size / GICV3_REDIST_SIZE;
		virt_max_cpus += vms->memmap[VIRT_HIGH_GIC_REDIST2].size / GICV3_REDIST_SIZE;
	else
		virt_max_cpus = GIC_NCPU;
		
	// 创建中断芯片过程
	create_gic(vms)
	
	create_pcie
		// 如果是SPI中断方式，创建引脚与中断号之间的中断映射
		qemu_fdt_setprop_cell(vms->fdt, nodename, "//interrupt-cells", 1);
		create_pcie_irq_map(vms, vms->gic_phandle, irq, nodename)
```

gic中断控制器组件的内存布局如下：

```
qemu hw/arm/virt.c
static const MemMapEntry base_memmap[] = {
    ...
    /* GIC distributor and CPU interfaces sit inside the CPU peripheral space */
    [VIRT_GIC_DIST] =           { 0x08000000, 0x00010000 },
    [VIRT_GIC_CPU] =            { 0x08010000, 0x00010000 },
    [VIRT_GIC_V2M] =            { 0x08020000, 0x00001000 },
    [VIRT_GIC_HYP] =            { 0x08030000, 0x00010000 },
    [VIRT_GIC_VCPU] =           { 0x08040000, 0x00010000 },
    /* The space in between here is reserved for GICv3 CPU/vCPU/HYP */
    [VIRT_GIC_ITS] =            { 0x08080000, 0x00020000 },
    /* This redistributor space allows up to 2*64kB*123 CPUs */
    [VIRT_GIC_REDIST] =         { 0x080A0000, 0x00F60000 },
    ...
 }
```

上述的 create_gic 函数用于创建中断芯片，根据 gic version 版本不同，创建 v2 或 v3 的中断芯片。create_pcie_irq_map 函数用于创建中断引脚与中断号之间的映射关系。

```
create_gic
	// 根据gic版本，获取gic类型
	gictype = (type==3) ? gicv3_class_name() : gic_class_name()
	// 创建gic中断控制器结构体
	vms->gic = qdev_create(NULL, gictype)
	// 设置cpu和中断数，注意，中断数在base 32基数上增加，其中前32保留给SGI，PPI
	qdev_prop_set_uint32(vms->gic, "num-cpu", num_cpus) 
	qdev_prop_set_uint32(vms->gic, "num-irq", NUM_IRQS+32)
	
	// 如果类型是v3，设置redist数量，如果region大于2个，分为redist和high redist两个区域
	if(type == 3)
		uint32_t rediste_capacity = vms->memmap[VIRT_GIC_REDIST].size / GICV3_REDIST_SIZE; 
		nb_redist_regions = virt_gicv3_redist_region_count(vms);
		
		if(nb_redist_regions == 2)
			uint32_t redist1_capacity = vms->memmap[VIRT_HIGH_GIC_REDIST2].size / GICV3_REDIST_SIZE;

	// 初始化gic中断控制器
	qdev_init_nofail(vms->gic)
	
	// 设置dist基地址
	sysbus_mio_map(gicbusdev, e, vms->memmap[VIRT_GIC_DIST].base)
	
	if(type == 3)
		// 如果是v3版本，设置redist基地址，包括普通和high地址
		sysbus_mmio_map(gicbusdev, 1, vms->memmap[VIRT_GIC_REDIST].base)
		sysbus_mmio_map(gicbusdev, 2, vms->memmap[VIRT_HIGH_GIC_REDIST2].base)
	else
		// 设置cpu基地址
		sysbus_mmio_map(gicbusdev, 1, vms->memmap[VIRT_GIC_CPU].base
		
	// 添加gic中断芯片node到fdt表
	fdt_add_gic_node(vms)
	
	//以下用于支持msi/msix，如果版本是v3，创建its，如果是v2，创建v2m
	if (type == 3 && vms->its) {
		create_its(vms);
	} else if (type == 2) {
		create_v2m(vms);
	}
```

上述qdev_create（NULL，gictype）用于创建一个中断控制器类型，其调用函数如下

如果是gicv2

```
hw\intc\arm_gic.c
arm_gic_class_init
	// init函数内设置rezlize函数
	device_class_set_parent_realize(dc, arm_gic_realize, &agc->parent_realize)

// realize函数主要用于初始化irq和相应mmio，并注册了回调函数gic_set_irq, gic_ops, gic_virt_opsa
arm_gic_realize
	gic_init_irqs_and_mmio(s, gic_set_irq, gic_ops, gic_virt_ops)
	for(i = 0; 1 < s->num_cpu; i++)
		// 为所有cpu汪册io region，用于cpu interface
		memory_region_init_io(&s->cpuiomem[i+1], OBJECT(s), &gic_cpu_ops, &s->backref[i], "gic_cpu", 0x100)
```

如果是gicv3

```
hw\intc\anm_gicv3_kvm.c
kvm_arm_gicv3_class_init
	// 设置realize函数
	device_class_set_parent_realize(dc, kvm_arm_gicv3_realize, &kgc->parent_realize);
	
kvm_arm_gicv3_realize
	// 初始化irqs 和 mmio region
	gicv3_init_irqs_and_mmio
	// 为CPU realize cpu interface
	for(i=0; i < s->num_cpu; i++) {
    	if(qemu_get_cpu(i)){
			kvm_arm_gicv3_cpu_realize(s, i);
	}
	
	// 创建gic v3设备
	kvm_create_device(kvm_state, KVM_DEV_TYPE_ARM_VGIC_V3, false)
	
	// 向kvm设置属性
	kvm_device_access(s->dev_fd, KVM_DEV_ARM_VGIC_GRP_CTRL, KVM_DEV_ARM_VGIC_CTRL_INIT, NULL, true, &error_abort)
	kvm_arm_register_device(&s->iomem_dist, -1, KVM_DEV_ARM_VGIC_GRP_ADDR, KVM_VGIC_V3_ADDR_TYPE_DIST, s->dev_fd, e)
	kvm_arm_register_device(&s->iomem_redist[i], -1, KVM_DEV_ARM_VGIC_GRP_ADDR, KVM_VGIC_V3_ADDR_TYPE_REDIST_REGION, s->dev_fd, addr_ormask)
	
	// 提交中断路由表
	kvm_irqchip_commit_routes(kvm_state)
```

### ITS组件

为了支持 mis/msix 中断方式，gic 需要额外增加 its 或 v2m 组件，创建 ITS 只需要实现 ITS 组件，添加 ITS 基地址。

```
hw\arm\virt.c
create_its
	// 创建和初始化its
	qdev_create(NULL, itsclass)
	qdev_init_nofail(dev)
	
	// 添加基地址
	sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, vms->memmap[VIRT_GIC_ITS].base)
	
	// its添加到fdt表
	fdt_add_its_gic_node(vms)
```

创建 its 组件函数如下

```
hw\intc\arm_gicv3_its_kvm.c
kvm_arm_its_realize
	// 使用系统调用，向kvm创建its设备
	kvm_create_device(kvm_state, KVM_DEV_TYPE_ARM_VGIC_ITS, false)
	// 向kvm注册its地址
	kvm_arm_register_device(&s->iomem_its_cntrl, -1, KM_DEV_ARM_VGIC_GRP_ADDR, KVM_VGIC_ITS_ADDR_TYPE, S->dev_fd, 8)
```

its 发送 msi 中断调用函数是 kvm_its_send_msi，该函数作用是获取 msi 信息，调用 KVM_SIGNAL_MSI 通知 kvm，完成中断注入。

```
hw\intc\arm_gicv3_its_kvm.c
kvm_its_send_msi
	msi.address_lo = extract64(s->gits_translater_gpa, 0, 32)
	msi.address_hi = extract64(s->gits_translater_gpa, 32, 32)
	msi.data = le32_to_cpu(value)
	
	kvm_vm_ioctl(kvm_state, KVM_SIGNAL_MSI, &msi)
```

### V2M组件

创建 v2m 需要额外创建一个 region，当 guest 内核使能 v2m 组件时，会对该 region 进行读写，然后配置 guest 内部 v2m 组件。

```
hw\arm\virt.c
create_v2m
	// 创建v2m结构体，并设置v2m基地址，设置irq中断起始号，中断数量
	qdev_create(NULL, "arm-gicv2m")
	sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, vms->memmap[VIRT_GIC_V2M].base)
	qdev_prop_set_uint32(dev, "base-spi", irq);
	qdev_prop_set_uint32(dev, "num-spi", NUM_GICV2M_SPIS)
	
	// 初始化v2m组件
	qdev_init_nofail(dev)
	
	// v2m添加到fdt表
	fdt_add_v2m_gic_node(vms)
```

创建并初始化 v2m 组件的调用如下

```
hw\intc\arm_gicv2m.c
gicv2m_class_init

gicv2m_init
	// 创建一段region，guest os通过读写这段region，对虚拟机内中断进行分配
	memory_region_init_io(&s->iomem, OBJECT(s), &gicv2m_ops, s, "gicv2m", 0x1000)

gicv2m_realize
	// 初始化irq
	for(i=0; i < s->num_spi; i++)
		sysbus_init_irq(SYS_BUS_DEVICE(dev), &s->spi[i])
```

其中 gicv2m_ops 是对 region 读写的回调函数。读的主要作用是根据 msi 类型，返回 base spi 和 num spi 值。写的作用是根据写入的value 值，转换为 spi 值，然后调用 gicv2m_set_irq，向 kvm 设置 spi 脉冲，触发中断。

```
static const MemoryRegionops gicv2m_ops = {
	.read = gicv2m_read,
	.write = gicv2m_write,
}

// 读过程
gicv2m_read
	case V2M_MSI_TYPER:
		val = (s->base_spi + 32) << 16
		val|= s->num_spi
		return val;
		
// 写过程
gicv2m_write
	case V2M_MSI_SETSPI_NS:
		spi = (value & 0x3ff) -(s->base_spi + 32)
		gicv2m_set_irq(s, spi)
			qemu_irq_pulse(s->spi[irq])
			qemu_set_irq(irq, 1)
			qemu_set_irq(irq, 0)
			ioctl(kvm->vm_fd, KVM_IRQ_LINE, &irq_level)
```

### 添加fdt表

在创建 gic 中断控制器时，会将 gic 各种信息，添加到 fdt 表。guest kernel 在启动过程中，会解析 fdt 表，获取设置的 gic 设备。
首先调用 create_pcie 创建 irq 和中断线的关系

```
hw\arm\virt.c
create_pcie_irq_map
	for (devfn = 0; devfn <= 0x18; devfn += 0x8)
		for (pin = 0; pin < 4; pin++)
			int irq_type = GIC_FDT_IRQ_TYPE_SPI;
			int irq_nr = first_irq +((pin + PCI_SLOT(devfn)) % PCI_NUM_PINS);
			int irq_level = GIC_FDT_IRQ_FLAGS_LEVEL_HI;
			uint32_t map[] = {
				devfn << 8, 0, 0,   /* devfn */
				pin+1,           /* PCI pin */
				gic_phandle, 0, 0, irq_type,irq_nr,irq_level /* GIC irq */
			
			for(i=0; i<10; i++)
				irq_map[i] = cpu_to_be32(map[i]);
			irq_map += 10;

	qemu_fdt_setprop(vms->fdt, nodename, "interrupt-map", full_irq_map,sizeof(full_irq_map));
	qemu_fdt_setprop_cel1s(vms->fdt, nodename, "interrupt-map-mask", 
							0x1800, 0, 0,   /*  devfn (PCI_SLOT(3)) */
							0x7       /*PCI irq */);
```

主要作用是使用虚拟的方式模拟下图显示的逻辑

![irq-pin](https://gitee.com/orangehaswing/blog_images/raw/master/images/irq-pin.png)

然后调用 create_gic-＞fdt_add_gic_node 函数，把 gic 添加到 fdt node。

```
hw\arm\virt.c
fdt_add_gíc_node
	nodename = g_strdup_printf("/intc@%" PRIx64, vms->memmap[VIRT_GIC_DIST].base)
	if (vms->gic_version == 3)
		qemu_fdt_setprop_sized_cells(vms->fdt, nodename, "reg",
									2, vms->memmap[VIRT_GIC_DIST].base,
									2, vms->memmap[VIRT_GIC_DIST].size,
									2, vms->memmap[VIRT_GIC_REDIST].base,
									2, vms->memmap[VIRT_GIC_REDIST].size,
									2, vms->memmap[VIRT_HIGH_GIC_REDIST2].base,
									2, vms->memmap[VIRT_HIGH_GIC_REDIST2].size);
	else
		qemu_fdt_setprop_sized_cells(vms->fdt, nodename, "reg",
									2, vms-smemmap[VIRT_GIC_DIST].base,
									2, vms->memmap[VIRT_GIC_DIST].size,
									2, vms->memmap[VIRT_GIC_CPU].base,
									2, vms->memmap[VIRT_GIC_CPU].size);
```

## kvm-tool实现gic设备

kvm tool 源码实现了一个最简单的 gic 设备，这里以 kvm-tool 为例，分析创建一个 gic 设备需要哪些步骤。首先是初始化 gic。起始函数是kvm_arch_init-＞gic_init_gic

```
gic_init_gic
	// 设置irq属性
	ioctl(gic_fd, KVM_HAS_DEVICE_ATTR, &nr_irqs_attr)
	
	// 初始化中断路由表
	irq_routing_init
	ioctl(gíc_fd, KVM_HAS_DEVICE_ATTR, &vgic_init_attr)
	
	// 向kvm提交中断路由表和irqfd的关系
	irq_setup_irqfd_lines
		irq_common_add_irqfd
			ioctl(kvm->vm_fd, KVM_IRQFD, &irqfd)
```

然后创建 gic 中断控制器，调用的是 kvm_arch_init-＞gic_create

```
gic_create(kvm, kvm->cfg.arch.irqchip)
	// 如果类型是gicv2m
	case IRQCHIP_GICV2M
		gic_msi_size = KVM_VGIC_V2M_SIZE
		gic_msi_base = ARM_GIC_CPUI_BASE - gic_msi_size
	// 如果类型是gicv3
	case IRQCHIP_GICV3_ITS
		gic_msi_size = KVM_VGIC_V3_ITS_SIZE
	case IRQCHIP_GICV3
		gic_redists_size = kvm->cfg.nrcpus * ARM_GIC_REDIST_SIZE
		gic_redists_base = ARM_GIC_DIST_BASE - gic_redists_size
```

上述主要保存内存布局的 base 和 size。

然后调用 gic_create_device 创建 gic 设备

```
gic_create_device
	// 创建gic设备
	ioctl(kvm->vm_fd, KVM_CREATE_DEVICE, &gic_device)
	
	// 如果是gicv2m，设置cpu特性
	case IRQCHIP_GICV2M:
	case IRQCHIP_GICV2:
		err = ioctl(gic_fd, KVM_SET_DEVICE_ATTR, &cpu_if_attr);
		
	// 如果是gicv3，设置redist特性
	case IRQCHIP_GICV3_ITS:
	case IRQCHIP_GICV3:
		err = ioctl(gic_fd, KVM_SET_DEVICE_ATTR, &redist_attr);
		
	// 设置通用属性dist
	ioctl(gic_fd, KVM_SET_DEVICE_ATTR, &dist_attr);
```

上述主要调用 ioctl 创建 gic 中断控制器，并对中断控制器的 cpu interface，distribution，redistribution 组件设置属性。

最后创建支持 msi 中断方式的组件

```
gíc_create_msi_frame
	// gicv2设置v2m组件
	case IRQCHIP_GICV2M:
		return gic_create_gicv2m_frame(kvm, msi_frame_addr)
	
	// gicv3设置its组件
	case IRQCHIP_GICV3_ITS:
		return gic_create_its_frame(kvm, msi_frame_addr)
```

对上述设置两个组建分别分析
gicv2m

```
gic_create_gicv2m_frame
	// 设置msi对路由表的回调操作
	msi_routing_ops = &gicv2m_routing;
	// 注册一段mmio region，回调函数是gicv2m_mmio_callback，用于向guest kernel返回base spi和num spi
	kvm_register_mmio(kvm, base, KVM_VGIC_V2M_SIZE, false, gicv2m_mmio_callback, kvm);
```

上述 gicv2m_routing 回调函数分为更新中断路由表，设置 spi 与中断 gsi 号对应关系，向 kvm 注入中断。

gicv3 its

```
gic_create_its_frame
	//创建its组件，额外设置its组件属性
	ioctl(kvm->vm_fd, KVM_CREATE_DEVICE, &its_device)
	ioctl(its_device.fd, KVM_HAS_DEVICE_ATTR, &its_attr)
	ioctl(its_device.fd, KVM_SET_DEVICE_ATTR, &its_attr)
	ioctl(its_device.fd, KVM_SET_DEVICE_ATTR, &its_init_attr)
```

同样，kvm-tool 也向 fdt 注册了 gic node 信息，设置 pci 设备与中断控制器的连接方式

```
gic_generate_fdt_nodes
	...
```

### guest kernel加载gicv2m

加载 gicv2m 的函数为 gicv2m_init

```
drivers\írqchíp\irq-gic-v2m.c
gícv2m_init
	//这里有个两种方式，从fdt加载或从acpi表加载
	if (is_of_node(parent_handle))
		return gicv2m_of_init(parent_handle, parent);
	return gicv2m_acpí_init(parent);
```

下面以从 fdt 表加载分析

```
gicv2m_of_init
	for child = of_fínd_matching_node(node, gicv2m_device_id)...
		//获取base spi和num spi值
		of_property_read_u32(child, "arm,msi-base-spi", &spi_start)
		of_property_read_u32(child, "arm,msi-num-spis", &nr_spis)
		
		//初始化一个gicv2m
		gicv2m_init_one
			//分配v2m结构体
			v2m = kzalloc(sizeof(struct v2m_data), GFP_KERNEL)
			//从region读取base值
			v2m->base = ioremap(v2m->res.start, resource_size(&v2m->res))
			
			//赋值base spi和num spi
			if(spi_start && nr_spis)
				v2m->spi_start = spi_start;
				v2m->nr_spis = nr_spis;
				
			// 创建msi vector bitmap
			v2m->bm = kcalloc(BITS_TO_LONGS(v2m->nr_spis), sizeof(long), GFP_KERNEL)
```

## 参考

1.https://blog.csdn.net/yhb1047818384/article/details/86708769

2.https://www.cnblogs.com/schips/p/arm-gic-serial.html

3.https://www.linaro.org/blog/kvm-pciemsi-passthrough-armarm64/



























































































