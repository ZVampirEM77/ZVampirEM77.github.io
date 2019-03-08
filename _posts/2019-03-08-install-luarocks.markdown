---
layout: post
title: 安装 luarocks 包管理器
date: 2019-03-08 00:00:00 +0300
stickie: false
tags: [Lua] # add tag
---

# 什么是 LuaRocks

Luarocks是一个Lua包管理器，基于Lua语言开发，提供一个命令行的方式来管理Lua包依赖、安装第三方Lua包等，社区比较流行的包管理器之一，另还有一个LuaDist，Luarocks的包数量比LuaDist多


# 安装 LuaRocks

首先从 http://www.luarocks.org/en/Download 下载最新版本的 LuaRocks，当前最新版本为 3.0.4，在本示例中，将使用这个版本。

```
$ wget https://luarocks.org/releases/luarocks-3.0.4.tar.gz
```

解压源码包

```
$ tar zxpf luarocks-3.0.4.tar.gz
$ cd luarocks-3.0.4
```

编译安装

```
$ ./configure
$ make
$ make install
```
