---
layout:     post
title:      "设计模式之工厂模式"
subtitle:   "透过源码分析工厂模式"
date:       2023-09-19
author:     "caitui"
categories:
    - Design Pattern

---

## 工厂模式

### 工厂模式定义

工厂方法模式是一种创建型设计模式， 其在父类中提供一个创建对象的方法， 允许子类决定实例化对象的类型。

### 主要角色和UML类图

#### 主要角色

抽象工厂模式主要包含4个角色：

- 抽象工厂（Factory）：是工厂方法模式的核心，与应用程序无关。任何在模式中创建的对象的工厂类必须实现这个接口。

- 具体工厂（Concrete Factory）：是实现抽象工厂接口的具体工厂类，包含与应用程序密切相关的逻辑，并且被应用程序调用以创建产品对象。

- 抽象产品（Product）：是工厂方法模式所创建的对象的超类型，也就是产品对象的共同父类或共同拥有的接口。

- 具体产品（Concrete Product）：这个角色实现了抽象产品角色所定义的接口。某具体产品有专门的具体工厂创建，它们之间往往一一对应。

#### UML类图

![Image](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/640-20230920105700353.jpeg)

### K8S调度器如何使用工厂模式

​	k8s的Framework是kubernetes扩展的第二种实现，主要是围绕着预选和优选阶段进行扩展，提供了更多的扩展点，其中每个扩展点都是一类插件，我们可以根据我们的需要在对应的阶段来进行扩展插件的编写，实现调度增强。主要的扩展点如下图所示：

![暂无图片](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/modb_20230417_e380aea8-dcc3-11ed-b19d-38f9d3cd240d-20230920110141362-20230920110144861.jpeg)

​	插件思想可以通过工厂模式来实现，一个插件暴露出来的是一个工厂函数，由调用者或者插件架构来将提供配置信息传入，生成插件实例。k8s的调度器代码是通过插件工厂来存储所有注册的插件工厂，然后通过插件工厂构建具体的插件。

```go
// 插件工厂注册表，用于存储所有注册的插件工厂
type Registry map[string]PluginFactory

// 向注册表注册一个插件工厂
func (r Registry) Register(name string, factory PluginFactory) error {
	if _, ok := r[name]; ok {
		return fmt.Errorf("a plugin named %v already exists", name)
	}
	r[name] = factory
	return nil
}
```

​	定义PluginFactory工厂方法类型，工厂函数即传入对应的参数，构建一个Plugin，其中framework.Handle主要是用于获取快照和集群的其他数据。

```go
type PluginFactory = func(configuration runtime.Object, f framework.Handle) (framework.Plugin, error)

```

​	函数NewInTreeRegistry()实现了InTree插件批量注册，根据插件名称初始化各插件函数保存到Registry注册表中。

```go
func NewInTreeRegistry() runtime.Registry {
	registry := runtime.Registry{
		dynamicresources.Name:                runtime.FactoryAdapter(fts, dynamicresources.New),
		selectorspread.Name:                  selectorspread.New,
		imagelocality.Name:                   imagelocality.New,
		tainttoleration.Name:                 tainttoleration.New,
		nodename.Name:                        nodename.New,
		nodeports.Name:                       nodeports.New,
		nodeaffinity.Name:                    nodeaffinity.New,
		podtopologyspread.Name:               runtime.FactoryAdapter(fts, podtopologyspread.New),
		nodeunschedulable.Name:               nodeunschedulable.New,
		noderesources.Name:                   runtime.FactoryAdapter(fts, noderesources.NewFit),
		noderesources.BalancedAllocationName: runtime.FactoryAdapter(fts, noderesources.NewBalancedAllocation),
		volumebinding.Name:                   runtime.FactoryAdapter(fts, volumebinding.New),
		volumerestrictions.Name:              runtime.FactoryAdapter(fts, volumerestrictions.New),
		volumezone.Name:                      volumezone.New,
		nodevolumelimits.CSIName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewCSI),
		nodevolumelimits.EBSName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewEBS),
		nodevolumelimits.GCEPDName:           runtime.FactoryAdapter(fts, nodevolumelimits.NewGCEPD),
		nodevolumelimits.AzureDiskName:       runtime.FactoryAdapter(fts, nodevolumelimits.NewAzureDisk),
		nodevolumelimits.CinderName:          runtime.FactoryAdapter(fts, nodevolumelimits.NewCinder),
		interpodaffinity.Name:                interpodaffinity.New,
		queuesort.Name:                       queuesort.New,
		defaultbinder.Name:                   defaultbinder.New,
		defaultpreemption.Name:               runtime.FactoryAdapter(fts, defaultpreemption.New),
		schedulinggates.Name:                 runtime.FactoryAdapter(fts, schedulinggates.New),
	}

	return registry
}
```

