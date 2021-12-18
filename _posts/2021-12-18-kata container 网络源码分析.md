---
layout: post
title: kata container 网络源码分析
date: 2021-12-05
tags: jekyll
---

## 使用方法

1. 设置 kata 配置文件(/usr/share/defaults/kata-containers/configuration.toml)

   ```
   internetworking_model = "tcfilter"
   disable_new_netns = false
   ```

2. 使用 containerd，并配置 config 使用 cni 插件(/etc/containerd/config.toml)

   ```
   [plugins.cri.cni]
   bin_dir = "/opt/cni/bin"
   conf_dir = "/etc/cni/net.d"
   ```

3. 编译 cni 插件二进制和配置文件

   ```
   /opt/cni/bin/
   ...
   
   /etc/cni/net.d/10-mynet.conf
   {
   	"cniVersion": "0.2.0",
   	"name": "mynet",
   	"type": "bridge",
   	"bridge": "cni0",
   	"isGateway": true,
   	"ipMasq": true,
   	"ipam":{
   		"type": "host-local",
   		"subnet": "172.19.0.0/24",
   		"routes":[
   			{ "dst": "0.0.0.0/0" }
   		]
   	}
   }
   ```

4. 使用 crictl 创建 sandbox

   ```
   crictl runp pod-config.json
   {
   	"metadata":{
   		"name": "nginx-sandbox1",
   		"namespace": "default",
   		"attempt":1,
   		"uid": "hdishd83djaidduwk28bcsb"
   	},
   	"log_directory": "/tmp",
   	"linux":{
   	}
   }
   ```

## 创建网络

当 containerd 使用 cni 时，会创建 cni0 网桥，在网桥上创建 veth pair。一个 veth 在主机 host，一个在虚拟机所在的 namespace
步骤：

1. containerd 创建 namespace

2. containerd 启动 vm，没有网卡

3. cni 组网

   a. 创建 bridge
   b. 创建 veth pair

4. kata-runtime kata-network add-iface(由cni发起)

   a. 创建 tap0

   b. 连通 tap 和 veth

   c. 热插网卡到 vm

