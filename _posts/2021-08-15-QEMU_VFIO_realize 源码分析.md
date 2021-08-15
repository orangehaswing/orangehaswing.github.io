---
layout: post
title: QEMU_VFIO_realize 源码分析
date: 2021-07-25
tags: jekyll   
---

## VFIO 编程介绍

VFIO 接口分为3类，分别是container， group， device。
第一类是 container 层面，通过打开 /dev/vfio/vfio 设备可以获得一个新的 container，包括如下 ioctl：

1. VFIO_GET_API_VERSION：报告 VFIO API 版本
2. VFIO_CHECK_EXTENSION：检测支持特定的扩展，如哪个 IOMMU
3. VFIO_SET_IOMMU：指定 IOMMU 类型，必须是通过 VFIO_CHECK_EXTENSION 确认驱动支持的
4. VFIO_IOMMU_GET_INFO：获得 IOMMU 信息，只针对 Type1 的 IOMMU
5. VFIO_IOMMU_MAP_DMA：指定设备 IO 地址到进程的虚拟地址间的映射

第二类是 group 层面，通过打开 /dev/vfio/groupid 得到一个 group，包括如下 ioctl：

1. VFIO_GROUP_GET_STATUS：得到 group 信息，如是否可用，是否设置了 container
2. VFIO_GROUP_SET_CONTAINER：设置 container 和 group 之间的管理，多个 group 可以属于单个 container
3. VFIO_GROUP_GET_DEVICE_FD：返回一个新的文件描述符 fd，描述具体设备

第三类是设备层面，其 fd 由 VFIO_GROUP_GET_DEVICE_FD 接口返回，device 的 ioctl 包括如下：

1. VFIO_DEVICE_GET_REGION_INFO：得到设备的指定 region 数据，这里的 region 包括 BAR，ROM 空间，PCI 配置空间
2. VFIO_DEVICE_GET_IRQ_INFO：得到设备的中断信息
3. VFIO_DEVICE_RESET：重置设备

## 实现设备直通

vfio_realize 是 VFIO 设备的具体实现函数 (/hw/vfio/pci.c)

该函数分为如下几个部分：

1. 通过指定的设备路径得到其在 IOMMU 中的所属 group。qemu 在配置 vfio 设备时，设置入参为：-device vfio-pci,host＝0000:1a:00.3。所以这里将设备的 domain, bus, slot, function 复制到 sysfsdev 中。 

   ```
   vdev->vbasedev.sysfsdev = 
   		g_strdup_printf("/sys/bus/pci/devices/%@4x:%02x:%02x.%01x",vdev->host.domain, 
   		vdev->host. bus,vdev->host.slot,vdev->host,function);
   		vdev->vbasedev.name = g_path_get_basename(vdev->vbasedev.sysfsdev);
   ...
   vdev->vbasedev.ops =&vfio_pci_ops;
   vdev->vbasedev.type =VFIO_DEVICE_TYPE_PCI;
   vdev->vbasedev.dev = DEVICE(vdev);
   ...
   tmp = g_strdup_printf("%s/iommu_group",vdev->vbasedev.sysfsdev); 
   len = readlink(tmp, group_path,sizeof(group_path)); 
   ...
   group_name=basename(group_path);
   if (sscanf(group_name,"%d",&groupid) != 1){
   	...
   )
   ```

   主要工作：
   sysfsdev 赋值为 /sys/bus/pci/devices/＋domain＋设备 bdf 号，例如通过上面的程序，sysfsdev 变成了 "/sys/bus/pci/devices/0000:1a:00.3"。之后，VFIOPCIDevice 的设备名，回调函数，设备类型在这时候被赋值。

   接着，temp 路径设置为 sysfsdev/iommu_group，例如 /sys/bus/pci/devices/0000:1a:00.3/iommu_group/，该目录就是 iommu group组，通过查看该目录下的 devices 文件夹，就可以查看有多少个设备是属于同一组的。iommu_group 是一个链接文件，指向iommu_group，最后得到 group_path 的 basename，存放到 groupid 中。该 group id 和 /dev/vfio/ 目录下设备数字对应。

