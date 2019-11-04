---
title: 分布式文件系统-eureka-server 搭建笔记
date: 2019-10-30 10:33:48
tags:
    - 学习笔记
    - 分布式文件系统
    - spring 
    - spring cloud
    - spring boot
categories: 分布式文件系统, spring-cloud
---
> 自己参考文档搭建一个 eureka-server，并记录中间存在的问题，要点。

<!-- more -->

## 依赖准备

进入 [Spring Initializr][spring-initializr] 去查询最新的 spring-boot 版本和 eureka-server 依赖的版本，根据页面提示填写选项，下载需要的配置文件或手动拷贝。

这里使用的版本为

- spring-boot 2.2.0.RELEASE
- spring-cloud.version Hoxton.RC1

[spring-initializr]: https://start.spring.io/

## 高可用，Zones 和 Regions

Eureka Server 没有用到后端存储，但是在注册中心进行注册的服务实例，必须要发送心跳来保持它们最新的注册状态（所以只需要在内存中就可以完成）。

Clients 侧同样拥有一个缓存在内存中的 Eureka Server 服务中心注册过的注册表（所以不需要每次请求服务都要去注册中心）。

默认情况下，每一个 Eureka Server 也是一个 Eureka Client 并且要求至少提供一个服务 URL 来注册身份。如果你没有提供，这个服务可以正常的运行和工作，但你的日志中会充满许多无法注册身份的提示。

## 单机模式

server 和 client 注册表的缓存和心跳机制使得单独的 eureka server 对故障具有相当的弹性。

在单机模式下，你可能更希望关闭 client 侧的行为从而使的它不再保持重试当失败注册时。

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

注意：serviceUrl 里的 host 和 port 和本地实例的保持一致。

## 集群模式

通过运行多个实例，并要求他们彼此互相注册， Eureka Server 可以变得更具弹性和高可用。事实上，这是默认的的行为，因此要使其正常工作，只需要添加有效的 serviceUrl

```yaml
---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2/eureka/

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1/eureka/
```

通过设置 **eureka.instance.preferIpAddress** 为 **true**，可以让应用在运行时更倾向于使用 ip。

## 保护 Eureka Server

你可以保护你的 Eureka Server 通过添加 **spring-boot-starter-security** 依赖。默认情况下，Spring Security 要求验证 CSRF 对于每个发送的请求。Eureka Clients 没有提供通用处理，所以你需要单独进行设置

```java
@EnableWebSecurity
class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().ignoringAntMatchers("/eureka/**");
        super.configure(http);
    }
}
```
