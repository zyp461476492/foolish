---
title: spring-cloud-netflix-文档阅读笔记
date: 2019-11-12 15:24:34
tags:
    - 学习笔记
    - spring 
    - spring cloud
    - netflix
categories: spring-cloud, netflix
---
> 官方文档阅读笔记，记录一些理解，和蹩脚翻译

<!-- more -->

## Circuit Breaker: Hystrix Clients （循环断路器）

### Circuit Breaker pattern

在软件系统中，不同进程间发起远程调用，或者通过不同机器发起调用是一件常见的事。和在内存之间互相调用最大的不同的是，远程调用可能会失败，或长时间无响应直到达到了设置的超时限制。如果你有很多请求是对一个无法响应的服务提供者，那么你可能会用尽你所有的重要资源，从而在多个系统之间引发灾难性的错误，这是多么的糟糕。

[Circuit Breaker pattern](circuit-breaker-pattern) 就是一种阻止上述灾难性错误的设计模式。

circuit breaker 的基本原理非常简单。你把要保护的方法或请求包含在一个可以监控调用失败的 circuit breaker 对象中。一旦故障到达了设置的临界值，circuit breaker 就会被触发（跳闸），所有的失败请求都会通过 circuit breaker 来返回一个所以，而不去对保护的对象进行请求。

通常，你也需要一些对 circuit breaker 跳闸的监控信息。

![circuit breaker](https://martinfowler.com/bliki/images/circuitBreaker/sketch.png)

[circuit-breaker-pattern]: https://martinfowler.com/bliki/CircuitBreaker.html

## Hystrix Clients

Netflix 创建了一个叫做 Hystrix 的库，它实现了 circuit breaker pattern。在微服务架构中，有多层的服务调用是一个很普遍的现象。

一个无法提供服务的底层服务可以引发级联的错误，从而导致上层的使用者无法正常使用。当我们请求一个服务时，在一个 **metrics.rollingStats.timeInMilliseconds** (默认为10s) 定义的滑动窗口内如果连续失败超过了 **circuitBreaker.requestVolumeThreshold** （默认为 20 次请求）并且失败率超过了 **circuitBreaker.errorThresholdPercentage** （默认 > 50%），circuit 断路将会被启动，不会继续进行请求。在这种错误和断路打开的情况下，一个 fallback 会被提供给开发者。

通过断路来阻止级联错误，并给了错误的服务时间来进行恢复。Fallback 可以是另外一个在 hystrix 保护下的请求，静态数据或合法的空值。回退可以是链式的，因此第一个回退会进行一些其他的业务调用，而这些调用又会回退到静态数据。

## 如何在项目中导入 Hystrix

首先，加入 hystrix 的依赖

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
    </dependencies>
```

通过 @EnableCircuitBreaker 注解来启动一个最小配置的 Hystrix Circuit breaker。通过 @HystrixCommand 和继承 HystrixCommand 等来实现 fallback 等内容。

## 客户端侧的负载均衡器-Ribbon

