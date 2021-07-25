# QEMU_VFIO_呈现 PCI 设备源码分析

## 获取 VFIO 设备配置空间

当 qemu 在启动过程中，会调用 vfio_realize 函数，收集直通设备的 PCI 配置空间数据。保存在 qemu 的 pdev.config，同时修改部分 PCI 配置空间数据，向虚拟机呈现。

首先是通过 ioctl 系统调用获取 config 大小和实际起始物理地址。例如 pci 大小是 256，pcie大小是 4096。

```
vfio_get_region_info(vbasedev, VFIO_PCI_CONFIG_REGION_INDEX, &reg_info)
vdev->config_síze = reg_info->size;
vdev->config_offset = reg_info->offset;
```

接着通过 vfio-pci 驱动获取 vfio 配置空间的实际数据

```
pread(vdev->vbasedev. fd, vdev- >pdev. config,
		MIN(pci_config_size(&vdev->pdev), vdev->config_size), 
		vdev->config_offset);
```

pread 实际调用 vfio-pci 驱动的 vfio-pci-read 函数。该函数的主要功能是将 vfio pci 配置空间数据从内核态拷贝到用户态。然后 qemu 从用户态读到 vfio pci space buffer。

```
drivers\vfio\pci\vfio_pci.c
vfio_pci_read
	vfio_pci_rw
		case VFIO_PCI_CONFIG_REGION_INDEX:
			return vfio_pci_config_rw(vdev, buf, count, ppos, iswrite);
				while (count) {
					vfio_config_do_rw
				}
```

## 模拟 PCI 配置空间，清除 PCI 部分字段

首先会清除 rom 的地址和 bar 地址。这是因为，在物理主机启动过程中，BIOS 将会给 vfio 设备（在没有绑定 vfio 驱动前，是一个 PCI 设备）设置 bar size，就是 PCI bar 寄存有对应的 bar 空间地址。但是，在直通给虚拟机后，这部分地址将由虚拟机 BIOS/kernel 启动阶段分配。因此，需要先把bar寄存器内容清空。

```
/* QEMU can choose to expose the ROM or not */
memset(vdev->emulated_config_bits+PCI_ROM_ADDRESS, 0xff, 4);
/*QEMU can also add or extend BARs */
memset(vdev->emulated_config_bits+ PCI_BASE_ADDRESS_0, 0xff, 6 * 4);

/*
* Clear host resource mapping info. If we choose not to register a
* BAR, such as might be the case with the option ROM, we can get
* confusing, unwrítable, residual addresses from the host here.
*/
memset(&vdev->pdev.config[PCI_BASE_ADORESS_0], 0, 24);
memset(&vdev->pdev.config[PCI_ROM_ADORESS], 0, 4);
```

对于 PCI 配置空间的部分字段，将在 qemu 模拟，例如 vendor_id，device_id，sub_vendor_id，sub_device_id ，Single/Multi Function 字段。 

下图展示了 PCI 配置空间的布局：

![pci_space](.\pci_space.jpeg)

## 复位COMMAND寄存器和设备

清除 command 的函数在 vfio_pci_reset。主要作用是复位 command。

```
vfio_pci_reset
	vfio_pci_pre_reset
		cmd =vfio_pci_read_config(pdev, PCI_COMMAND, 2);
		cmd &=~(PCI_COMMAND_IO |PCI_COMMAND_MEMORY|PCI_COMMAND_MASTER | PCI_COMMAND_INTX_DISABLE);
		vfio_pci_write_config(pdev, PCI_COMMAND, cmd, 2);

	ioctl(vdev->vbasedev.fd, VFIO_DEVICE_RESET))
```

command 寄存机的标记了设备状态。

![pci_command](.\pci_command.png)

如上图所示，只有bit 0, 1, 2, 6, 8, 10 有效。IO space 和 MEM space 分别用于控制响应 bar IO 和 mem 空间的访问请求。Interrupt disable 用于控制设备是否产生 INTx 中断。

## VFIO配置空间读写

前端访问 0xcf8 和 0xcfc，最终调用 vfio_pci_read_config 和 vfio_pci_write_config。写配置会同时往 vfio-pci 驱动和 qemu 写，向驱动写的过程，由 vfio-pci 驱动决定需要保留哪部分数据。读配置根据需要从 vfio-pci 或 qemu 读，由vfio_realize阶段的模拟标志位决定。

