---
title: springboot 管理系统 jwt 认证
date: 2019-06-10 20:41:55
tags: 
  - spring
  - 管理系统
categories: springboot-管理系统
---

> 在基于 springboot 的管理系统中，将 jwt 集成在系统中，并作为系统前后端分离的认证方式。

<!-- more -->

## 什么是 jwt

jwt 全称为 json web token，它是一个开放标准，它是一种基于 json 格式，传递可以被验证和信任的信息。

jwt 由三个部分组成，这三个部分为

- header
- payload (body)
- signature
