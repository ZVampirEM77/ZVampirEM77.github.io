---
layout: post
title: CentOS7 上安装 python 3.6
date: 2018-08-11 00:00:00 +0300
img: linux/python/install_python36_in_centos7_1.png # Add image post (optional)
stickie: false
tags: [Python] # add tag
---

# 获取相应版本的 Python 源码包

本文以 Python 3.6.3 为例

```
$ wget https://www.python.org/ftp/python/3.6.3/Python-3.6.3.tar.xz
```

解压

```
$ unxz Python-3.6.3.tar.xz

$ tar -xvf Python-3.6.3.tar
```

<br />
<br />

# 安装几个可选安装包

因为后面处理过程中需要，所以预先在此将这几个安装包进行安装

```
# 解决 import bz2 报错

$ yum install  bzip2-devel

# 解决 import curses 报错

$ yum install  ncurses-devel

# 解决 import sqlite3 报错

$ yum install sqlite-devel

# 解决 _dbm _gdbm 缺失提醒

$ yum install gdbm-devel

# 解决 read-line 缺失提醒以及方向键不 work 的问题

$ yum install readline-devel
```

<br />
<br />

# 编译安装源码包

```
$ ./configure --prefix=/usr/local/python3.6 --enable-optimizations --with-ssl --enable-shared
```

此处，加入 --with-ssl 编译选项是为了解决后面通过 pip 安装软件包时，可能会报出 ssl 相关的错误。

加入 --enable-shared 配置选项，以支持后面其他服务依赖 python3.6 的库文件来编译动态共享库，否则会报出类似如下的错误:

```
relocation R_X86_64_32 against `a local symbol' can not be used when making a shared object; recompile with -fPIC
...
```

```
$ make -j16

$ make install
```

至此，已成功安装 python 3.6，安装目录为 /usr/local/python3.6。


<br />
<br />

# 创建所需的虚拟环境

```
$ python3 -m venv .env

$ source ~/.env/bin/activate
```
