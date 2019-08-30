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