![kata-containers-network](https://gitee.com/orangehaswing/blog_images/raw/master/images/kata-containers-network.png)

在kata 2.x 新版本，没有 kata-network 组件，所以在 kata 源码中，使用冷启动网卡的方式，创建设备 kata runtime 创建过程如下：

```
virtcontainers/api.go
CreateSandbox
	createSandboxFromConfig(ctx, sandboxConfig, factory)
		err = s.createNetwork(ctx)
```

在创建 sandbox 过程中，会判断是否需要创建网络。如果需要，则在 namespace 做相应的操作。

```\
virtcontainers/sandbox.go
createNetwork
	// 如果在配置空间设置disable network namespace，直接返回，不创建网络
	// 即上述 configuration.toml 设置disable＿new＿netns ＝true
	if s.config.Networkconfig.DisableNewNetNs || s.config.NetworkConfig.NetNSPath == ""
		 return nil
```

接着保存 network NS，添加虚拟机网卡设备

```
s.networkNS = NetworkNamespace(
	NetNsPath:	s.config.NetworkConfig.NetNSPath,
	NetNsCreated:	s.config.NetworkConfig.NetNsCreated,
)

// 这里 factory 设置false
s.network.Add(ctx,&s.config.NetworkConfig, s,false)
// 保存创建的设备类型，为veth设备
s.networkNS.Endpoints = endpoints
// 如果使能了netmon组件，就启动netmon监控
if s.config.NetworkConfig.NetmonConfig.Enable
	s.startNetworkMonitor()
```

深入分析 network 添加设备的过程

```
s.network.Add
	// virtcontainers/network.go
	createEndpointsFromScan(config.NetNSPath, config)
	
	// Add入参false，所以不是热插方式
	doNetNS {
		if hotplug
			endpoint.HotAttach
		else
			endpoint.Attach
	}
	
// virtcontainers/veth_endpoint.go
Attach
	// virtcontainers/network.go
	xConnectVMNetwork(ctx,endpoint,h)
		// configuration.toml配置tcfileter模式
		case NetxConnectTCFilterModel:
			return setupTCFiltering(endpoint,queues,disableVhostNet)
				// 创建tap设备
				createLink(netHandle, netPair,TAPIface,Name, &netlink,Tuntap{}, queues)
	
	// 根据hypervisor不同，调用不同设备虚拟化组件
	h.addDevíce(ctx, endpoint, netDev)
		// 匹配网络设备，为qemu添加network命令行
		case Endpoint:
			s.appendNetDev
```

其中创建 namespace 中 tap 设备使用的函数是 createEndpointsFromScan()

```
virtcontainers/network.go
createEndpointsFromScan
	// 获取namespace路径，并获取handle
	netns.GetFromPath(networkNSPath)
	netlink.NewHandleAt(netnsHandle)
	
	// 设置namspace网络路由，地址，iface等属性，并创建tap设备
	networkInfoFromLínk(netlinkHandle, link)
	
	// 创建
	createEndpoint(netInfo, idx, config.InterworkingModel, link)
		// 创建veth pair
		else if netInfo.Iface.Type == "veth"
			createVethNetworkEndpoint(idx,netInfo.Iface.Name,model)
```

上述过程完整的说明了如何在 namespace 创建 tap 设备，并把 net 网卡设备添加到 qemu 启动命令行

## 热插网卡

当使用 factory 模板启动时，需要用到热插网卡，kata container 源码注释内容如下

```
virtcontainers/sandbox.go
startVM
	// In case of vm factory, network interfaces are hotplugged
	// after vm is started.
```

从上可知，当第一次使用模板启动，第二次开始，使用网卡热插方式。

```
virtcontainers/sandbox.go
startVM
	// 如果factory 不为空，则热插
	if s.factory != nil
		// 注意这里参数是true
		s.network.Add(ctx, &s.config.NetworkConfig, s.true)
```

入参为 true，则会在 Add 函数走 hotplug 流程。详细代码如下：

```
virtcontainers/network.go
Add
	createEndpointsFromScan(config.NetNSPath, config)
	doNetNS {
		if hotplug
			endpoint.HotAttach(ctx, s.hypervisor)
	}
```

veth pair 的 HotAttach() 函数代码主要流程和 Attach 类似。不同的是，最终调用 hypervisor 层的 hotplugAddDevice()，即热插网卡。

```
virtcontainers/veth_endpoint.go
HotAttach
	// tcfilter模式，创建tap设备
	xConnectvNetwork(ctx,endpoint, h)
	// 调用热插函数
	h.hotplugAddDevice
```

在 qemu 执行网卡热插操作

```
virtcontainers/qemu.go
hotplugAddDevice
	q.hotplugNet(ctx, devinfo.(Endpoint))
```

热插时序图：

![kata-containers-network-hotplug](https://gitee.com/orangehaswing/blog_images/raw/master/images/kata-containers-network-hotplug.png)

上述使用 kata-network 接口函数 add-interface 从命令行完成网卡热插。

## 更新虚拟机网卡信息

无论是启动时加载网卡，或热插网卡，当创建虚殿机后，都需要更新虚拟机内网卡信息。这个步骤在 startVM 函数完成。

```
virtcontainers/sandbox.go
startVM
	s.agent.startSandbox(ctx,s)
		// 如果使用vsock方式，返回vsock通信路径
		k.setAgentURL()
		
		// 获取主机dns信息，用于发送给虚拟机，当虚拟机启动后，可以正常使用该dns 连接外部网络
		k.getDNS(sandbox)
		
		// 检查grpc server服务正常
		k.check(ctx)
		
		// 获取设置的namespace网络信息，例如mac，ip地址，设备名，pci信息
		generateVCNetworkStructures()
		
		// 更新虚拟机网卡信息
		k.updateInterfacesHwAddrByName()
		k.updateInterfaces
		k.updateRoutes
		k.addARPNeighbors
		
		// 调用agent，创建sandbox
		req := &grpc.CreateSandboxRequest{
			...
		}
		k.sendReq(ctx, req)
```

























































































