---
layout: post
title: QEMU_KVM_PIC 中断控制器源码分析
date: 2021-10-17
tags: jekyll
---

## 硬件逻辑

QEMU 和 KVM 均独立实现了 PIC，APIC( IOAPIC + LAPIC )

![8259A](https://gitee.com/orangehaswing/blog_images/raw/master/images/8259A.jpg)

中断传递过程：

1. 中断由 IRO-IR7 进入，进入之后 PIC 会自动设置 IRR 的对应 bit
2. 经过 IMR 和优先级处理之后，选择优先级最高的中断，通过 INT 管脚发送中断给 CPU
3. CPU 收到中断后，如果决定处理该中断，就会通过 INTA 管脚向 PIC 发送中断已接收通知
4. PIC 接到该通知之后，设置 ISR，表明某中断正在由 CPU 处理。

处理完成中断，信号传递过程：

1. 当 CPU 处理完该中断之后，会向 PIC 发送 EOl(end of interrupt)信号

2. PIC 收到 EOI 后，会将 ISR 中对应的 bit 清除掉

PIC 记录 ISR 的作用之一是当后续收到新的中断时，会将新的中断和正在处理的中断的优先级进行比较，进而否打断 CPU 正在处理的中断。如果 PIC 处于 AEOI(auto EOI)模式，CPU 无需向 PIC 发送 EOI 信号。

注意：x86 中，系统中使用的前 32 号 vector 用于 CPU 自身。对 PIC 的 IRn 加上一个 base(32)，才能转化为真正使用的 vector 号。

## 数据结构

PIC 的状态由 kvm_kpic_state 结构描述

```c
arch\x86\kvm\irq.h 
struct kvm_kpic_state { 
	u8 		last_irr; 	/* edge detection */ 
	u8 		irr;  		/* interrupt request register */ 
	u8 		imr;  		/* interrupt mask register */ 
	u8 		isr;  		/* interrupt service register */ 
	u8 		priority_add;  		/* highest irg priority */ 
	u8 		irq_base; 			// 用于将IRQn转化为VECTORx 
	u8 		read_reg_select;	// 选择要读取的寄存器
	u8 		po1l; 
	u8 		special_mask; 
	u8 		init_state; 
    u8 		auto_eoi;			// 标志auto_eoi模式
	u8 		rotate_on_auto_eoi; 
	u8 		special_fully_nested_mode; 
	u8 		init4;   			/* true if 4 byte init */ 
	u8 		elcr;   			/* PIIX edge/trigger selection */ 
	u8 		elcr_mask; 
	u8 		isr_ack; 			/* interrupt ack detection */ 
	struct kvm_pic ＊pics_state;		// 指向高级PIC抽象结构
}; 
```

kvm_kpic_state 结构体模拟 8259 中断控制器的特征，寄存器 IRR，IMR，ISR 等。

PIC 高级结构

```c
struct kvm_pic { 
	spinlock_t 	lock; 
	bool 		wakeup_needed; 
	unsigned 	pending_acks; 
	struct 		kvm *kvm; 
	struct 		kvm_kpic_state pics[2]; 	/* 0 is master pic, 1 is slave pic */ 
	int 		output;   	// intr from master PIC，vcpu将从output判断是否有中断要处理
	struct 		kvm_io_device dev_master;  	// 这里说明PIC是一个KVM内的I0设备，包括master，slave等 
	struct 		kvm_io_device dev_slave; 
	struct 		kvm_1o_device dev_eclr; 
	void(*ack_notifier)(void *opaque, int irq); 
	unsigned long irq_states[PIC_NUM_PINS]; 
}; 
```

## kvm PIC 初始化

通过参数 kernel-irqchip ＝ x，控制中断芯片由谁模拟。x＝on 表示由 kvm 模拟所有中断芯片。本文所有内容都基于 kvm 模拟。

qemu 调用 ioctl 创建 kvm irqchip 时，kvm 会调用 kvm_arch_vm_ioctl 函数，创建 2 个 8259A 级联的 PIC。同时，也会创建 APIC 和设置默认的中断路由。

kvm_pic_init 负责初始化 PIC。

```c
arch\x86\kvm\i8259.c 
kvm_pic_init(kvm) 
	// 设置寄存器值
	s->pics[0].elcr_mask = 0xf8; 
	s->pics[1].elcr_mask = 0xde; 
	s->pics[0].pics_state = s; 
	s->pics[1]·pics_state = s; 
	
	// 初始化设备，注册回调函数。有三个设备，master，slave，eclr．将PIC注册为KVM的IO设备，以及注册该I0设备ops.
	kvm_iodevice_init(&s->dev_master, &picdev_master_ops); 
	kvm_iodevice_init(&s->dev_slave, &picdev_slave_ops); 
	kvm_iodevice_init(&s->dev_eclr, &picdev_eclr_ops); 
	
	// 向总线注册pic设备和读写端口号。
	ret = kvm_io_bus_register_dev(kvm, KVM_PIO_BUS, 0x20, 2, &s->dev_master); 
	ret = kvm_io_bus_register_dev(kvm, KVM_PIO_BUS, 0xa0, 2, &s->dev_slave); 
	ret = kvm_io_bus_register_dev(kvm, KVM_PIO_BUS, 0x4d0, 2, &s->dev_eclr); 

```

## QEMU PIC初始化

qemu 虚拟机的中断状态由 GSIState 保存

```c
// qemu include\hw\i386\x86.h 
typedef struct GSIState {
	qemu_irq 	18259_frq[ISA_NUM_IRQS]; 
	qemu_irq 	ioapic_irq[IOAPIC_NUM_P1NS]; 
	qemu_irq 	ioapic2_irq[10APIC_NUM_PINS]; 
} GSIState;

typedef struct IRQState *qemu_irq; 
struct IRQState { 
	Object parent_obj; 
	qemu_irq_handler handler; 
	void *opaque; 
	int n;
}; 
```

GSIState 包括 PIC 和 IO APIC 两种芯片的中断信息组成，qemu_irq 是一个指向 IRQState 的指针。IRQState 表示一个中断引脚，其中 handler 表示执行的函数。n 是引脚号。

```c
hw\i386\pc_piix.c 
pc_init1 
	// 分配一个GSI state结构体，保存在gsi_state
	gsi_state = pc_gsi_create(&x86ms->gsi, pcmc->pci_enabled) 
		// 分配结构体
		s = g_newo(GSIState, 1) 
		if(kvm_ioapic_in_kernel()) { 
			kvm_pc_setup_irq_routing(pci_enabled); 
		}
		// 分配一组qemu_irq，handler设置为gsi_handler 
		*irqs = qemu_allocate_irqs(gsi_ handler, s, GSI_NUM_PINS) 
		
// 创建默认gsi对应的中断路由表	
kvm_pc_setup_irq_routing 
	for(i=0; i<8; ++i) { 
		if(i==2) { 
			continue;
		} 
		kvm_irqchip_add_irq_route(s, i, KVM_IRQCHIP_PIC_MASTER, i); 
	}

	for (i=8; i<16; ++1) { 
		kvm_irqchip_add_irq_route(s, i, KVM_IRQCHIP_PIC_SLAVE, i-8); 
	}
	
	if (pci_enabled) { 
		for (i = 0; i < 24; ++1) { 
			if (i == 0) { 
				kvm_irqchip_add_irq_route(s, 1, KVM_IRQCHIP_I0APIC, 2); 
			} else if (i == 2) { 
				kvm_irqchip_add_irq_route(s, 1, KVM_IRQCHIP_IOAPIC, i); 
			}
		} 
	}
	kvm_irqchip_commit_routes(s) 

// 分配IRQState结构体内容
qemu_allocate_irqs 
	qemu_extend_irqs 
		qemu_irq *s; 
		s = old ? g_renew(qemu_irq, old, n + n_old) : g_new(qemu_irq, n) 
		for (i = n_old; i <n + n_old; i++){ 
			s[i]=qemu_allocate_irq(handler, opaque, i); 
		}

qemu_allocate_irq 
	struct IRQState  *irq; 
	irq = IRQ(object_new(TYPE_IRQ)); 
	irq->handler = handler;		// 复制gsi_handler
	irq->opaque = opaque;
	irq->n = n;
	
gsi_handler 
	switch (n) 
		case O...ISA_NUM_IRQS-1 
			qemu_set_irq(s->18259_irq[n], level) 
		case ISA_NUM_IRQS...IOAPIC_NUM_PINS-1 
			qemu_set_irq(s->ioapic_irq[n], leve1) 
		case IO_APIC_SECONDARY_IRQBASE 
			qemu_set_irq(s->ioapic2_irq[n-10_APIC_SECONDARY_IRQBASE], leve1) 
```

qemu_allocate_irqs 函数调用 qemu_extend_irqs，分配了 n 个 qemu_irq。然后调用 qermu_allocate_irq，创建 IRQState 结构体，使用 TYPE_IRQ，handler，opaque 和 n 初始化 IRQState 结构体。

```c
hw\i386\pc_piix.c 
pc_init1 
	if (pcmc->pci_enaled) 
		// i440fx主板初始化
		i440fx_init 
		// piix3南桥设备初始化
		piix3 = piix3_create(pci_bus, &isa_bus) 
	// 中断irq复制给isa bus
	isa_bus_irqs(isa_bus, x86ms->gsi) 
	pc_i8259_create(isa_bus, gsi_state->i8259_irq) 
```

在 i440fx_init 函数，x86ms-＞gsi 会被赋值给南桥 piix3 的 bus irqs 成员。PCI 设备的中断从这里开始分发。南桥 piix3 在具现化的时候创建一条 ISA 总线，并保存到全局变量 isa_bus 中，调用 isa_bus _irqs 将 x86ms-＞gsi 设置到 isa_bus 的 irqs 成员。isa 设备的中断从这里开始分发。

如果 PIC 在内核中模拟，会调用 kvm_i8259_init，否则调用 i8259_init 函数

```c
hw\i386\pc.c 
pc_i8259_create 
	if (kvm_pic_in_kernel()) { 
		i8259 = kvm_i8259_init(isa_bus); 
	} else { 
		i8259 = 18259_init(isa_bus, x86_allocate_cpu_irq()); 
	}
	for (size_t i = 0; i < ISA_NUM_IRQS; i++) { 
		i8259_irqs[i] = i8259[i]; 
	}

// kvm模拟 hw\i386\kvm\i8259．c 
kvm_18259_init 
	// 创建master PIC 
	i8259_init_chip(TYPE_KVM_18259, bus, true); 
	// 创建slave PIC
	18259_init_chip(TYPE_KVM_18259, bus, false); 
	// 分配16个qemu_irq结构，将这些qemu_irq复制给gsi_state的i8259_irq成员。这里设置handler为kvm_pic_set_irq 
	return qemu_allocate_irqs(kvm_pic_set_irq, NULL, ISA_NUM_IRQS); 
	
// qemu模拟 hw\intc\18259．c 
18259_in1t 
	irq_set = g_new0(qemu_irq, ISA_NUM_IRQS) 
	isadev = i8259_init_chip(TYPE_I8259, bus, true) 
	qdev_connect_gpio_out(dev, 0, parent_irq) 
```

如果在 kvm 创建 PIC 设备，两个 i8259_init_chip 用来创建类型为 TYPE_KVM_18259 对象并实现具现化。并设置 qemu_irq handler 为 kvm_pic_set_irq。

## 中断过程

isa 设备申请中断通过 isa_init_irq 函数申请 irq 资源

```c
hw\isa\isa-bus.c 
// 根据设备和中断引脚，申请中断向量
isa_init_irq 
	*p = isa_get_irq(dev, isairq) 
		// 这里的irqs［isairq］就是上述pcms-＞gsi数组
		return isabus->irqs[isairq] 
```

假设 Guest 需要从一个外设读取数据，一般流程为 vmexit 到 kvm/qemu，传递相关通知信息给 qemu 后，直接返回 Guest。当数据获取完成之后，qemu/kvm 向 Guest 发送中断，通知 Guest 数据准备就绪。

qemu/kvm 通过 PIC 向 Guest 发送中断过程：

1.pic 产生

当外设设备在 qemu 中模拟

```c
qemu: 
void qemu_set_irq 
	// 入参为qemu_irq结构，中断引脚，触发电平。handler为回调函数
	irq->handler(irq->opaque, irq->n, level) 
	
// 回调注册过程，就是上述分析的pc_init1过程
// 分配一组qemu_irq，其中handler为kvm_pc_gsi_handler，opaque为gsi_state 
pcms->gsi = qemu_allocate_irqs(kvm_pc_gsi_handler, gsi_state, GSI_NUM_PINS); 
kvm_pc_gsi_handler 
	// 根据中断引脚，选择不同芯片
	if(n < ISA_NUM_IRQS) { 
		qemu_set_irq(s->i8259_irq[n], level); 
	} else { 
		qemu_set_irq(s->ioapic_irq[n], level); 
	}
	
qemu_set_irq 
	irq-＞handler // 由上可知，这里的回调函数是kvm_pic_set_irq
	delivered = kvm_set_irq(kvm_state, irq, level)

// 调用kvm注入中断
int kvm_set_irq 
	// 构造中断电平和引脚
	event.level = level; 
	event.irq = irq; 
	// 通过ioct1，向kvm发送中断注入
	ret=kvm_vm_ioctl(s, s->irq_set_ioct1, &event);
	
accel\kvm\kvm-all.c 
int kvm_init 
	s->irq_set_ioctl = KVM_IRQ_LINE; 
```

kvm部分

```c
kvm: virt\kvm\kvm_main.c 
kvm_vm_ioctl 
	case KVM_IRQ_LINE: 
		r = kvm_vm_ioctl_irq_line(kvm, &irq_event, ioctl==KVM_IRQ_LINE_STATUS);
        irqchip_in_kernel(kvm)	// 判断pic是否在kvm实现
        kvm_set_irq 
        
virt\kvm\1rqchip.c 
kvm_set_irq 
	// 设置irq号到gsi号的转换
	kvm_irq_map_gsi(kvm, irq_set, irq); 
	irq_set[i].set(＆irq_set[i]， kvm， irq_source_id， level， line_status); //  set回调 

// 回调位置
e->set = kvm_set_pic_irq; 
kvm_set_pic_irq 
	kvm_pic_set_irq 
		__kvm_irq_line_state	// 计算中断信号电平并保存
		pic_set_irq1 		// 设置pic的IRR寄存器
		pic_update_irq(s);	// 更新pic-＞output
		pic_unlock(s);		// 如果需要唤醒cpu，就在vcpu挂一个KVM_NFQ_EVENT求

arch\x86\kvm\18259.c 
pic_set_irq1 
	mask = 1 << irq;
	if(s->elcr & mask) 		/*level teiggered */ 
		if(level){
			ret != (s-＞irr ＆ mask);		//  将1HR对应bit置1
			s->trr |= mask; 
			s->last_frr|= mask; 
		} else { 			// 将IRR对应bit置0
			s->irr &= ~mask; 
			5->1ast_irr &= ~mask;
		} 
	else  					/* edge triggered */ 
		if(level) {
			if((s-＞last_irr ＆ mask) ＝ 0) { 	// 如果出现了一个上升沿
				ret != (s-＞frr & mask);		 // 将IRR对应bit置1 
				s->irr |= mask; 
			}
			s->last_irr != mask; 
		} else {
			s->last_irr &= ~mask; 
		}

arch\x86\kvm\18259.c 
pic_update_irq 
	irq2 = pic_get_irq(&s->pics[1]); 
	if(irq2 >= 0)｛		// 先检查slave PIC的IRQ情况 
		// if irq request by slave pic, signal master PIC 
  		pic_set_irq1(&s->pics[0], 2, 1);  // 调用pic_irq_request使PIC的output为1 
  		pic_set_irq1(&s->pics[0], 2, 0); 
  	}
  	irq = pic_get_irq(&s->pics[0]); 
  	pic_irq_request(s->kvm, irq >= 0); 
  	
  arch\x86\kvm\i8259.c 
  pic_unlock 
  	// 如果需要唤醒cpu
  	if (wakeup) { 
  		// 遍历所有vcpu
  		kvm_for_each_vcpu(i, vcpu, s->kvm) { 
  			// 如果该vcpu可以接受pic中断信号
  			if(kvm_apic_accept_pic_intr(vcpu)) { 
  				// 向该vcpu挂KVM_REQ_EVENT请求
  				kvm_make_request(KVM_REQ_EVENT, vcpu); 
  				// 踢醒vcpu，让vcpu立刻处理终端
  				kvm_vcpu_kick(vcpu); 
 				return; 
 			}
 		}
 	}		
```

小结：qemu 通过 ioctl 向 kvm 的 pic 注入中断，kvm 收到请求后，调用相应 ioctl 函数进行 pic 设置。

具体为：设置 PIC 的 IRR 寄存器的对应 bit，然后通过 pic_get_irq 获取 PIC 的 IRR 状态，并通过 IRR 状态设置模拟 PIC 的 output。

2.pic-＞output 中断注入到 guest

在每次准备进入 Guest 时，KVM 查询中断芯片，如果有待处理的中断，则执行中断注入。设置的 pic-＞output，会在每次切入 Guest 之前被检查。

进入 VMX non-root 之前，kvm 会调用 vcpu_enter_guest 检查请求，inject_pending_event 完成中断注入。

```c
arch\xs6\kva\xs6.c 
vepu_enter_guest 
	if(kvm_check_request(KVM_REQ_EVENT, vcpu) || req_int_win) 
		inject_pending_event(vcpu)	 // 参考中断虚拟机章节
			else if (kvm_cpu_has_injectable_intr(vcpu)) { 
				if(is_guest_mode(vcpu) && kvm_x86_ops->check_nested_events) {
					r = kvm_x86_ops->check_nested_events(vcpu); 
				}
				if (kvm_x86_ops->interrupt_allowed(vcpu)) {
					kvm_queue_interrupt(vcpu, kvm_cpu_get_interrupt(vcpu), false); 
				}
				kvm_x86_ops->set_irq(vcpu);
			}
```

3.虚拟机中断注入过程

```c
vmx_inject_inq 
	vmcs_write32(VN_ENTRY_INTR_INFO_FIELD, Intr); 
```


其 vmcs 涉及到一张表：

![vmcs-interrupt](https://gitee.com/orangehaswing/blog_images/raw/master/images/VMCS_VM_Entry_field.png)

vm entry 时，kvm 将 irq(vector_number)写入了 Format of the VM-Entry Interruption-Information Field 的 bit7：0 ，并标记了该中断/异常类型为 External Interrupt，并将 bit31 置为 1，表明本次 vm entry 应该注入该中断。

在 vm entry 之后，vcpu 就会获得这个外部中断，并利用自己的 IDT 去处理该中断。

4.vCPU 处于 Guest 中或 vCPU 处于睡眠状态时的中断注入
在将中断注入 Guest 时，我们看到，在每次 vm entry 时，kvm 才会检测是否有中断，并将中断信息写入 VMCS，但是还有 2 种比较特殊的情况，会为中断注入带来延时。

1. 中断产生时，vCPU 处于休眠状态，中断无法被 Guest 及时处理。
2. 中断产生时，vCPU 正运行在 Guest 中，要处理本次中断只能等到下一次 vm exit 并 vm entry 时。

针对情况 1，即 vCPU 处于休眠状态，也就是代表该 vCPU 的线程正在睡眠在等待队列(waitqueue)中，等待系统调度运行，此时，kvm_vcpu_kick() 会将该 vCPU 踢醒，即对该等待队列调用 wake_up_interruptible()。

针对情况 2，即 vCPU 运行在 Guest 中，此时，kvm_vcpu_kcik() 会向运行该 vCPU 的物理 CPU 发送一个 IPI(核间中断)，该物理 CPU 就会 vm exit，以尽快在下次 vm entry 时收新的中断信息。

```c
virt\kvm\kvm_main.c 
kvm_vcpu_kick 
	me=get_cpu(); 
	smp_send_reschedule(cpu) 
	put_cpu(); 
	
// arch\arm64\kernel\smp．c 以arm为例 
smp_send_reschedule
	smp_cross_call
```


5.IRQn 转换为 VECTORx 

```c
arch\x86\kvm\irq.c 
kvm_cpu_get_interrupt 
	// 将irq转化为Vector
	vector = kvm_cpu_get_extint(v); 
		// arch\x86\kvm\18259.c 
		kvm_pic_read_irq(v->kvm) 	
	if (vector != -1) 
		return vector;      /*PIC */ 

kvm_pic_read_irq 
	irq ＝ pic_get_irq(＆s-＞pics[0]);		// 读取master pic 的irq
	if (irq ＞＝0) ｛				// 如果master pic上产生了中断，需要模拟CPU向PIC发送ACK信号，并设置ISR
		pic_intack(&s->pics[0], irq); 
		if (irq == 2) ｛ 		//  master pic的IRQ2连接的是slave pic的输出
			irq2 = pic_get_irq(&s->pics[1]); 
			if (irq2 >= 0) {
				pic_intack(&s->pics[1], irq2); 
			} else {
				// spurious IRQ on slave controller 
				irq2 = 7; 
			intno = s->pics[1].irq_base + irq2; 
			irq = irq2 + 8;
		} else {
			intno = s->pics[0].irq_base + irq; 	 
		}
	else {
		// spurious IRQ on host controller
        irq = 7; 
  		intno = s->pics[0].irq_base + irq; 
	}
	
pic_intack 
  // elcr的bit1为1表示IRQ1电平触发，所以这里表示在沿触发时，需要手动clear掉IRR
  if (!(s->elcr &(1 << irg)))
  	s->irr &= ~(1 << irq); 
  
  if (s->auto_eoi)｛		//  PIC在auto_eoi模式下时，无需设置ISR
  		if (s->rotate_on_auto_eoi) 
  			s->priority_add = (irq +1) & 7; 
  		pic_clear_isr(s, irq); 
  }
```

在调用 vmx_inject_irq() 之前，需要获取正确的 vector 号码，由 kvm_cpu_get_interrupt ＝＞kvm_pic_read_irq 完成

在 kvm_pic_read_irq 中，根据 IRQn 在 master 还是 slave pic 上，计算出正确的 vector 号码

1. 如果在 master 上，正确的 vector 号码＝master_pic.irq_base ＋irq_number 
2. 如果在 slave上，正确的 vector 号码＝slave_pic.irq_base ＋ irq_number

而 master 和 slave pic 的 irq_base 初始化时为 0，之后由 Guest 配置 PIC 时，通过创建 PIC 时注册的 picdev_write 函数来定义。

## 参考

1.https://www.cnblogs.com/haiyonghao/p/14440424.html

2.https://image3.slideserve.com/6132873/block-diagram-architecture-of-8259-l.jpg











































