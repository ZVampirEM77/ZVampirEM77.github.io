---
layout: post
title: 修改 MySQL 数据库系统时区
date: 2019-12-09 00:00:00 +0300
stickie: false
tags: [Linux, MySQL] # add tag
---

# 修改 MySQL 时区

MySQL 默认的 UTC 时区和我们所在的东八区相差了 8 个小时，因此，如果不进行显式设置，表中记录的时间就会和现实时间产生偏差。

## 修改方法

1. 输入

```bash
show variables like "%time_zone%";
```

显示当前时区。


2. 通过

```bash
set global time_zone = '+8:00';
```

设置全局时间为东八区 (+ 8小时)


3. 通过

```bash
set time_zone = '+8:00';
```

修改当前会话的时区


4. 通过

```bash
flush privileges;
```

刷新，使改动立即生效。