2. 打开 groupid 设备，并连接到 container

   ```
   group = vfio_get_group(groupid, pci_device_iommu_address_space(pdev), errp);
   ```

   该函数将会完成创建 group，并且连接到 vfio container 的功能。但是如果 group 已经存在，会在遍历 group list 之后，直接返回group，不再继续往下执行。调用关系如下：

   ```
   vfio_get_group 
   	＃查找list，如果group已经存在，返回
   	QLIST_FOREACH(group,&vfio_group_list, next){ 
   		...
   	}
   
   	＃ 获取直通设备的文件句柄fd
   	group->fd =qemu_open("/dev/vfio/$groupid") 
   	
   	＃判断设备的可用状态，确保所有设备都绑定到同一个group组
   	ioctl(group->fd, VFIO_GROUP_GET_STATUS, &status) 
   	status.flags & VFIO_GROUP_FLAGS_VIABLE 
   	
   	＃ 将group连接到container
   	vfio_connect_container
   		＃ 查找container，如果container存在，直接连接
   		QLIST_FOREACH(container, &space->containers, next) 
   			ioctl(group->fd, VFIO_GROUP_SET_CONTAINER, &container->fd) 
   
   		＃如果没有container，获取设备句柄
   		fd = qemu_open_old("/dev/vfio/vfio", O_RDWR) 
   
   		＃ 检查API 版本
   		ioctl(fd, VFIO_GET_API_VERSION) 
   
   		＃ 初始化container
   		vfio_init_container
   			ioctl(group_fd, VFIO_GROUP_SET_CONTAINER, &container->fd)
   			ioctl(container->fd, VFIO_SET_IOMMU, iommu_type)
   		ioctl(fd, VFIO_IOMMU_GET_INFO,&info)
   		＃向kvm vfio dev添加group
   		vfio_kvm_device_add_group
   			ioctl(vfio_kvm_device_fd, KVM_SET_DEVICE_ATTR,&attr)
   		＃向address_space注册内存监听器，这部分涉及VFIO iommu dma map虚拟机内存，在文章最后详细分析
   		memory_listener_register 
   ```

   主要工作：
   a. 打开 /dev/vfio/$groupid 得到直通设备所属 group id
   b. 判断设备的可用状态
   c. 调用 vfio_init_container 设置 group 的 container。注意：在这过程中会注册内存监听器，监听虚拟机内存的状态改变，并完成直通设备的 DMA 地址到虚拟地址的映射

3. 得到直通设备 fd

   ```
   vfio_get_device
   	＃根据group fd和和device name，得到device fd
   	fd = ioctl(group->fd, VFIO_GROUP_GET_DEVICE_FD, name)
   
   	＃得到直通设备的基本信息：irq数量，region数量，flag信息。
   	ioctl(fd,VFIO_DEVICE_GET_INFO,&dev_info) 
   	vbasedev->fd =fd; 
   	vbasedev->num_irqs = dev_info.num_irqs; 
   	vbasedev->num_regions = dev_info.num_regions; 
   	vbasedev->flags = dev_info.flags; 
   ```

   主要工作：通过 ioctl 获得直通设备的 fd，然后得到设备的基本信息，包括：irq 数量，region 数量，flags。其中 irq 数量反映了 vfio 设备最多会用多少个 irqfd 作为中断通知，regio n的 0～5 region 表示 vfio 设备的 bar，region6 表示 rom 空间，region 7 表示 pci config，region 8 表示 VGA。因此，设备通常有9个 region。flags 反映了设备是否支持 region mmap，mmap 的 port read/write， cap 信息。 