​	拿NodeAffinity插件为例，该插件的初始化函数New的类型为PluginFactory，在函数NewInTreeRegistry()中会调用该初始化函数，将构建NodeAffinity插件的函数注册到Registry注册表中。

```go
func New(plArgs runtime.Object, h framework.Handle) (framework.Plugin, error) {
	args, err := getArgs(plArgs)
	if err != nil {
		return nil, err
	}
	pl := &NodeAffinity{
		handle: h,
	}
	
  ... 
  
	return pl, nil
}
```

​	以上就是工厂函数注册的全流程，那么Plugin又是何时初始化的呢？在profile（用来保存不同调度器的 framework 框架, framework 则用来存放 plugin）创建时，会初始化NewFramework，该函数调用生成的插件工厂注册表，来通过每个插件的Factory构建Plugin插件实例, 保存到pluginsMap中。

```go
// NewFramework initializes plugins given the configuration and the registry.
func NewFramework(ctx context.Context, r Registry, profile *config.KubeSchedulerProfile, opts ...Option) (framework.Framework, error) {
	options := defaultFrameworkOptions(ctx.Done())
	for _, opt := range opts {
		opt(&options)
	}

	f := &frameworkImpl{
		registry:             r,
		snapshotSharedLister: options.snapshotSharedLister,
		scorePluginWeight:    make(map[string]int),
		waitingPods:          newWaitingPodsMap(),
		clientSet:            options.clientSet,
		kubeConfig:           options.kubeConfig,
		eventRecorder:        options.eventRecorder,
		informerFactory:      options.informerFactory,
		metricsRecorder:      options.metricsRecorder,
		extenders:            options.extenders,
		PodNominator:         options.podNominator,
		parallelizer:         options.parallelizer,
	}

  ...

	pluginsMap := make(map[string]framework.Plugin)
	for name, factory := range r {
    
		args := pluginConfig[name]
		if args != nil {
			outputProfile.PluginConfig = append(outputProfile.PluginConfig, config.PluginConfig{
				Name: name,
				Args: args,
			})
		}
    // 调用PluginFactory函数对插件进行初始化，构建plugin
		p, err := factory(args, f)
		if err != nil {
			return nil, fmt.Errorf("initializing plugin %q: %w", name, err)
		}
		pluginsMap[name] = p

		f.fillEnqueueExtensions(p)
	}

  ...
  
	return f, nil
}
```



### 工厂模式使用场景

工厂方法模式主要适用于以下应用场景。

- 创建对象需要大量重复的代码。

- 客户端不依赖产品类实例如何被创建、实现等细节。

- 一个类通过其子类来指定创建哪个对象。

### 工厂模式优缺点

#### 优点

-  避免创建者和具体产品之间的紧密耦合。
-  *单一职责原则*。 将产品创建代码放在程序的单一位置， 从而使得代码更容易维护。
-  *开闭原则*。 无需更改现有客户端代码， 就可以在程序中引入新的产品类型。

#### 缺点

​	应用工厂方法模式需要引入许多新的子类， 代码可能会因此变得更复杂。 最好的情况是将该模式引入创建者类的现有层次结构中。



