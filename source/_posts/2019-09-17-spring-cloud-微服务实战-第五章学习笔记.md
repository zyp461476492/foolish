---
title: spring-cloud-微服务实战-第五章学习笔记
date: 2019-09-17 16:41:29
tags:
    - 学习笔记
    - spring 
    - spring cloud
    - spring boot
categories: 学习笔记
---

> spring-cloud 学习笔记，第四章，介绍了 Spring-cloud-hystrix 相关内容 ，记录笔记，供个人回顾使用。

<!-- more -->

## 背景

在微服务架构中，我们将系统拆分成了很多个单元，各个单元之间通过服务注册和服务订阅的方式互相依赖，由于都处在不同的进程中，依赖远程调用的方式，很有可能因为网络或者自身服务问题，出现调用故障和延迟，而延迟也会导致调用方的服务积压，最后因为积压从而使自身的服务也瘫痪。

所以，hystrix起到了一个熔断器的作用，类似于电路中的断路器，当发生故障时，及时切断故障的电路（服务），从而避免整个服务发生瘫痪。

参考官方给出的 hystrix 工作流程，我们进行逐步分析。

## 创建 HystrixCommand 或 HystrixObservableCommand 对象

构建 HystrixCommand 或 HystrixObservableCommand 对象，用来表示对依赖服务的操作请求。通过命名我们可以看出，它们采用了命令模式来实现对服务调用操作的封装。这两个 command 对象分别针对不同的应用场景。

- HystrixCommand 用在依赖服务返回单个操作结果的时候。
- HystrixObservableCommand 用在依赖服务返回多个操作结果的时候。

### 命令模式

> 命令模式是一种行为型设计模式。在命令模式中，所有的请求都会被包装成为一个对象。

命令模式中，一般有如下几个角色：

- command：命令的抽象接口，其中包含 execute 方法。
- concreteCommand：具体的命令实现类。每一个请求，都会映射一个具体的命令实现类。对于每个具体命令实现类，都会实现 execute 方法，同时依赖于 receiver，也就是接受者对象，实际具体的命令实现类是将对应的操作交给接受者对象来实现。
- receiver：请求的接受者，也是请求的最终执行者，被命令实现类所依赖。
- invoker：请求调用者，调用者会调用所有传入命令对象的 execute 方法，从而运行命令，但它不会和最终执行者 receiver 耦合，两者通过命令实现类来关联。
- client：进行接受者和命令对象的创建，并建立两者之间的关系。

关键点在于调用者 invoker 和操作者 receiver 通过 command 命令接口实现了解耦。

## 2.命令执行

Hystrix 在执行时会根据创建的 Command 对象以及具体的情况来选择一个执行。其中 HystrixCommand 实现了下面两个执行方式

- execute(): 同步执行，从依赖的服务返回一个单一的结果对象，或者是在发生错误时抛出异常。
- queue()：异步执行，直接返回一个 Future 对象，其中包括了服务执行结束时要返回的单一结果对象。

```java
R value = command.execute();
Future<R> fValue = command.queue();
```

HystrixObservableCommand 实现了另外两种执行方式。

- observe()：返回 observeable 对象，它代表了操作的多个结果，它是一个 HotObservable 。
- toObservable()：同样会返回一个 Observable 对象，也代表了操作的多个结果，但它是一个 Cold Observable

```java
Observable<R> ohValue = command.observe();
Observable<R> ocValue = command.toObservable();
```

Hot Observable 它不论“事件源”是否有订阅者，都会在创建后对事件进行发布，所以对于 Hot Observable 的每一个订阅者，都可能是从 “事件源” 的中途开始的，可能只是看到了整个操作的局部过程。

Cold Observable 在没有订阅者的时候不会发布事件，而是进行等待。直到有订阅者之后才发布事件。所以对于 Cold Observable 的订阅者，它保证了从一开始看到整个操作的全部过程。

## 3.结果是否缓存

若当前命令的请求缓存功能是被启用的，并且该命令缓存命中，那么缓存的结果会立即以 Observable 对象的形式返回。

## 4.断路器是否打开

在命令结果没有缓存的时候，Hystrix 在执行命令前需要检测断路器是否为打开状态

- 如果断路器是打开的，那么 Hystrix 不会执行命令，而是转到 fallback 处理逻辑。
- 如果断路器是关闭的，那么 Hystrix 跳到第五步（线程池/请求队列/信号量是否占满），检查是否有可用资源来执行命令。

## 5.线程池/请求队列/信号量是否占满

这一步检查命令相关的线程池/请求队列或信号量是否占满，如果占满，Hystrix 不会执行命令，而是转到 fallback 处理逻辑。

这里 Hystrix 所判断的线程池并非容器的线程池，而是每个依赖服务专有线程池。

## 6.HystrixObservableCommand.construct 或 HystrixCommand.run

Hystrix 会根据我们编写的方法来决定采取什么样的方式请求依赖服务。

- HystrixCommand.run(): 返回单一的结果，或者抛出异常。
- HystrixObservableCommand.construct 返回一个 Observable 对象来发射多个结果，或通过 onError 发送错误通知。

## 7.计算断路器的健康度

## 8.fallback 处理

当命令执行失败时，Hystrix 会进入 fallback 尝试回退处理，通常也叫做 “服务降级”。

通常引起服务降级的情况有一下几种：

- 第四步，当前命令处于 “熔断/断路” ，断路器是打开状态。
- 第五步，当前命令的线程池，请求队列，信号量全满。
- 第六步，抛出异常时。