```
vfio_pci_read_config
	// 从qemu读
	if(emu_bits) {
		emu_val = pci_default_read_config(pdev, addr, len);
	}
	
	// 从vfio-pci驱动读配置
	if (~emu_bits & (ØxffffffffU >> (32 - 1en * 8))) {
		ssize_t ret;
		ret=pread(vdev->vbasedev.fd, &phys_val, 1en, vdev->config_offset +addr);
		phys_val=1e32_to_cpu(phys_val);
	}
	
vfio_pci_write_config
	// 直接向vfio 设备写，由物理设备配置空间mask决定，哪些能写
	pwrite(vdev->vbasedev.fd, &val_le, 1en, vdev->config_offset + addr)

	//  如果写mis
	if (ranges_overlap(addr, len, pdev->msi_cap,vdev->msi_cap_size)) 
		// 用来判断msi是否已经初始化
		is_enabled = msi_enabled(pdev) 
		// 向qemu 配置空间写一份
		pci_default_write_config(pdev, addr, val, len) 
		// 由当前设备状态决定对msi操作
		vfio_msi_enable(vdev) / vfio_update_msi / vfio_msi_disable 

	# 如果写msix
	else if (ranges_overlap(addr,len,pdev->msix_cap,MSIX_CAP_LENGTH))
		int is_enabled,was_enabled =msix_enabled(pdev)
		pci_default_rite_config(pdev, addr, val, len)
		is_enabled·msix_enabled(pdev)
		// 这望会根据写前后的状态，决定使能msix还是重置msix。这是因为enable msix的过程中，会向KVM注册irgevenfd，vfio中断事件将触发写irqfd，因此，当disable msix，需要解绑这种状态。
		vfio_msix_enable(vdev) / vfio_msix_disable
		// 如果写bar 或command区域
		else 1f(ranges_overlap(addr,1em,PCI_BASE_ADORESS_e,24) Il range_covers_byte(addr,len, PCI_COMAND)) 
			pci_default_urite_config(pdev, addr, wal, len); 
			for(bar = 0; bar < PCI_ROM_SLOT; bar++) { 
				//  对bar 做update mmap 
				vfio_sub_page_bar_update_mapping(pdew, bar): 
			}
```

## 虚拟机读写数据

这里打印了 qemu 启动 vfio 设备过程中，向 PCI 配置空间读写数据的过程。

第一阶段：虚拟机读 PCI 配置空间，这时还会从设备读取 command，cap 信息。

```
pci 0000:00:01.0:[1234:1111] type 00 class 0x030000
pci e000:00:01.0: reg 0x10: [men exc0000000-0xc0ffffff pref]
pci 00ee:00:01.0: reg 0x18: [men @xc1012000-0xc1012fff]
pci e000:00:01.0: reg 0x30: [men exffff0000-0xffffffff pref]
```

read contig

|      addr      |       val（十六进制）       |
| :------------: | :-------------------------: |
|       0        |          37d18086           |
|       14       |              0              |
|       6        |             10              |
|       52       |             40              |
|       64       |            5004             |
|  80(MSI地址)   | 7005（05 表示 msi cap id）  |
| 112(MSI-X地址) | a011（11 表示 msix cap id） |
|      160       |            e010             |
|      162       |             92              |
|      164       |            8ce2             |
|       8        |           2000009           |
|      256       |          14020001           |
|       0        |          37d18086           |
|      256       |          14020001           |
|      320       |          1a010003           |
|      416       |          1b010017           |
|      432       |            1000d            |

第二阶段：给 bar 空间分配 gpa 地址。bar 寄存器的有些 bit 位是只读的，因此在在写地址之前，会写 0xffffffff，如果值没有变化，那边这些 bit 位就是只读。可变 bit 区域就会被清除，然后重新写入 bar size。

read && wnite bar register

| addr read/write | val（十六进制） |
| :-------------: | :-------------: |
|     4 read      |       146       |
|     4 write     |       546       |
|     4 read      |       546       |
|     4 write     |       146       |
|     4 read      |       146       |
|     4 write     |       144       |
|     16 read     |        0        |
|    16 write     |    ffffffff     |
|     16 read     |    ff00000c     |
|    16 write     |        C        |
|     20 read     |        8        |
|    20 write     |    ffffffff     |
|     20 read     |    ffffffff     |
|    20 write     |        0        |
|     24 read     |       ...       |

