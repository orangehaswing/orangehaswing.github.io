---
layout: post
title: kata container 设备直通源码分析
date: 2021-12-05
tags: jekyll
---

## 使用方式

docker 启动命令行指定设备是 vfio 设备：--devoce /dev/vfio/40

```
docker run -it --runtime "io.containerd.kata.v2" --rm --device /dev/vfio/40 --net=none busybox:latest sh
```

## 创建过程

在创建完 sandbox 后，会创建 container。在创建 container 过程中，准确的说是在调用 create 创建容器前，会先把容器需要的镜像，设备等资源热插到虚拟机内，提供给 kata agent 启动容器使用。

```go
src\runtime\virtcontainers\container.go
create
	// 如果支持devmapper block，就是使用热插方式，增加一个block设备
	if c.checkBlockDeviceSupport(ctx)
		c.hotplugDrive(ctx)
		
	// 如果是qemu是q35主板，会判断vfio需要large bar
	if machineType == QemuQ35
		if isLargeBarSpace 
			delayAttachedDevs = append(delayAttachedDevs, device)
		else 
			normalAttachedDevs = append(normalAttachedDevs, device)
	// 其余虚拟机类型，直接增加需要添加的设备
	else
		normalAttachedDevs = c.devices
	
	// 如果有设备，则把设备添加到虚拟机
	if len(normalAttachedDevs) > 0 
		c.attachDevices(ctx, normalAttachedDevs)
```

上述的 normalAttachDevs 用于保存设备的数据结构，具体内容如下

```go
type ContainerDevice struct {
	ID string
	ContainerPath string
	FileMode os.FileMode
	UID uint32
	GID uint32
}
```

添加设备过程，直接调用 device manager 方法，对遍历的所有设备做热插操作

```go
virtcontainers/container.go
attachDevices
	for _, dev := range devices
		c.sandbox.devManager.AttachDevice(ctx, dev.ID, c.sandbox)
		// 根据id取设备
		d, ok := dm.devices[id]
		// 添加设备
		d.Attach(ctx, dr)
```

根据设备类型不同，调用不同的设备 Attach 函数，当使用 vfio 设备热插拔时，会调用 vfio.go 文件下对应的函数

```go
virtcontainers/device/drivers/vfio.go
Attach
	// 获取group id，iommu 设备路径，设备文件
	vfioGroup := filepath.Base(device.DeviceInfo.HostPath)
	iommuDevicesPath := filepath.Join(config.SysIOMMUPath, vfioGroup,"devices")
	deviceFiles, err := ioutil.ReadDir(iommuDevicesPath)
	
	// 遍历设备，获取vfio设备bdf号信息
	for i, deviceFile := range deviceFiles
		deviceBDF, devicesysfsDev, vfioDeviceType = getVFIODetails()
		// 保存vfio设备信息
		vfio := &config.VFIODev {
			...
		}
		device.VfioDevs = append(device.VfioDevs, vfio)
		
		// 根据设备信息，选择热插或令加载
		coldPlug := device.DeviceInfo.ColdPlug
		
		// 这里走热插流程
		if coldPlug
			devReceiver.AppendDevice
		else
			devReceiver.HotplugAddDevice
```

当进行热插时，会继续调用 sandbox 接口 HotplugAddDevice，接下来的代码流程就比较清晰

```go
virtcontainers/sandbox.go
HotplugAddDevice
	// 根据设备类型，选择vfio设备
	switch devlype
		case config.DeviceVFIO
		// 获取设备信息
		vfioDevices: device.GetDeviceInfo().([]*config.VFIODev)
		for _, dev =: range vfioDevices
			// 遍历设备，调用对应的hypervisor热插设备
			s.hypervisor.hotplugAddDevice(ctx, dev, vfioDev)
```

例如在 qemu 里，调用热插 vfio 设备如下

```go
virtcontainers/qemu.go
hotplugAddDevice
	// 增加addDevice参数
	q.hotplugDevice(ctx, devInfo,devType, addDevice)
		// 选择热插vfio设备分支
		case vfioDev:
			q.hotplugVFIODevice(ctx, device, op)
				// 如果直接把设备热插到root bus
				if q.state.HotplugVFIOOnRootBus
					// 如果是q35，且为pcie设备，建议热插到root port
				
				// 如果是普通vfio设备（相对的是gpu设备）
				switch device.Type
					case config.VFIODeviceNormalType
						//调用qmp合令热插
						q.qmpMonitorCh.qmp.ExecuteVFIODeviceAdd
				
				//如果不把设备热插到root bus，不用判断虚拟机机型，直接调用qmp命令
				addr,bridge :=  q.arch.addDeviceTo8ridge(ctx, devID, types.PCI)
                q.qmpMonitorch.qmp.ExecutePCIVFIODeviceAdd
```

## 获取设备信息

上述设备是如何获取到命令行参数是 vfio device，并把对应的设备添加到 devManager 的 device？在创建 sandbox 过程中，首先会设置 sandbox 结构体参数，然后使用这个 sandbox，提前创建网卡，启动虚拟机。然后创建容器，保存容器

```go
virtcontainers/api.go
CreateSandbox
	createSandboxFromConfig
		// 创建sandbox结构体
		createSandbox()
		// 创建网卡，实际是为虚拟机启动添加网络命令行
		s.createNetwork()
		// 创建虚拟机，并加虚拟机内的kata agent启动，更新虚拟机内的网卡
		s.startM(ctx)
		
		// 创建容器过程中，会保存设备，并使用设备信息进行热插
		s.createContainers(ctx)
```

下面具体分析在创建过程中是如何保存设备，并对设备热插。其中热插部分将和上述第二节衔接上。

```go
virtcontainers/sandbox.go
createContainers
	for i := range s.config.Containers
		newContainer(ctx, s, &s.config.Containers[i])
		c.create(ctx)
```

c.create(ctx) 函数会执行上述的创建过程，进而热插 block，vfio，然后调用 kata agent，在虚拟机运行容器。下面具体分析 newContainer 函数过程

```go
virtcontainers/container.go
newContainer
	// 保存container数据结构内容
	c := &Container{
		...
	}
	//创建设备信息
	c.createDevices(contConfig)
	
createDevices
	// 遍历config device info信息
	for _,info := range contConfig.DeviceInfos
		// 添加到devmagger
		c.sandbox.devManager.NewDevice(info)
		// virtcontainers/device/manager/manager.go
		// 保存dev信息
		dev := dm.createDevice(devInfo)
		// 第二章节通过ID号可以直接查出device信息，在这里被赋值
		dm.devices[dev.DeviceID()] = dev
```

在保存 device 信息的过程中，会根据设备类型不同，进行校验。

```go
virtcontainers/device/manager/manager.go
createDevice
	// 如果不是memory设备，就获取主机路径
	if !devInfo.Pmem
		path := config.GetHostPathFunc()
		
	// 在只考虑vfio分支的情况下，isVFIO判断满足vfio路径格式，然后创建vfio dev设备信息
	if isVFIO(devInfo.HostPath)
		return drivers.NewVFIODevice(&devInfo)
```

判断和创建具体过程如下：

```go
isVFIO
	// 如果设备是/dev/vfio/vfio，则忽略
	if strings.HasPrefix(hostPath, filepath.Join(vfioPath, "vfio"))
	// 如果是/dev/vfio/number，则是vfio设备
	if strings.HasPrefix(hostPath, vfioPath) && len(hostPath)>len(vfioPath)
	
NewVFI0Device
	return VFIODevice{
		ID:   devInfo.ID,
		DeviceInfo: devInfo,
	}
```

