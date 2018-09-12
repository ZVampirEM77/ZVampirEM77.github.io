---
layout: post
title: 在 CentOS 上安装 MySQL
date: 2018-09-12 00:00:00 +0300
img: linux/install_mysql/install_mysql_1.jpg # Add image post (optional)
stickie: false
tags: [Linux, MySQL] # add tag
---

# 安装 MySQL 官方 yum 源

从 MySQL 官网下载所需的 rpm 包

https://dev.mysql.com/downloads/repo/yum/

因为笔者本地的环境是 CentOS 7，并且想使用最新的 MySQL 8.0 版本，所以笔者下载的是

mysql80-community-release-el7-1.noarch.rpm

rpm 包。

下载到 rpm 包之后，在本地进行安装

```
sudo yum localinstall mysql80-community-release-el7-1.noarch.rpm
```

执行完上述命令后，会在本地 /etc/yum.repos.d 目录下看到 MySQL 相关的 yum 源配置文件：

- mysql-community.repo
- mysql-community-source.repo

<br />
<br />

# 安装 MySQL

```
sudo yum install mysql-community-server
```

<br / >
<br />

# 启动 MySQL

```
systemctl start mysqld.service

systemctl list-units | grep mysql
```

<br />
<br />

# 访问 MySQL

MySQL 默认创建了 root 用户的密码，这个密码打印在 MySQL 的日志文件/var/log/mysqld.log中，可以通过temporary password关键字来找出这个临时的密码。

使用该密码访问 MySQL

```
mysql -u root -p

# Input the password
```

<br />

修改密码

```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```

将密码修改为了 "MyNewPass4!" 。

<br />

至此，就可以开始使用 MySQL 数据库了。
