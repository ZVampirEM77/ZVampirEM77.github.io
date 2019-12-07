---
layout: post
title: MySQL 8.0.18 版本修改 root 用户密码 
date: 2019-12-07 00:00:00 +0300
stickie: false
tags: [Linux, MySQL] # add tag
---
# CentOS 7 运行 MySQL

## 安装 MySQL

从 http://repo.mysql.com/ 上获取对应版本的 rpm 包

```shell
wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
```

或者直接通过 yum 安装

```shell
yum install mysql-server -y
```

## 启动 MySQL

```shell
service mysqld start
```

至此，我们已经在 CentOS7 环境中运行起了一个 MySQL 数据库。


# 修改 MySQL 数据库 root 用户密码

初始，我们并不知道 MySQL 数据库中 root 用户的密码，因此，在通过

```shell
mysql -u root -p
```

登录时，可能会报错。因此需要手动对 root 用户的密码进行修改。

## 设置 MySQL 数据库支持免密登录

```shell
vim /etc/my.cnf

# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove the leading "# " to disable binary logging
# Binary logging captures changes between backups and is enabled by
# default. It's default setting is log_bin=binlog
# disable_log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
#
# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_default_authentication_plugin
# default-authentication-plugin=mysql_native_password

datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

在其中添加 skip-grant-tables 配置项

```shell
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove the leading "# " to disable binary logging
# Binary logging captures changes between backups and is enabled by
# default. It's default setting is log_bin=binlog
# disable_log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
#
# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_default_authentication_plugin
# default-authentication-plugin=mysql_native_password

datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

skip-grant-tables

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

重启 mysqld

```shell
service mysqld restart
mysql -u root -p
```

## 修改 root 用户密码

查看当前 root 用户的相关信息，在 MySQL 数据库的 user 表中

```shell
select host, user, authentication_string, plugin from user;
```

其中 authentication_string 字段记录的即为用户密码经过加密后的内容，在 MySQL v5.7.9 之后
废弃了 password 字段和 password() 函数

如果当前 root 用户的 authentication_string 字段不为空，则需要将其先置为空，以方便后面将 MySQL
修改为支持密码登录后，仍能对 root 用户进行免密登录

Note:

****MySQL 修改用户密码操作必须在进行密码校验的运行模式下进行，即将 MySQL 设置为用户
免密登录的状态下，是无法对用户密码进行修改的

```shell
use mysql;
update user set authentication_string='' where user='root';
```

然后退出 MySQL，重新编辑 my.cnf 配置文件，将其中刚才加入的 skip-grant-tables 配置项删除，
然后重启 MySQL。

使用 root 用户进行登录，因为已经将上面的 authentication_string 设置为空，所以可以免密码
登录

```shell
mysql -u root -p
```

使用 alter 修改 root 用户密码

```shell
alter user 'root'@'localhost' identified by 'YOUR PASSWORD'
```

至此，root 用户的密码就修改成功了，后面就可以通过修改后的密码来登录 MySQL 数据库
