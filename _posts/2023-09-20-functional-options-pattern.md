---
layout:     post
title:      "设计模式之函数式选项模式"
subtitle:   "透过源码分析函数式选项模式"
date:       2023-09-20
author:     "caitui"
categories:
    - Design Pattern


---

## ## 函数式选项模式

### 函数式选项模式定义

​	Go 语言没有构造函数，一般通过定义 New 函数来充当构造函数。然而，如果结构有较多字段，那应该如何初始化这些字段，如何为某些配置项指定默认值？这时候有一种很好的设计模式可以使用--函数式选项模式。	

​	函数式选项（Functional options） 是一种创造性的设计模式，允许你使用接受零个或多个函数作为参数的可变构造函数构建复杂结构。在该模式中，你可以声明一个不透明的 `Option` 类型，该类型在某些内部结构中记录信息。你接受这些可变数量的选项，并根据内部结构上的选项记录的完整信息进行操作。将此模式用于构造函数和其他公共 API 中的可选参数，你预计这些参数需要扩展，尤其是在这些函数上已经有三个或更多参数的情况下。

### K8S调度器函数式选项模式应用

​	K8S的调度器scheduler是一个复杂的结构体，在初始化该结构体时，需要大量的可变参数。在初始化 `Scheduler` 对象时，使用了选项函数进行传参和初始化配置。

![img](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/d5219dfece9f908adb085c69c77bfbb3.png)

​	在函数式选项模式中，我们首先定义一个 Option 函数类型，该函数接收一个参数：schedulerOptions。

```go
// Option configures a Scheduler
type Option func(*schedulerOptions)
```

​	Scheduler的构造函数包含了一个 Option 类型的不定参数，Option是函数类型，遍历传入的opts进行参数赋值。

```go
// New returns a Scheduler
func New(ctx context.Context,
	client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	opts ...Option) (*Scheduler, error) {

	stopEverything := ctx.Done()
  
  // 函数选项式模块初始化调度器结构体
	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}
  
  ...
  
	sched := &Scheduler{
		Cache:                    schedulerCache,
		client:                   client,
		nodeInfoSnapshot:         snapshot,
		percentageOfNodesToScore: options.percentageOfNodesToScore,
		Extenders:                extenders,
		StopEverything:           stopEverything,
		SchedulingQueue:          podQueue,
		Profiles:                 profiles,
	}

	return sched, nil
}
```

​	那选项如何起作用？需要定义一系列相关返回 Option 的函数：

```go
// WithComponentConfigVersion sets the component config version to the
// KubeSchedulerConfiguration version used. The string should be the full
// scheme group/version of the external type we converted from (for example
// "kubescheduler.config.k8s.io/v1")
func WithComponentConfigVersion(componentConfigVersion string) Option {
	return func(o *frameworkOptions) {
		o.componentConfigVersion = componentConfigVersion
	}
}

// WithClientSet sets clientSet for the scheduling frameworkImpl.
func WithClientSet(clientSet clientset.Interface) Option {
	return func(o *frameworkOptions) {
		o.clientSet = clientSet
	}
}

// WithKubeConfig sets kubeConfig for the scheduling frameworkImpl.
func WithKubeConfig(kubeConfig *restclient.Config) Option {
	return func(o *frameworkOptions) {
		o.kubeConfig = kubeConfig
	}
}

// WithEventRecorder sets clientSet for the scheduling frameworkImpl.
func WithEventRecorder(recorder events.EventRecorder) Option {
	return func(o *frameworkOptions) {
		o.eventRecorder = recorder
	}
}

// WithInformerFactory sets informer factory for the scheduling frameworkImpl.
func WithInformerFactory(informerFactory informers.SharedInformerFactory) Option {
	return func(o *frameworkOptions) {
		o.informerFactory = informerFactory
	}
}

...
```

​	我们在使用时，只需要传入这些 Option 的函数即可，未来增加选项，只需要增加对应的 WithXXX 函数即可。

```go
	// Create the scheduler.
	sched, err := scheduler.New(ctx,
		cc.Client,
		cc.InformerFactory,
		cc.DynInformerFactory,
		recorderFactory,
		scheduler.WithComponentConfigVersion(cc.ComponentConfig.TypeMeta.APIVersion),
		scheduler.WithKubeConfig(cc.KubeConfig),
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithPodMaxInUnschedulablePodsDuration(cc.PodMaxInUnschedulablePodsDuration),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
		scheduler.WithParallelism(cc.ComponentConfig.Parallelism),
		scheduler.WithBuildFrameworkCapturer(func(profile kubeschedulerconfig.KubeSchedulerProfile) {
			// Profiles are processed during Framework instantiation to set default plugins and configurations. Capturing them for logging
			completedProfiles = append(completedProfiles, profile)
		}),
```

### 函数式选项模式使用场景

​	在服务配置场景，一般会有很多可选参数，比如针对Mysql、Redis等进行参数配置，使用函数式选项模式可以方便动态灵活的配置想要配置的参数。

### 函数式选项模式优缺点

#### 优点

- 任意顺序传递参数
- 支持默认值
- 向后兼容性
- 很容易维护和扩展