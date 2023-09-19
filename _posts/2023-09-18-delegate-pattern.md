---
layout:     post
title:      "设计模式之委派模式"
subtitle:   "透过源码分析委派模式"
date:       2023-09-18
author:     "caitui"
categories:
    - Design Pattern
---

## 委派模式

### 委派模式定义

委派模式是一种行为模式，基本的作用是负责任务的调度和分配。

### 主要角色和UML类图

#### 主要角色

Task （抽象任务）： 定义一个抽象接口， 它有多个实现类。

ConcreteTask （具体任务执行者、被委派者）： 具体任务角色，真正执行任务的角色就是被代理类。

Delegate （委派者）： 代理类，负责在各个具体角色实例之间做出决策，判断并调用具体实现类的方法。

#### UML类图

![delegate](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/delegate.png)

### K8S的apiserver如何实现委派模式

​	k8s的apiserver使用委派模式，通过DelegationTarget接口，把Aggretgator、APIServer、APIExtensions链式串联起来，对外提供服务。

![image-20230918191055617](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/image-20230918191055617.png)

​	构建委派调用链由函数CreateServerChain实现，该函数构造Aggretgator、API Server、APIExtensions这三个apiserver。当用户请求进来，先判断 `AggregatorServer` 能否处理，否则代理给 `kubeApiServer` ，如果 `kubeApiServer` 不能处代理给 `ApiExtensionServer` 处理，如果都不能处理则交给 notFoundHandler 处理。

![](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/20230918192307-20230918192337708.jpeg)

​	我们截取kubeApiServer的pathHandlers和prefixHandlers代码片段来说明委托模式，当精确匹配和前缀匹配都没命中时，就会委派notFoundHandler进行处理，而notFoundHandler属于apiextensionsServer。

![image-20230918192755588](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/image-20230918192755588.png)

​	再来看看apiextensionsServer的customresources_handler函数，当没命中 /apis/<group>/<version> 和 /apis/<group> 时，会委派给下一handler进行处理。

![企业微信截图_d45fbffd-43a5-41bc-bcec-2dd0ce3e9b49](https://raw.githubusercontent.com/caitui/caitui.github.io/main/blog-image/企业微信截图_d45fbffd-43a5-41bc-bcec-2dd0ce3e9b49.png)

### 应用场景

- 委派对象本身不知道如何处理一个任务或一个请求，把请求交给其他对象来处理。

- 可以实现程序的解耦

### 委派模式优缺点

#### 优点

​	通过任务委派能够将一个大型的任务细化，然后通过统一管理子任务的完成情况实现任务的跟进，能够加快任务执行的效率。

#### 缺点

​	委派模式需要根据任务的复杂程度进行不同的改变，在任务比较复杂的情况下可能需要进行多重委派，容易造成系统过于复杂。
