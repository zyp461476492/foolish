---
title: spring-cloud微服务实战 第二章学习笔记
date: 2019-08-29 11:19:34
tags:
    - 学习笔记
    - spring 
    - spring cloud
    - spring boot
categories: 学习笔记
---

> spring-cloud 学习笔记，第二章，记录学习中觉得重要的知识点，供个人回顾使用。

<!-- more -->

## spring-boot 配置文件

spring-boot 默认配置文件路径为 /src/main/resources/application.properties，目前实际项目中，一般使用 .yaml 格式文件作为配置文件。

yaml 采用的格式是类似大纲形式的缩减形式。

spring-boot 可以自定义参数。

```yaml
server:
  port: 1234

book:
  name: 123
```

上述的配置中，我自定义了 book.name = 123 这个自定义属性，此时，如果是用 IDEA 作为开发工具，会提示 "cannot  resolve configuration property 'book.name'"。

遇到这种问题，我们需要通过 [Annotation Processor][annotation-processor] 生成我们自己的 Metadata 

首先，添加如下的依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

随后在项目的 /src/main/resources/META-INF/ 目录下，新增 additional-spring-configuration-metadata.json 文件，文件内容如下

```json
{
  "properties": [
    {
      "name": "book.name",
      "type": "java.lang.String",
      "description": "Description for book.name."
    }
  ]
}
```

这样，就完成了自定义参数的定义步骤。随后就可以在 java 代码中，通过 @Value 注解和 PalceHolder （格式为 ${...}）或 SpEl 表达式的方式来注入参数。

在 spring-boot 配置文件中，可以使用 ${random} 来生成一个随机数。

```java
@Component
public class TestComponent {
    @Value("${book.name}")
    private String name;
}
```

## 命令行参数

在使用命令行的方式启动 spring-boot 应用时，连续的两个减号 -- 就是对 application.properties 中的属性进行赋值操作。

```shell
# 通过命令参数对 server.port 进行赋值
java -jar xxx.jar --server.port=4567
```

## 多环境配置

实际项目，不通环境，例如生产和测试，对应的配置文件都有所不同。在 spring-boot 中，可以对多环境的配置文件进行配置。
多环境配置的文件名要满足 application-{profile}.properties 的格式，其中 {profile} 对应环境标识，例如

- application-dev.yaml 开发
- application-test.yaml 测试
- application-prod.yaml 生产

## spring-boot 加载顺序

1. 命令行传入参数
2. SPRING_APPLICATION_JSON 中的属性。SPRING_APPLICATION_JSON 是 JSON 格式配置在系统环境变量中的内容。
3. java:comp/evn 中的 JNDI 属性。
4. Java的系统属性，可以通过 System.getProperties() 获取的内容。
5. 操作系统的环境变量。
6. 通过 random.* 配置的随机属性。
7. 位于当前应用 jar 包之外，针对不同 {profile} 环境的配置文件内容，例如 application-{profile}.yaml 配置文件。
8. 位于当前应用 jar 包之内，针对不同 {profile} 环境的配置文件内容，例如 application-{profile}.yaml 配置文件。
9. 位于当前应用 jar 包之外的 application.yaml 配置文件。
10. 位于当前应用 jar 包之内的 application.yaml 配置文件。
11. 在 @Configuration 注解修改的类中，通过 @PropertySource 注解定义的属性。
12. 应用默认属性，使用 SpringApplication.setDefaultProperties 定义的内容。

优先级按照上面的顺序从高到低，数字越小优先级越高。

## spring-boot-starter-actuator 概述

actuator 按照端点的用途，可以将原生端点分为以下三类

- 应用配置类：获取应用程序中加载的应用配置，环境变量，自动化配置报告等与 spring boot 应用密切相关的配置类信息。
- 度量指标类：获取应用程序运行过程中用于监控的度量指标，比如内存信息，线程池信息，HTTP 请求统计等。
- 操作控制类：提供了对应用的关闭等操作类功能。

### 应用配置类

- /autoconfig: 该端点用来获取应用的自动化配置报告。
- /beans: 该端点用来获取应用上下文中创建的所有 Bean。
- /configprops: 该端点用来获取应用配置中属性信息报告。
- /env: 该端点和 /configprops 不同，它用来获取应用所有可用的环境属性报告。
- /mappings: 该端点用来返回所有 Spring MVC 的控制器映射关系报告。
- /info: 该端点用来返回一些应用自定义的信息。

### 度量指标类

- /metrics: 该端点用来返回当前应用的各类重要度量指标，比如内存信息，线程信息，垃圾回收信息等。
- /health: 该端点用来获取应用的各类健康指标信息。
- /dump: 该端点用来暴露程序运行中的线程信息。
- /trace: 该端点用来返回基本的 HTTP 跟踪信息。

### 操作控制类

- /shutdown: 原生端点中只提供了用来关闭应用的端点。

[annotation-processor]: https://docs.spring.io/spring-boot/docs/2.1.7.RELEASE/reference/html/configuration-metadata.html#configuration-metadata-annotation-processor