4. 将直通设备的信息取出来，包括 bar region 参数，pci 配置，irq 信息。对于 vfio 物理设备 bar size 不为 0 的区域，将会在 qemu 建立对应的 bar region。

   ```
   vfio_populate_device
   	＃检查是否为PCI设备，设备irq和region数量是否正常
   	...
   	
   	＃ 获取Bar region信息，创建region
   	for (i = VFIO_PCI_BAR0_REGION_INDEX; i < VFIO_PCI_ROM_REGION_INDEX; i++){ 
   		vfio_region_setup 
   			＃ ioctl获取bar region 信息
   			vfio_get_region_info
   				＃获取直通设备的region信息：flags， cap_offset， size， offset 
   				ioctl(vbasedev->fd, VFIO_DEVICE_GET_REGION_INFO, *info) 
   				＃如果argsz大于vfio_region_info size，则扩展info大小
   				if ((*info)->argsz > argsz){
   					argsz = (*info)->argsz;
   					*info =g_realloc(*info,argsz);
   				}
   
   			region->flags = info->flags;
   			region->size = info->size;
   			region->fd_offset = info->offset;
   			region->nr = index;
   
   			＃如果region size不为0，则为该vfio在虚拟机创建对应的bar region内存
   			if(region->size){
   				＃创建VFIO虚拟设备的region，设置vfio_region_ops读写回调函数。但是直通设备的BAR会直接映射给虚拟机，所以访问的时候不会导致VM-exit，不会调用到vfio_region_ops。
   				# 有一种情况例外，因为msix不能mmap，当msix 不能mmap的长度 ＞＝region size，虚拟机会陷出到bar region，调用vfio_region_ops回调函数。
   				region->mem =g_new0(MemoryRegion, 1);
   				memory_region_init_io(region->mem, obj, &vfio_region_ops, region, name, region->size)
   				
   				＃如果这个region需要mmap，就截取mmap需要的region offset和size信息
   				if(region->flags & VFIO_REGION_INFO_FLAG_MMAP) {
   					＃ 如果region中间有空洞（稀疏内存），就分别将需要mmap的区间 offset，size取出保存，目的是更细粒度管理 bar region
   					vfio_setup_region_sparse_mmaps(region, info)
   						for (i = 0, j = 0; i < sparse->nr_areas; i++) {
   							if (sparse->areas[i].size){
   								region->mmaps[j].offset = sparse->areas[i].offset;
   								region->mmaps[j].size = sparse->areas[i].size;
   								j++;
   							}
   						}
   						region->nr_mmaps = j;
   					＃ 如果没有空洞，就mmap 这个region所有区间
   					if (ret){
   						region->nr_mmaps = 1;
   						region->mmaps[0].offset =0;
   						region->mmaps[0],size = region->size;
   					}
   
   ＃获取 PCI 配置空间信息，需要获取大小以及在设备fd文件中的偏移
   vfio_get_region_info(vbasedev, VFIO_PCI_CONFIG_REGION_INDEX, &reg_info)
   	ioctl(vbasedev->fd, VFIO_DEVICE_GET_REGION_INFO,*info)
   	vdev->config_size = reg_info->size;
   	vdev->config_offset = reg_info->offset;
   
   ＃获取irq 信息，VFIO_PCI_ERR_IRQ_INDEX表示内核支持一个设备发生不可恢复错误时，通知qemu
   irq_info.index = VFIO_PCI_ERR_IRQ_INDEX;
   ioctl(vdev->vbasedev.fd, VFIO_DEVICE_GET_IRQ_INFO, &irq_info)
   ```

   主要工作：
   a. 在一个 for 循环对6个 BAR 依次获取 vfio 设备的 BAR 信息 (VFIO_PCI BAR0_REGION_INDEX～
   VFIO_PCI_BAR6_REGION_INDEX)，并对 region size＞0 创建对应的 bar region。 
   b. vfio_get_region_info 获取 PCI 配置空间，VFIO_PCI_CONFIG_REGION_INDEX 是 PCI 配置空间的索引，需要获取其大小以及设备 fd 文件的偏移。
   c. 获取中断信息

   到这步为止，所有的 vfio 设备信息都获取到了，接下来要做的是，利用 vfio-pci 驱动，向虚拟机呈现一个 PCI 设备。这样，虚拟机就可以在启动阶段，驱动会识别到对应的直通设备，可以直接使用。

