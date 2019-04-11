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

```bash
# 全局安装 hexo-cli
$ npm install hexo-cli -g
```

安装 Hexo 完成后，执行下列命令，新建一个Hexo项目

```bash
$ hexo init <folder>
$ cd <folder>
# 安装所需要的依赖等
$ npm install
```

随后修改  **_config.yml** 配置文件，首先配置网站的基本信息，我的配置如下所示

```xml
title: 张蕴鹏的博客                    # 你的网页标题
subtitle: stay hungry stay foolish    # 你的副标题
description: Just do IT!              # 你的网站描述
keywords: IT LOG STUDY                # 你的网站关键字
author:                               # 作者名称
language: zh-CN                       # 语言类别
timezone:                             # 时区
```

至此，Hexo 的基础配置就已完成，执行下列命令，启动本地预览（默认地址为 localhost:4000）

```bash
$ hexo s
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```

效果如下

## Hexo基础操作

### 发布一篇文章

执行下列的命令来发布一篇新的文章

```bash
$ hexo new [layout] <title>
INFO  Created: E:/blog/foolish/source/[layout path]/<year>-<month>-<day>-<title>.md
```

其中 layout 是可选参数，默认设置的 layout 为 `post` ，如果需要修改默认配置的 layout ，可以通过修改 `_config.yml` 的 `default_layout` 来变更默认 `layout` 。

title 是对应文章的名称，生成的 MD 文件的名称可以通过修改 `_config_yml` 的 `new_post_name` 来变更。

### Layout

Hexo 默认包含了下列表格中所示的 Layout ，具体如果新建 Layout ，笔者暂时还没有弄清楚。

Layout | 路径 | 功能
:---:|:---:|:---:
post | source/_posts | 发布
page | source | 未知功能
draft | source/_drafts | 草稿

### 草稿

执行下列的命令可以将 `layout` 为 `draft` 内的文章移动至 `source/_posts` 下

```bash
$ hexo publish [layout] <title>
INFO  Published: E:/blog/foolish/source/[layout path]]/<year>-<month>-<day>-<title>.md
```

### 文章模板(Scaffold)

新建文章时，可以通过指定模板名称来新建文件，命令如下

```bash
$ hexo new [layout] [scaffold name] <title>
INFO  Created: E:/blog/foolish/source/[layout path]/<year>-<month>-<day>-<title>.md
```

其中 `scaffold name` 为 `scaffolds/` 目录下对应的模板名称。

>注意，layout 要在模板名称之前， 否则默认 layout 为 post

## 主题选择及配置

本博客选用的主题为 [nexT][nexT]。

### nexT 主题的安装和配置

本文选择了下载最新的 `release` 版本，手动配置的方式，实际可以下载git仓库版本等，具体请参考 [nexT][nexT] 文档。

执行下列命令获取最新的 `release` 版本，并将下载后主题样式文件放在 Hexo 生成目录的 `themes/` 文件夹下

```bash
$ cd your-blog-path/
$ mkdir themes/next
$ curl -s https://api.github.com/repos/theme-next/hexo-theme-next/releases/latest | grep tarball_url | cut -d '"' -f 4 | wget -i - -O- | tar -zx -C themes/next --strip-components=1
...输出下载信息
```

修改 [`themes/next/_config.yml`][nextConfig] 文件，修改下列内容。更多定制内容可以根据 nexT 的文档

```yml
# 页脚设置
footer:
    ...
# 右上角是否展示 github banner 信息
github_banner:
    ...
# 配置首页菜单
menu:
   ...
# 配置展示的社交链接
social:
   ...
```

## hexo 部署至Github Pages

## 遇到问题及存在的疑惑

[node]: https://nodejs.org/
[git]: https://git-scm.com/
[atom]: https://atom.io/
[hexo]: https://hexo.io/zh-cn/
[next]: https://theme-next.org/
[nextConfig]: https://github.com/zyp461476492/foolish/blob/master/themes/next/_config.yml
