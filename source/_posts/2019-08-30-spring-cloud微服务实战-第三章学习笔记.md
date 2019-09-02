---
title: spring-cloud微服务实战-第三章学习笔记
date: 2019-08-30 15:14:47
tags:
    - 学习笔记
    - spring 
    - spring cloud
    - spring boot
categories: 学习笔记
---

> spring-cloud 学习笔记，第三章，介绍了 eureka 相关内容 ，记录笔记，供个人回顾使用。

<!-- more -->

## 服务治理

spring-cloud-eureka,使用了 Netflix Eureka 来实现服务注册与发现，它即包含了服务端组件，也包含了客户端组件。

- eureka-服务端：也可以称作服务注册中心，它同其他的服务注册中心一样，支持高可用配置。主要功能有，服务注册，服务续约，服务下线等。
- eureka-客户端，处理服务的注册和发现，eureka客户端向注册中心注册自身服务，并周期性的发送心跳来更新它的服务租约。同时，它也可以从服务端查询当前注册的信息并缓存本地周期性的刷新服务状态。

## 搭建 eureka 服务注册中心

pom.xml 中添加如下的依赖

```xml
<properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR2</spring-cloud.version>
</properties>

<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
</dependencies>

<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

通过 @EnableEurekaServer 注解启动一个服务注册中心。

服务注册中心常用配置选项

- eureka.instance.hostname 主机名称
- eureka.client.regiter-with-eureka [true/false] 是否将自身注册到服务注册中心
- eureka.client.fetch-registry [true/false] 是否检索注册信息

## 注册一个 eureka-client 服务提供者

pom.xml 加入如下依赖，其中 web 是为了测试，实际可根据需求来增加

```xml
<properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR2</spring-cloud.version>
</properties>

<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
</dependencies>

<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
</dependencyManagement>
```

在启动类添加 @EnableDiscoveryClient 注解来开启服务提供功能。

eureka 服务提供者常用配置

- spring.application.name = xxx 应用名称
- eureka.client.serviceUrl.defaultZone = http://xxxx:port/eureka/ 服务注册中心地址

启动后，就构建了一个服务提供者。

## 服务治理机制

### 服务提供者-服务注册

服务提供者在启动时，会通过 REST 请求的方式，讲自己注册到 Eureka Server 上，并携带一些元数据信息。Eureka Server 再接收到 REST 请求后，会将元数据存储在一个双层的 Map 中，其中第一层 key 是服务名，第二层的 key 是具体服务的实例名称。

同时，服务注册前，要确保 eureka.client.register-with-eureka = true 从而开启自动注册功能。

### 服务提供者-服务同步

服务提供者在 eureka server 进行注册时，通过服务同步，服务注册中心会将该请求转发给集群中相连接的其他注册中心，从而实现注册同步。

### 服务提供者-服务续约

注册完服务后，服务提供者会维护一个心跳来持续告诉 eureka server 服务存活的信息，通过心跳来避免 eureka server 将服务实例剔除。

主要通过下列两个参数进行配置，调整

- eureka.instance.lease-renewwal-interval-in-seconds = 30 # 心跳间隔
- eureka.instance.lease-expiratition-duration-in-seconds = 90  # 服务过期间隔

### 服务消费者-获取服务

启动服务消费者时，它会发送 REST 请求给服务注册中心，来获取上面注册的服务请求。为了性能考虑，eureka server 会维护一份可读的服务清单返回给客户端，同时该缓存清单每隔30秒更新一次。

通过设置 eureka.client.fetch-regitsry = true 来设置可以获取服务，如果是false ，则消费者无法进行获取服务。

通过设置 eureka.client.registry-fetch-interval-seconds = 30 来设置服务获取的间隔

### 服务消费者-服务调用

服务消费者获取服务清单后，通过服务名来获取具体提供服务的实例名和该实例名对应的元数据。根据这些信息，消费者来决定具体调用那些实例。

对于访问实例的选择，eureka 中有 region 和 zone 的概念，一个 region 中可以包含多个 Zone ，每个服务客户端需要被注册到一个 Zone 中，所以每一个客户端对应一个 region 和 一个 zone 。

进行服务调用时，优先访问同一个 zone 中的服务提供方。若访问不到，就访问其他的 zone。

### 服务注册中心-失效剔除

eureka-server 在启动时会执行一个定时任务，默认每隔一段时间（60S），去剔除清单中超时（默认90S）没有续约的服务。

### 服务注册中心-自我保护

eureka-server 在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%，如果出现低于的情况，会将当前的实例注册信息保护起来，让这些实例不会过期。

这样会引发一个问题，如果保护期间内，这些实例出现问题，客户端很容易拿到实际已经不存在的服务器实例，会出现调用失败的现象，所以客户端要必须要有对应的容错机制。