5. 获取设备内存空间资源后，进行设备 PCI 配置空间处理

   ```
   ＃ 从vfio直通设备获取一份配置空间，放在pdev.config.pread会调用vfio-pci驱动的read函数返回设备的pci配置
   ret = pread(vdev->vbasedev.fd, vdev->pdev.config, 
   			MIN(pci_config_size(&vdev->pdev), vdev->config_size), 
   			vdev->config_offset); 
   
   ＃当虚拟机内部访问配置空间，如果该部分内容由qemu模拟，就把emulated_config_bits设置对应地址的字节（相应位置1），直接访问pdev．config．否则需要读取实际vfio设备的配置空间。
   vdev->emulated_config_bits = g_malloce(vdev->config_size);
   
   /* QEMU can choose to expose the ROM or not */
   memset(vdev->emulated_config_bits + PCI_ROM_ADDRESS, 0xff, 4); 
   /* QEMU can also add or extend BARS */ 
   memset(vdev->emulated_config_bits + PCI_BASE_ADDRESS_0, 0xff, 6*4); 
   
   ＃写入设备的配置空间数据和控制数据，使之向虚拟机呈现完整的PCI设备模样
   vfio_add_emulated_word(vdev, PCI_VENDOR_ID, vdev->vendor_id, ~0) 
   vfio_add_emulated_word(vdev, PCI_DEVICE_ID, vdev->device_id, ~0)
   vfio_add_emulated_word(vdev, PCI_SUBSYSTEM_VENDOR_ID, vdev->sub_vendor_id, ~0) 
   vfio_add_emulated_word(vdev, PCI_SUBSYSTEM_ID, vdev->sub_device_id, ~0) 
   
   ＃清理host侧 bar 和 rom mapping 信息，因为从物理设备获取的bar size是由物理bios分配，在虚拟机启动阶段，会重新为bar region分配gap地址，将gpa地址放在bar寄存器。因此，在这里需要提前清除。rom也是同样原因。
   memset(&vdev->pdev.config[PCI_BASE_ADDRESS_0], 0, 24) 
   memset(&vdev->pdev.config[PCI_ROM_ADDRESS], 0, 4) 
   
   vfio_pci_size_rom
   	＃为有rom的直通设备创建region，设置回调函数vfio_rom_ops
   	memory_region_init_io(&vdev->pdev. rom, OBJECT(vdev), &vfio_rom_ops, vdev, name,size) 
   	pci_register_bar    ＃ 注册bar 
   	
   ＃ 用到的模拟函数设置了pci掩码，写掩码，模拟标志位置位
   vfio_add_emulated_word
   	vfio_set_word_bits(vdev->pdev.config + pos, val, mask)
   	vfio_set_word_bits(vdev->pdev.wmask + pos,~mask,mask)
   	vfio_set_word_bits(vdev->emulated_config_bits + pos, mask, mask)
   ```

   主要工作：
   a. 调用 pread，根据 offset 和 size 信息，获取直通设备的配置空间，复制到 pdev.config 中
   b. vfio_add_emulated_word 写入设备的配置空间及控制数据，对配置空间做细微调整，是之呈现完整的 PCI 模样
   c. 调用 vfio_pci_size_rom 处理设备 rom，创建一个 MemoryRegion，注册成虚拟设备的 rom（如果直通设备有 rom）

