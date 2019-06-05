---
title: ubuntu 18.04 make 方式安装 redis
tags:
  - redis
  - ubuntu
  - linux
categories: 'redis, linux'
date: 2019-06-05 13:57:00
---

> 以本地编译的方式，在 ubuntu 18.04 环境下部署 redis

<!-- more -->

## 下载及编译

参考官网教程，执行如下命令来进行 redis 本地编译。

```bash
$ wget http://download.redis.io/releases/redis-5.0.5.tar.gz
$ tar xzf redis-5.0.5.tar.gz
$ cd redis-5.0.5
$ make
print some tips ...
```

## 遇到问题即解决方式

如果出现下列的错误提示：

```bash
cd src && make all
make[1]: Entering directory '/home/ubuntu/redis-x.x.xx/src'
    CC adlist.o
/bin/sh: 1: cc: not found
Makefile:228: recipe for target 'adlist.o' failed
make[1]: *** [adlist.o] Error 127
make[1]: Leaving directory '/home/ubuntu/redis-x.x.xx/src'
Makefile:6: recipe for target 'all' failed
make: *** [all] Error 2
```

说明没有安装 gcc ，输入下列的命令安装 gcc 即可。

```bash
sudo apt-get install gcc
```

上述问题解决后，如果出现下列的错误提示：

```bash
cd src && make all
make[1]: Entering directory '/home/ubuntu/redis-4.0.11/src'
    CC adlist.o
In file included from adlist.c:34:0:
zmalloc.h:50:10: fatal error: jemalloc/jemalloc.h: No such file or directory
 #include <jemalloc/jemalloc.h>
          ^~~~~~~~~~~~~~~~~~~~~
compilation terminated.
Makefile:228: recipe for target 'adlist.o' failed
make[1]: *** [adlist.o] Error 1
make[1]: Leaving directory '/home/ubuntu/redis-4.0.11/src'
```

则可通过下列命令来解决：

```bash
make MALLOC=libc
```

## 运行

按照官网的提示，输入下列命令启动:

```bash
zyp@DESKTOP-CI621JQ:~/redis-5.0.5$ src/redis-server
5509:C 05 Jun 2019 11:53:36.175 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
5509:C 05 Jun 2019 11:53:36.175 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=5509, just started
5509:C 05 Jun 2019 11:53:36.175 # Warning: no config file specified, using the default config. In order to specify a config
5509:M 05 Jun 2019 11:53:36.175 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.5 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 5509
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

```
