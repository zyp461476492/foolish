---
title: Hexo + Github Page 搭建个人博客
categories: 教程
tags:
  - hexo
  - 教程
date: 2019-04-10 14:43:02
---

>借助 Hexo + Github Page 搭建个人博客，使用主题为nexT

<!-- more -->

## 环境准备

- 系统版本：win10 64 位
- [Node.js][node] 版本: 10.14.1
- [git][git] 版本: 2.19.2.windows.1
- [atom][atom]: 用做 markdown 语法编辑器

## hexo 配置
进入 [hexo][hexo] 官网，点击文档，参考文档进行配置。

首先安装需要安装 Node.js（版本要大于6.9） 和 Git。随后执行命令安装hexo。

```
# 全局安装 hexo-cli
$ npm install hexo-cli -g
```

安装 Hexo 完成后，执行下列命令，新建一个Hexo项目

```
$ hexo init <folder>
$ cd <folder>
# 安装所需要的依赖等
$ npm install
```

随后修改  **_config.yml** 配置文件，首先配置网站的基本信息，我的配置如下所示

```
title: 张蕴鹏的博客                    # 你的网页标题
subtitle: stay hungry stay foolish    # 你的副标题
description: Just do IT!              # 你的网站描述
keywords: IT LOG STUDY                # 你的网站关键字
author:                               # 作者名称
language: zh-CN                       # 语言类别
timezone:                             # 时区
```


## 主题选择及配置

## hexo 部署至Github Pages

## 遇到问题及存在的疑惑


[node]: https://nodejs.org/
[git]: https://git-scm.com/
[atom]: https://atom.io/
[hexo]: https://hexo.io/zh-cn/