6. 获取虚拟设备 BAR，创建 region 并注册为 bar。

   ```
   ＃获取bar寄存器信息，包括IO/MEM，MEM32/MEM64类型，是否支持预取，bar大小。
   vfio_bars_prepare 
   	for (i = 0; i < PCI_ROM_SLOT; i++) { 
   		vfio_bar_prepare(vdev, i); 
   	}
   	
   vfio_bar_prepare
   	＃ 如果bar region size为0，则不需要配置bar的类型
   	if (!bar->region.size) {
   		return;
   	}
   	
   	＃ 读取BAR寄存器的值到pci_bar，只读取前四个字节。
   	pread(vdev->vbasedev.fd, &pci_bar, sizeof(pci_bar),
   	vdev->config_offset + PCI_BASE_ADDRESS_0 + (4*nr))
   	＃ 根据pci_bar数据，赋值bar类型，包括IO/MEM，大小，是否是64位
   	bar->ioport
   	bar->mem64
   	bar->type
   	bar->size
   	
   ＃经过此步骤后，前端访问BAR空间，直接走EPT页表，不会导致VM-exit
   vfio_bars_register
   	for (i = 0; i < PCI_ROM_SLOT; i++){
   		vfio_bar_register(vdev, i);
   	}
   	
   vfio_bar_register
   	＃如果没有bar size，即vfio region size 为0，就不需要创建region
   	if (!bar->size) {
           return;
        }
        
   	bar->mr =g_new0(MemoryRegion, 1); 
   	＃创建region，类型为IO region 
   	memory_region_init_io(bar->mr, OBJECT(vdev), NULL, NULL, name,bar->size) 
   	＃设置region的mr为bar的子region，该region就是前面提到的可以mmap的region
   	memory_region_add_subregion(bar->mr, 0, bar->region.mem) 
   
   	＃建立host和guest内存映射
   	vfio_region_mmap
   		＃ 会调用vfio-pci驱动的mmap函数，mmaps[i].mmap表示得到的hva地址。其中的细节：mmap地址偏移region-＞fd_offset ＋ region-＞mmaps[i].offset， 表示region offset + mmap offset，即起始位置需要做一次加法，得到hpa实际需要mmap地址位置。
   		region->mmaps[i].mmap = mmap(NULL, region->mmaps[i].size, prot,
   									MAP_SHARED, region->vbasedev->fd,
   									region->fd_offset + region->mmaps[i].offset);
   		
   		＃创建ram device的region类型给虚拟机
   		memory_region_init_ram_device_ptr
   			memory_region_init(mr, owner, name, size);
   			＃为true，当pci枚举时，就会将此mr添加到FlatView中，进而注册到kvm建立EPT页表，这既是上述说的前端访问bar，可以直接走EPT页表，不会陷出的原因。
   			mr->ram = true;
   			mr->terminates = true;
   			mr->ram_device = true;
   			mr->ops = &ram_device_mem_ops;
   			mr->opaque = mr;
   			mr->destructor = memory_region_destructor_ram;
   			mr->ram_block=qemu_ram_alloc_from_ptr(size, ptr, mr, &error_fatal);
   		
   		＃ 将region mem添加为子region
   		memory_region_add_subregion
   		
   		＃将bar-＞region.mem赋值给pci_dev-＞io_regions[i].memory，此时并没有添加到pci_dev-＞io_regions[i]．address_space，添加到address_space是在前端通过0xcf8、Oxcfc写BAR地址的时候
   		＃ 向qemu config配置空间注册一份bar region信息
   		pci_register_bar
   ```

   主要工作：获取 bar 参数，创建 bar region，对 bar 的 region 做 mmap 映射。

