---
title: ubuntu18.04 配置 redis 以 systemctl 方式启动
date: 2019-06-06 13:59:21
tags: 
  - redis
  - linux
categories: redis, liunx
---

> 本地编译安装 redis 后，启动，停止要通过加上路径来执行，如果能以服务的方式进行启动，停止，会方便很多，本文就是介绍如何将 redis 注册为服务。

<!-- more -->

## systemctl 概述

内核启动第一个用户空间进程是由init开始的，你可以在/sbin目录下找到init，它主要复制启动和

systemd 是新出现的 init，目前很多发行版的 linux 都已经或计划转入 systemd ，如果你的系统中有 /usr/lib/systemd 或 /etc/systemd 路径，则表示你所安装的系统中已经存在了 systemd 。

systemd 常见的命令

```bash
# 打开服务
sudo systemctl start service
# 关闭服务
sudo systemctl stop service
# 重启服务
sudo systemctl restart service
# 不中断正常功能下重新加载服务
sudo systemctl reload service
# 设置服务的开机自启动
sudo systemctl enable service
# 关闭服务的开机自启动
sudo systemctl disable service
# 查看活跃的单元
systemctl list-units
# 查看某个服务的状态
systemctl status service
```

## redis 配置 systemctl

进入 /usr/lib/systemd/system 路径中，创建 redis.service 文件

```text
#表示基础信息
[Unit]
#描述
Description=Redis
#在哪个服务之后启动
After=syslog.target network.target remote-fs.target nss-lookup.target

#表示服务信息
[Service]
Type=forking
#注意：需要和redis.conf配置文件中的信息一致
PIDFile=/var/run/redis_6379.pid
#启动服务的命令
#redis-server安装的路径 和 redis.conf配置文件的路径
ExecStart=/opt/redis/src/redis-server /opt/redis/redis.conf
#重新加载命令
ExecReload=/bin/kill -s HUP $MAINPID
#停止服务的命令
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

#安装相关信息
[Install]
#以哪种方式启动
WantedBy=multi-user.target
#multi-user.target表明当系统以多用户方式（默认的运行级别）启动时，这个服务需要被自动运行。
```

常用命令

```bash
# 开机启动
systemctl enable redis.service
# 启动
systemctl start redis.service
# 重启
systemctl restart service.service
# 停止
systemctl stop service.service
# 查看状态
systemctl status service.service
```

## 参考链接

- [ubuntu 服务相关命令说明][service]
- [redis 配置 systemctl][redis-config-systemctl]

[service]: https://blog.csdn.net/qq_37993487/article/details/79868857
[redis-config-systemctl]: https://blog.csdn.net/u011389474/article/details/72303156
