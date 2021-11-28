---
layout: post
title: QEMU_KVM_APIC 中断控制器源码分析
date: 2021-11-21
tags: jekyll
---

## 简介

在 x86 中，当设备向 CPU 发出中断，中断不会直接发给 CPU，而是通过中断控制器，处理对各设备的中断请求。在 SMP 架构下，使用高级可编程中断控制器（APIC）。由 Local APIC 和 I/O APIC 两部分构成，一般来说，所有 LAPIC 都连接到一个 I/O APIC 上，形成一个一对多的结构。用 APIC 替换 PIC，发挥 SMP 系统的并发优势，实现 IPI(核间中断)

APIC有两种工作模式：

1. 8259A 模式: 禁用 LAPIC, APIC 直连 CPU
2. 标准模式: 启用 LAPIC，所有的外部中断通过 IOAPIC 接收后转发给对应的 LAPIC

整体架构：

![apic](https://gitee.com/orangehaswing/blog_images/raw/master/images/apic_structure.png)

APIC 包含2个部分，LAPIC 和 IOAPIC 

1. LAPIC 即 local APIC，位于处理器中，除接收来自 IOAPIC 的中断外，还可以发送和接收来自其它核的 IPI。每个 LAPIC 都有自己的一系列寄存器、一个内部时钟(TSC)、一个本地定时设备和两条 IRQ 线 LINT0 和 LINT1

   常用的寄存器包括：

- ICR（Interrupt Command Register）用于发送IPI 
- IRR（Interrupt Request Register）当前LAPIC接收到的中断请求 
- ISR（In-Service Register）当前LAPIC送入CPU中（CPU正在处理）的中断请求
- TPR（Task Priority Register）当前CPU处理中断所需的优先级
- PPR（Processor Priority Register）当前CPU处理中断所需的优先级，只读，由TPR决定

IRR 与 ISR 两个寄存器，在处理一个 vector 的同时，缓存一个相同的 vector，vector 通过2个 256-bit 的寄存器标识，256个 bit 代表 256 个可能的 vector，置1表示上报了相应的 vector 请求处理或者正在处理中。

LAPIC结构如下图：

![lapic](https://gitee.com/orangehaswing/blog_images/raw/master/images/lapic.png)

2. IOAPIC 一般位于南桥上，响应来自外部设备的中断，并将中断发送给 LAPIC，然后由 LAPIC 发送给对应的 CPU，IOPIC 和 LAPIC 直接使用了系统总线。

当 IO APIC 收到设备的中断请求时，通过寄存器决定将中断发送给哪个 LAPIC（CPU）。位于 0x10 - 0x3F 地址偏移的地方，存放着 24个 64bit 的寄存器，每个对应 IOAPIC 的 24 个管脚之一，这 24 个寄存器统称为 Redirecti on Table ，每个寄存器都是一个 entry。该重定向表会在系统初始化时由内核设置，在系统启动后也可动态修改该表。

- IOREGSEL (I/O REGISTER SELECT REGISTER)：选择要读写的寄存器
- IOWIN (I/O WINDOW REGISTER)：读写 IOREGSEL 选中的寄存器 
- IOAPICVER (I/O APIC VERSION REGISTER)：IOAPIC 的硬件版本 
- IOAPICARB (I/O APIC ARBITRATION REGISTER)：IOAPIC 在总线上的仲裁优先级 
- IOAPICID (I/OAPIC IDENTIFICATION REGISTER)：IOAPIC 的 ID，在仲裁时将作为 ID 加载到 IOAPICARB 中 
- IOREDTBL (I/O REDIRECTION TABLE REGISTERS)：有0 - 23共24个，对应24个引脚，每个长 64bit。当该引脚收到中断信号时，将根据该寄存器产生中断消息送给相应的 LAPIC

## 数据结构

kvm 为 LAPIC 定义了结构体 kvm_lapic

```
struct kvm_lapic{ 
	unsigned 	long base_address; 		 	// LAPIC的基地址
	struct 		kvm_io_device dev; 			// 准备将LAPIC注册为I0设备
	struct 		kvm_timer lapic_timer;		// 定时器
	u32 		divide_count; 
	struct kvm_vcpu *vcpu;				   // 所属vcpu
	bool 		sw_enabled; 
	bool 		irr_pending; 
	bool 		lvt0_in_nmi_mode;
	s16 		isr_count; 
	int 		highest_isr_cache; 
	void 		*regs; 
	gpa_t 		vapic_addr; 
	struct 		gfn_to_hva_cache vapic_cache; 
	unsigned long pending_events; 
	unsigned int sipi_vector; 
}; 
```

LAPIC 由3部分组成

1. 即将作为 IO 设备注册到 Guest 的设备相关信息
2. 一个定时器
3. lapic 的 apic-page 信息 

kvm 为 IOAPIC 定义了结构体 kvm_ioapic

```
struct kvm_foapic{ 
	u64 	base_address;	// IOAPIC的基地址 
	u32 	ioregsel;		// 用于选择APIC-page中的寄存器
	u32 	id;				// IOAPIC的ID 
	u32 	irr;			// IRR 
	u32 	pad; 
	union 	kvm_ioapic_redirect_entry redirtb1［I0APIC_NUM_PINS］; // 重定向表 
	unsigned long 	irq_states[10APIC_NUM_PINS]; 
	struct 	kvm_io_device dev； // 准备注册为Guest的10设备
	struct kvm 	*kvm; 
	void 	(*ack_notifier)(void *opaque, int irq); 
	spinlock_t 	lock;
	struct 	rtc_status rtc_status;
	struct 	delayed_work eoi_inject;
	u32 	irq_eoi[IOAPIC_NUM_PINS];
	u32 	irr_delivered;
};
```

IOPIC 由3部分组成

1. 即将作为 IO 设备注册到 Guest 的设备信息
2. IOAPIC 的基础，如用于标记 IOAPIC 身份的 APIC ID，用于选择 APIC-page 中寄存器的寄存器 IOREGSEL。
3. 重定向表

每个 kvm_ioapic_redirect_entry 中有对应的中断向量号，触发模式，发送 local APIC 的 id 等。

## 创建过程

### 虚拟IOAPIC 

PIC 的创建过程时，qemu 通过 IOCTL(KVM_CREATE_IRQCHIP) 在内核中申请创建一个虚拟 PIC。

IOAPIC 也一样，在不知道 Guest 支持什么样的中断芯片时，会对 PIC 和 IOAPIC 都进行创建，Guest 会自动选择合适自己的中断芯片。

```
qemu: 
accel\kvm\kvm-all.c 
kvm_irqchip_create 
	// 探测是否支持创建一个irqchip
	ret = kvm_arch_irqchip_create(machine, s); 
	// 调用ioctl，创建kvm chip
	ret = kvm_vm_ioctl(s, KVM_CREATE_IRQCHIP); 

kvm: 
arch\x86\kvm\x86.c 
kvm_arch_vm_ioctl 
	case KVM_CREATE_IRQCHIP 
		// 这里与pic不同，pic调用kvm_pic_init 
		kvm_ioapic_init（kvm）
			ioapic-kzəlloc(sizeof(struct kvm_ioapic), GFP_KERNEL) 
			kvim->arch.vioapíc = ioapic; 
			// 初始化iopic ops
			kvm_iodevice_init(&ioapic->dev, &ioapic_mmio_ops); 
			ioapíc->kvm = kvm; 
			// 注册iopic设备
			kvm_lo_bus_register_dev 

kvm_iodevice_init 
```


上述注册 ops，loaplc_mmio_read 和 loapic_mmio_write 用于 Guest 对 IOAPIC 进行配置，具体到达方式为 MMIO Fxit。

### 虚拟LAPIC 

LAPIC 每个 vcpu 都应该有一个，所以 KVM 将创建虚拟 LAPIC 的工作放在了创建 vcpu 时，即当 QEMU 发出 IOCTL(KVM_CREATE_VCPU)，KVM 在创建 vcpu 时，检测中断芯片(PIC/IOAPIC)是否在内核中，如果在内核中，就调用  kvm_create_lapic() 创建虚拟 LAPIC。

```
arch\x86\kvm\svm.c 
kvm_vcpu_init 
	kvm_create_lapic
	
kvm_create_lapic
	// 分配结构体
	apic = kzalloc(sizeof(*apic), GFP_KERNEL)
	// 为apic page分配空间
	apic->regs = (void *)get_zerod_page()GFP_KERNEL)
	// 绑定对应CPU
	apic->vcpu = vcpu
	// 初始化计时器
	hrtimer_init(&apic->lapic_timer.timer, CLOCK_MONOTONIC, HRTIMER_MODE_ABS_PINNED);
	// 设置基地址
	vcpu->arch.apic_base = MSR_TA32_APICBASE_ENABLE;
	// 初始化ops
	kvm_iodevice_init(&apic-<dev, &apic_mmio_ops);
```

## 中断流程

与 PIC 稍有不同的是，APIC 既可以接收外部中断，也可以处理核间中断，而外部中断和核间中断的虚拟化流程是不同的，因此这里分为两个部分。

### 外部中断

假设 Guest 需要从一个外设读取数据，一般流程为 vmexit 到 kvm/qemu，传递相关信息给 qemu，然后返回到 Guest。当 qemu 获取数据完成之后，qemu/kvm 向 Guest 发送外部中断，通知 Guest 数据准备就绪。

这里 qemu 的操作和 pic 发送中断请求一样，都调用 kvm_set_irq 向 kvm 发送中断请求。


通过 APIC 发送中断过程：

```
qemu: 
accel\kvm\kvm-all.c 
int kvm_init 
	s->irq_set_ioctl = KVM_IRQ_LINE; 

// 调用kvm注入中断
int kvm_set_irq 
	// 通过ioctl，向kvm发送中断注入
	ret = kvm_vm_ioctl(s, s->irq_set_ioctl, &event); 

kvm: 
virt\kvm\kvm_main.c 
kvm_vm_ioctl 
	case KVM_IRQ_LINE: 
		r = kvm_vm_ioctl_irq_line(kvm, &irq_event, ioct1 == KVM_IRQ_LINE_STATUS); 
		irqchip_in_kernel(kvm)	// 判断pic是否在kvm实现
		kvm_set_irq 
			// 设置irq号到gsi号的转换
			kvm_irq_map_gsi(kvm, irq_set, irq); 
			// set回调
			irq_set[i].set(＆irq_set[i], kvm, irq_source_id, level, line_status); 

// 与pic不同，这里的回调函数是kvm_set_ioapic_irq，回调由向kvm注册的kvm_irq_routing_entry irqchip决定arch\x86\kvm\irq_comm.c 
kvm_set_routing_entry 
	e->set = kvm_set_ioapic_irq 
	
kvm_set_ioapic_irq 
	kvm_ioapic_set_irq 
		// arch\x86\kvm\ioapic,c 
		ioapic_set_irq 
		entry ＝ ioapic-＞redirtbl[irq]; 		// irq对应的entry 
		// 这里的level已经表示输入有效性而非电平高低 1-有效，0-无效
		if (lirq_level)
			ioapic-＞irr ＆= ～mask；		  // 如果输入无效，则clear掉irq对应的IRR
			ret= 1; 
			goto out; 

		ioapic-＞irr ｜= mask； // set irq对应的IRR
		if (edge)  			// trigger mode为边沿触发 且 IRR出现了沿
			ioapic->irr_delivered &= ~mask; 
			if (old_irr ＝= ioapic-＞irr) // 远方的LAPIC正在处理中断
				ret = 0; 
				goto out; 
			// 真正进行中断服务
			ioapic_service 

ioapic_service 
	if (entry-＞fields.mask || // 如果irq没有被ioapic屏蔽
		// 电平触发时，还需要将remote_irr置1，在收到LAPIC的EOI信息后将remote_irr置0
	  （entry-＞fields.trig_mode ＝= 10APIC_LEVEL_TRIG && entry->fields.remote_irr)) 
 	irqe.dest_id = entry->fields.dest_id; 
	irqe.vector = entry->fields.vector; 
	irqe.dest_mode = entry->fields.dest_mode; 
	irqe.trig_mode = entry->fields.trig_mode; 
	irqe.delivery_mode = entry->fields.delivery_mode << 8; 
	irqe.level = 1; 
	irqe.shorthand = 0; 
	irqe.msi_redir_hint = false; 
	
	if(irqe.trig_mode ==IOAPIC_EDGE_TRIG) 
		ioapic-＞irr_delivered |= 1 << irq；// 沿触发时，ioapic_service为IRR产生一个下降沿
	// 向apic发送irq
	kvm_irq_delivery_to_apic 
		// 查找对应vcpu
		kvm_for_each_vcpu(i, vcpu, kvm) 
		// 查找到有多个vcpu
		if (dest_vcpus != 0) 
			int idx = kvm_vector_to_index(irq->vector, dest_vcpus, dest_vcpu_bitmap, KVM_MAX_VCPUS); 
			lowest = kvm_get_vcpu(kvm,idx); 
		// 发送设置irq
		if (lowest) 
			r = kvm_apic_set_irq(lowest, irq, dest_map); 
kvm_apic_set_irq 
	_apic_accept_irq 
		// 根据apic不同类型，踢醒vcpu
		kvm_vcpu_kick(vcpu); 
```


小结：当 qemu 通过 kvm_set_irq，调用 ioctl 发送中断请求，kvm 就会经过一系列调用，设置 apic 的 IRR 和 TMR，然后踢醒 vcpu， vmentry 时，会触发扫描中断检查，查询到对应中断。

### 核间中断

物理机上，CPU-0 的 LAPIC-x 向 CPU-1 的 LAPIC-y 发送核间中断时，会将中断向量和目标的 LAPIC 标识符存储在自己的 LAPIC-x 的 ICR (中断命令寄存器)中，然后该中断会顺着总线到达目标 CPU。

虚拟机上，当 vcpu-0 的 lapic-x 向 vcpu-1 的 lapic-y 发送核间中断时，会首先访问 apic-page 的 ICR 寄存器（因为要将中断向量信息和目标 lapic 信息都放在 ICR 中），在没有硬件支持的中断虚拟化时，访问(write) apic-page 会导致 mmio vmexit，在 KVM 中将所有相关信息放在 ICR 中，在之后的 vcpu-1 的 vm entry 时会检直中断，进而注入 IPI 中断。

在没有硬件辅助中断虚拟化的情况下，对 apic-page 的读写会 vm exit 最终调 apic_mmio_write()/apic_mmio_read()。即上述的 ops 注册回调函数。

回调函数：

```
arch\x86\kvm\lapic.c 
apic_mmio_write 
	kvm_lapic_reg_write(apic,offset & Oxffø,val) 
		switch(reg) 
		case APIC_ICR: 
			kvm_lapíc_set_reg(apic, APIC_ICR, val & ~(1 << 12)); 
				apic_send_ipi（apic）；// 发送ipi 

apic_send_ipí
	u32 icr_low ＝ kvm_lapíc_get_reg(apic, APIC_ICR);	// ICR低32bit 
	u32 icr_high ＝ kvm_lapic_get_reg(apic, APIC_ICR2); // ICR高32bit 
	
	irq.vector = icr_low&APIC_VECTOR_MASK;		//vector 
	irq.delivery_mode = icr_low&APIC_MOOE_MASK;
	írq.dest_mode = icr_low&APIC_DEST_MASK;		// Destination Mode 
	irq.level = (ícr_low & APIC_INT_ASSERT) !-0; 
	irq.trig_mode = icr_low& APIC_INT_LEVELTRIG; 
	irq.shorthand = icr_low& APIC_SHORT_MASK; 
	irq.msi_redír_hint = false;
	irq.dest_id = GET_APIC_DEST_FIELD(icr_high)	// Destination Field 

kvm_irq_delivery_to_apic 	// 发送到lapic 
	kvm_for_each_vcpu		// 遍历所有vcpu 
		// 如果vcpu有lapic且vcpu是该中断的目标
		kvm_apic_match_dest(vcpu, src, irq->shorthand, irq->dest_id, irq->dest_mode) 
```

可以看到，核间中断由 Guest 通过写 apic-page 中的 ICR(中断控制寄存器)发起，vm exit 到 KVM，最终调用 apíc_mmio_write()。

```
apic_mmio_write() 
	=>apic_send_ipi() 
		=>_apic_accept_irq() 
		｜- 设置apic-page的IRR 
		｜- 设置apic-page的TMR 
		｜- 踢醒（出）vCPU
vm entry时触发中断检测
```

## 参考

1.https://www.cnblogs.com/wsg1100/p/14055863.html