7. 中断初始化和中断使能，这里以 MSI-X 为例

   ```
   ＃该函数上面有段注释，说明了vfio设备不允许mmap msix table区域(高级特性中，msix支持mmap，由设备决定), 同时不知道pci_add_capability函数是否会将msix插入，因此这里做msix early处理。
   vfio_msix_early_setup
   	＃ 判断直通设备是否支持misx能力
   	pci_find_capability(&vdev->pdev,PCI_CAP_ID_MSIX) # PCI_CAP_ID_MSIX=0x11
   	＃获取misx table，pba，信息
   	pread（fd，＆ctrl， sizeof(ctrl)， vdev-＞config_offset ＋ pos ＋ PCI_MSIX_FLAGS）＃读取 PCI_MSIX_FLAGS=2 
   	pread(fd, &table, sizeof(table), vdev->config_offset + pos + PCI_MSIX_TABLE) # PCI_MSIX_TABLE=4 
   	pread(fd, &pba, sizeof(pba), vdev->config_offset + pos +PCI_MSIX_PBA)	# PCI_MSIX_PBA=8
   	
   	msix-＞table_bar＝ table＆PCI_MSIX_FLAGS_BIRMASK；	＃取出msix table映射的BAR index
   	msix-＞table_offset ＝ table ＆～PCI_MSIX_FLAGS_BIRMASK；	＃取出table在BAR的offset
   	msix-＞pba_bar ＝ pba ＆PCI_MSIX_FLAGS_BIRMASK；	＃取出pba映射的BAR index
   	msix-＞pba_offset ＝ pba＆～PCI_MSIX_FLAGS_BIRMASK；	＃取出pba在BAR的offset
   	msix-＞entries ＝（ctrl ＆ PCI_MSIX_FLAGS_QSIZE）＋1；	＃设置msix vector中断向量数
   
   	＃将msix_table映射的BAR区间从vdev-＞bars[table_bar]-＞region-＞mem中抠除，按页对齐，pba不用抠除。ARM不会整改，但是msix的bar空间也一样去从region中抠除，抠除的大小是vdev-＞entries＊16，然后以host page对齐。
   	vfio_pci_fixup_msix_region
   	＃页表对齐
   	start = vdev->msix->table_offset & qemu_real_host_page_mask; 
   	end = REAL_HOST_PAGE_ALIGN((uint64_t)vdev->msix->table_offset + (vdev->msix->entries * 													PCI_MSIX_ENTRY_SIZE)); 
   	
   vfío_add_capabilíties
   	vfio_add_std_cap
   		＃ 初始化msix
   		vfio_msix_setup
   			＃ table_bar_pre ＝ table_bar等于msix被映射的BAR index
   			msix_init()
   				＃ 获取table size
   				table_size = nentries * PCI_MSIX_ENTRY_SIZE;
   				＃ 向pci添加msix，msix cap id为0x11
   				pci_add_capability(dev, PCI_CAP_ID_MSIX,cap_pos, MSIX_CAP_LENGTH, errp);
   				pdev-＞msix_cap ＝ cap ＃ cap是在pdev-＞config的偏移
   				pdev->cap_present|= QEMU_PCI_CAP_MSIX # QEMU_PCI_CAP_MSIX = 0x11
   
   				＃设置qemu模拟的pci config相应值
   				pci_set_word(pdev->config + cap + PCI_MSIX_FLAGS, entries-1)
   				pdev->msix_entries_nr = entries
   				pdev->msix_function_masked = true
   				pci_set_long(pdev->config + cap + PCI_MSIX_TABLE, table_offset | table_bar)
   				pci_set_long(pdev->config + cap + PCI_MSIX_PBA, pba_offset | pba_bar)
   				pdev-wmask = pdev->msix_table=g_malloc0(table_size)
   				pdev->msix_pba = g_malloc0(pba_size)
   				msix_mask_all（pdev，entries）	＃设置pdev-＞msix_table为原始mask状态
   			＃当前端访问BAR中msix区间时，会vM-exit，调用msix_table_mmio_ops，下文具体分析
   			memory_region_init_io(&pdev->msix_table_mmio, OBJECT(qdev), &msix_table_mio_ops,dev, 
   								"msix-table", table_size) 
   			memory_region_init_io(&pdev->msix_pba_mmio, OBJECT(qdev), &msix_pba_mmio_ops, dev, 
   								"msix-pba",pba_síze)
   			memory_region_add_subregion(vdev->bars[vdev->msix->table_bar_pre].region.mem, 
   								table_offset, &dev->msix_table_mmio) 
   			memory_region_add_subregion(vdev->bars[vdev->msix->pba_bar_pre].region.mem, 
   								pba_offset, &dev->msix_pba_mmio) 
   		vfio_add_ext_cap
   ```
   
   主要工作：
   a. 获取必要的信息，如 table_bar， table_offset， pba_bar， pba_offset 等在 BRA 空间的偏移量。
   b. 完成 MSI-X 的初始化工作：vfio_msix_setup 完成直通设备 MSI-X 初始化，调用 pci_add_capability 为设备添加PCI_CAP_ID_MSIX Capability，注册 MSI-X 的 BRA 空间到虚拟机物理地址空间。同时也为虚拟机设备添加 Capability
   c. 完成 INTx 的 enable，虚拟设备的中断初始化。

bar 中扣除 table table 的具体执行过程

