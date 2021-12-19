---
layout: post
title: kata container 模板启动源码分析
date: 2021-12-05
tags: jekyll
---

## 使用方法

前提条件：configuration.toml 将模板启动的配置项打开

```
enable_template = true
```

使用模板启动的方式有两种

1.使用 kata runtime 创建模板，使用容器命令启动

```
kata-runtime factory init
docker run -tid --runtime "io.containerd.kata.v2" --net=none busybox:latest sh
```

2.直接使用容器命令行启动。在这种情况，第一次启动因为制作模板的原因，启动会稍慢

```
＃ 第一次制作模板
docker run -tid --runtime "io.containerd.kata.v2" --net=none busybox:latest sh
＃ 第二次使用模板启动
docker run -tid --runtime "io.containerd.kata.v2"--net=none busybox:latest sh
```

## 启动源码解析

### kata runtime创建模板

kata runtime 提供了创建模板的命令行。当我们输入命令行开始，就自动使用模板启动方式。

所有的命令行都是从 runtime main 函数开始

```
cli/main.go
func main()
	createRuntime(ctx)
		createRuntimeApp(ctx, os.Args)
			app.Run(args)
				＃ 根据执行的命令名字，调用对应的函数
				c.Run(context)
				...
```

函数最终调用 factory 模板工厂的方法

```
cli/factory.go
var initFactoryCommand = cli.Command {
	＃ 保存factory config配置信息
	factoryConfig := vf.Config {
		...
	}
	
	＃ 如果configuration.toml配置了 template，就创建factory
	 if runtimeConfig.FactoryConfig.Template 
	 	vf.NewFactory(ctx, factoryconfig, false) 
	 		template.New(ctx, config.VMConfig, config.TemplatePath)
}
```

上述最终调用 template 方法 new() 创建一份模板

```
virtcontainers/factory/template/template.go
New
	＃ 用于检查模板目录下有memory和state文件，如果有，返回err，不需要创建模板
	t.checkTemplateVM()
	
	＃ 准备创建的文件名，权限
	t.prepareTemplateFiles()
	
	＃ 执行创建模板的函数
	t.createTemplateVM(ctx)
```

执行真正创建模板函数 createTemplateVM，用于设置全局变量标记位，启动虚拟机，暂停虚拟机，创建模板。

```
virtcontainers/factory/template/template.go
createTemplateVM
	＃ 配置环境
	config.HypervisorConfig.BootToBeTemplate = true
	config.HypervisorConfig.BootFromTemplate = false
	config.HypervisorConfig.MemoryPath = t.statePath + "/memory"
	config.HypervisorConfig.DevicesStatePath = t.statePath +"/state"
	
	＃ 创建一个虚拟机
	vc.NewWM(ctx,config)
		＃ 步骤一，设置hypervisor，并设置启动vm命令行
		newHypervisor(config.HypervisorType)
		hypervisor.createSandbox(ctx, id, NetworkNamespace{}, &config.HypervisorConfig)

		＃ 步骤二，设置agent
		agent := getNewAgentFunc(ctx)
		agent.configure(ctx, hypervisor, id, vmSharePath, config.Agentconfig) agent.setAgentURL()
		
		＃步骤三：启动虚拟机
		hypervisor,startSandbox(ctx, vmStartTimeout)
		
		# 暂停虚拟机
		vm.Pause(ctx)
		# 下发qp命令，保存模板
		vm.Save()
		＃ 返回前，暂停虚拟机
		defer vm.Stop(ctx)
```

当我们使用 docker 再次创建容器时，流程变为判断是否有factory，如果是，则直接饮复虚拟机状态。这样，节省了启动虚拟机花费的大量时间。流程如下

```
virtcontainers/api.go
CreateSandbox
	createSandboxFromConfig
		s.startM(ctx)
			if s.factory !=nil
				vm := s.factory.Get(ctx,WConfig(
					HypervisorType:  s.confie.HygerviserType,
					HypervisorConfig: s.cenfig.HypervisorConfig,
					AgentConfig:    s.confie.AgentConfia,
				))
				
				# 添加虚拟机文件和sandbox链接关系
				return vm.assignband(s)
```

上述使用 GetVM 函数，直接恢复虚细机，如退国超起机状态。

```
virtcontainers/factory/factory-go
GetVN
	＃ 检测并复位config信息
	f.checkConfig(config)
	
	＃ 获取虚拟机基础信息
	f.base.GetBase\M(ctx, config)
	
	＃ 恢复虚拟机
	vm.Resume(ctx)
	
	# 重新设置随机数值
	vm.ReseedRNG(ctx)
	vm.SyncTime(ctx)
	
	# 设置cpu和内存
	if baseConfig.NumVCPUs < hypervisorConfig.NumvCPUs 
		vm.AddcPus()
	if baseConfig.Memorysize < hypervisorConfig.MemarySize
		vm.Addvemory()
	
	# 上线cpu内存
	vw.onlineCPNemory
```