```
pci 0000:00:02.0: reg 0x10: [mem 0x800000000-0x800ffffff 64bit pref]
```

bar0 的地址值复制完成了。为什么这里给 offset 16 和 20 都赋值了? 因为设备是一个 64 位 MEM bar。因此，将 bar 0 和 bar 1 合并起来，作为一个 64 位的 bar。

bar 的结构参考下图：

![bar_regsier](.\bar_regsier.png)


同样的，将 bar3，bar4 视作一个 bar，赋值 bar size

```
pci 0000:00:02.0:reg 0x1c:[mem 0x801000000-0x801007fff 64bit pref]
```

当将所有的 bar 地址写入后，会向 COMMAND 寄存使能 MEM_SPAC E位.

即：

```
offset：4 read val 144 二进制：10010000
offset：4 write val：146 二进制：10010010
```

第三阶段：使能 MSI-X，根据 MSI-X 的 cap offset 可知，offset＝112。在这个过程中，会先读取 MSI-X cap 信息，然后使能标志位。

```
offset:114 read    		val 80
offset:114 write     	val:  c080
offset:114 read    		val:  c080
offset:114 write    	val:  8080
```

MSI-X cap 结构如下：

![MSIX_cap](.\MSIX_cap.png)

其中 msg ctl 字段含义如下：

![MSIX_msgctl](.\MSIX_msgctl.png)

由上可以知道，当写入 0xc080 时，会将第 14,15 置位，即屏蔽所有中断，使能 MSI-X。当写入 0x8080 时，会复位全局 Mask，可以进行中断请求。

在这个阶段就会进入 vfio_msix_enable(vdev) / vfio_msix_disable。该过程将写入的 MSI-X message 作为 route，更新到全局routes 路由表，并向 kvm 注册。这样 kvm 就可以通过 gsi 找到对应的中断路由表。

在第一次进入vfio_msix_enable时，会分配一个 irqfd，并向 kvm 注册，这样就将中断和 irqfd 绑定，当中断来的时候，irqfd 会被写，kvm 将会一直监听 irqfd。另外还会做的一步是将 gsi 和 irqfd 绑定，注册到 kvm。当 irqfd 被写，kvm 就知道 irqfd 对应的 gsi 有中断事件，最终根据中断路由表，将中断注入到虚拟机。

## 设备驱动加载

当虚拟机在 PCI 设备枚举完，就给设备的 PCI 配置空间分配了必要信息。接下去，就会进入驱动加载过程。以 i40e 网卡为例，加载驱动的函数是 i40e_probe

```
drivers\net\ethernet\intel\i40e\i4Øe_main.c
i40e_probe
	// 对pci配置空间进行读写，即设置上述的command寄存器值，使能mem space标志位。
	pci_enable_device_mem 
	pci_request_mem_regions 
	// 从pci配置空间读取厂商ID，设备ID等，联系上述模拟PCI配置空间过程，这里将获取qemu内的pci数据信息。
	hw->vendor_id =pdev->vendor; 
	hw->device_id = pdev->device; 
	hw->subsystem_vendor_id=pdev->subsystem_vendor;
	hw->subsystem_device_id = pdev->subsystem_device;
	// 从设备驱动获取设备固件信息
	i4ee_init_adming
		i4ee_aa_get_firmware_version
		i4ee_read_nvm_word
		i40e_get_oem_version
	
	// 初始化msix
	i40e_init_interrupt_scheme
		i40e_init_msix
```

该部分的代码和上述虚拟机向 PCI 配置空间的读写数据对应上了。在获取 bar 寄存器地址后，虚拟机驱动程序将会从 bar 空间获取对应的数据。因为在 vfio_realize 函数中，已经建立起 bar 空间和物理设备的内存。虚拟机将会通过 EPT 页表，直接访问到物理设备。

i40e 的原理是向物理设备固定的寄存器写 command 命令，物理设备将固件信息通过 DMA 操作，发送到 guest memory。然后向虚拟机返回存放数据的地址值命令，虚拟机在收到命令后，取出地址值，去相应的虚拟机内存取出固件信息。