```
if (!start) {
        if (end >= region->size) {
            region->nr_mmaps = 0;
            g_free(region->mmaps);
            region->mmaps = NULL;
            trace_vfio_msix_fixup(vdev->vbasedev.name,
                                  vdev->msix->table_bar, 0, 0);
        } else {
            region->mmaps[0].offset = end;
            region->mmaps[0].size = region->size - end;
            trace_vfio_msix_fixup(vdev->vbasedev.name,
                              vdev->msix->table_bar, region->mmaps[0].offset,
                              region->mmaps[0].offset + region->mmaps[0].size);
        }

    /* Maybe it's aligned at the end of the BAR */
    } else if (end >= region->size) {
        region->mmaps[0].size = start;
        trace_vfio_msix_fixup(vdev->vbasedev.name,
                              vdev->msix->table_bar, region->mmaps[0].offset,
                              region->mmaps[0].offset + region->mmaps[0].size);

    /* Otherwise it must split the BAR */
    } else {
        region->nr_mmaps = 2;
        region->mmaps = g_renew(VFIOMmap, region->mmaps, 2);

        memcpy(&region->mmaps[1], &region->mmaps[0], sizeof(VFIOMmap));

        region->mmaps[0].size = start;
        trace_vfio_msix_fixup(vdev->vbasedev.name,
                              vdev->msix->table_bar, region->mmaps[0].offset,
                              region->mmaps[0].offset + region->mmaps[0].size);

        region->mmaps[1].offset = end;
        region->mmaps[1].size = region->size - end;
        trace_vfio_msix_fixup(vdev->vbasedev.name,
                              vdev->msix->table_bar, region->mmaps[1].offset,
                              region->mmaps[1].offset + region->mmaps[1].size);
    }
```

## MSI-X 中断的读写操作

```
＃ 注册了msix_table_mmio_ops
memory_region_init_io(&pdev->msix_table_mmio, OBJECT(qdev), &msix_table_mmio_ops, dev, "msix-table", table_size) 

static const MemoryRegionOps msix_table_mmio_ops = {
	.read = msix_table_mmio_read,
	.write = msix_table_mmio_write,
}

＃ misx read操作，直接从table读出数据
msix_table_mmio_read
	pci_get_long(dev->msix_table + addr);
＃ misx write 向qemu misx table写入数据，另外对中断路由设置或更新
msix_table_mmio_write
	＃获取上次msix mask状态
	was_masked = msix_is_masked(dev, vector);
	# 向qemu msix table写val
	pci_set_long(dev->msix_table + addr, val);
	msix_handle_mask_update
		msix_fire_vector_notifier
        	dev-＞msix_vector_release_notifier ＃ 回调函数
			＃如果这次msix pending，则向kvm发送信号通知
			if (lis_masked 8& msix_is_pending(dev, vector)) {
				msix_clr_pending(dev, vector)
				msix_notify(dev, vector)
					msg=msix_get_message
					＃向kvm发送msg
					msi_send_message
			}

＃这里简单列举注册回调函数和回调函数关系，后面文章具体介绍
＃注册过程
vfio_pci_write_config
	vfio_msix_enable
		msix_set_vector_notifiers(&vdev->pdev, vfio_msix_vector_use, vfio_msix_vector_release, NULL)
			dev-＞msix_vector_use_notifier ＝ use_notifier； ＃ 注册vfio_msix_vector_use
			dev->msix_vector_release_notifier = release_notifier;
			dev->msix_vector_po1l_notifier = poll_notifier;

vfio_msix_vector_use
	vfio_msix_vector_do_use
		if (!vector->use) {
			vector->virq = -1;
			event_notifier_init(&vector->interrupt, 0)
		}
		
		vfio_remove_kvm_msi_virq
		vfio_update_kvm_msi_virq
			vfio_add_kvm_msi_virq
			if (vdev->nr_vectors < nr + 1){
				vdev->nr_vectors =nr + 1
        		 vfio_enable_vectors
			}
```

## 设备 I/O 地址空间模拟

上述过程是实现设备直通的基本流程，在 vfio_populate_device 函数调用阶段，会调用 vfio_region_setup 虚拟机的所有 BAR 信息都存放在虚拟设备结构体 VFIOPCIDevice 的 bars 数组成员中，类型为 VFIOBAR。其中有一个重要成员 VFIORegion，存放虚拟设备的 BAR 信息。

定义如下：

```
typedef struct VFIORegion{
	struct VFIODevice *vbasedev;
	off_t fd_offset; /* offset of region within device fd */
	MemoryRegion *mem;/* slow,read/write access */
	size_t size;
	uint32_t flags; /* VFIO region flags (rd/wr/mmap) */
	uint32_t nr_mmaps;
	VFIOMmap *mmaps;
	uint8_t nr; /* cache the region number for debug */
} VFIORegion;
```

fd_offset 表示该 BAR 在直通设备 fd 文件中的偏移，mem 指向该 BAR 对应 MemoryRegion，size 和 flags 是该 BAR 的基本信息。nr_mmaps 和 mmaps 表示映射信息。

```
vfio_region_setup
	vfio_get_region_info
	memory_region_init_io
		memory_region_init
	...
	vfio_setup_region_sparse_mmaps
```

a. vfio_get_region_info 通过 ioctl VFIO_DEVICE_GET_REGION_INFO 获取 vfio_region_info。将内容复制到成员 VFIORegion中。 
b. 为 BAR 创建一个 MemoryRegion 结构体，使用 BAR 信息初始化，最后分配 VFIORegion 的 mmaps 空间及初始化相关成员(只记录BAR的基本信息)。

随后调用 vfio_bars_prepare，vfio_bars_register 对每一个 BAR 进行初始化。

```
vfio_bar_prepare
/*Determine what type of BAR this is for registration */
ret = pread(vdev->vbasedev.fd, &pci_bar, sizeof(pci_bar),
			vdev->config_offset + PCI_BASE_ADDRESS_0 + (4*nr));

```

调用pread获取直通设备BAR的信息，存放在pci_bar：该BAR类型（I/O，MMIO），是否为64位MMIO等。

```
vfio_bar_register
	vfio_region_mmap
```

vfio_region_mmap 根据 hpa，获取设备的 hva，将直通设备的物理 BAR 地址空间映射到 QEMU 中。上述已经获取 VFIORegion mmaps的 offset 和 size 参数。在 vfio_region_mmap 函数，直接调用 mmap 系统调用，将直通设备的 BAR 映射到 QEMU 地址空间。

小结：mmap 调用到内核，将直通设备对应的 MMIO 空间映射到内核中，然后再通过内核映射到用户空间。接着使用返回的虚拟地址初始化，VFIORegion mmaps 的 mem 成员，产生一个使用实际内存作为后端的 MemoryRegion，随后将 MemoryRegion 加入 VFIORegion 的 mem 成员 MemoryRegion 中。

## DMA 重定向

DMA 重定向是设置设备端的物理内存到 QEMU 进程虚拟地址间的映射。简单来说，就是建立虚拟机内存和设备内存的 EPT 页表，dma 可以直接将数据传输到虚拟机内存，虚拟机也可以直接通过读特定内存，读取到设备信息。该功能由 vfio_listener_region_add 函数实现。在创建 container 时，会向 memory 注册 listener。上述已经分析创建 container 的过程， 

```
vfio_connect_container
	＃ vfio_memory_listener在每次container增加memory时，就会回调
	container->listener = vfio_memory_listener;
	memory_listener_register(&container->listener, container->space->as);

static const MemoryListener vfio_memory_listener={
	.region_add = vfio_listener_region_add,
	.region_del = vfio_listener_region_del,
};
```

当调用 region_add 添加 region 时，实际调用的是 vfio_listener_region_add。vfio_listener_region_add 完成地址映射

```
vfio_listener_region_add
	iova = TARGET_PAGE_ALIGN(section->offset_within_address_space);
	vaddr = memory_region_get_ram_ptr(section->mr) + 
					section->offset_within_region +
    				(iova - section->offset_within_address_space);
	vfio_dma_map
		ioctl(container->fd, VFIO_IOMMU_MAP_DMA, &map)
```

每次内存拓扑逻辑改变，都会调用注册所有的 MemoryListener。如果添加 MemoryRegion，这里会调用 vfio_listener_region_add，判断是否需要建立映射。只有虚拟机实际的物理内存，也就是对应 QEMU 中配有实际虚拟地址空间的 MemoryRegion，才进行 DMA Remapping。 
接着算出 iova，即 GPA，就是 MemoryRegion 在 AddressSpaace 中的起始位置，虚拟机物理内存的地址。算出 vaddr，即 HVA，就是QEMU 分配的虚拟机物理内存的虚拟地址，调用 vfio_dma_map ioctl 完成 iova 到 vaddr 的映射。

## 代码流程总结



















































































