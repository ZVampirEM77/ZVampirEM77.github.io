---
layout: post
title: Ceph 开发者月报 - 二月篇
date: 2018-02-27 00:00:00 +0300
img: ceph-monthly-feb.png # Add image post (optional)
stickie: false
tags: [Ceph 月报] # add tag
---

# 版本发布

2018 年 2 月，Ceph 社区发布了其 Luminous 版本的第三个 bugfix 小版本 v12.2.3。该版本中的变更详见：

https://ceph.com/releases/v12-2-3-luminous-released/

<br />
<br />

# 对象存储

**RGW Beast 前端支持监听特定 IP 地址**

rgw: allow beast frontend to listen on specific IP address (https://github.com/ceph/ceph/pull/20000)

之前RGW 的 HTTP 前端 Beast 只支持监听来自指定端口的请求。上面的提交，实现了 Beast 前端支持监听特定 IP 地址请求的功能，通过在 rgw_frontend 配置项的 port 参数中以 “1.2.3.4:8080” 的形式来指定 RGW 的 Beast 前端只监听来自 IP: 1.2.3.4 的请求。因为当前默认的HTTP  前端 Civetweb 支持在 port 参数中配置对某一特定 IP 地址进行监听，所以这一改动不存在与 Civetweb 的兼容问题。

<br /> 

**RGW Beast 前端支持同时监听多个 endpoints**

rgw: beast frontend can listen on multiple endpoints (https://github.com/ceph/ceph/pull/20188)

基于上面 RGW Beast 前端支持监听特定 IP 地址 的提交，社区再度对 RGW Beast 前端进行了扩展，为 Beast 前端新增了 endpoint 配置项，并且对已有的 port 参数进行了扩展，支持同时配置多个 endpoint 和 port ，以实现 Beast 前端能够同时监听多个 endpoint 和 port。在内部，使用 multimap 来对配置的多个 endpoint / port 进行记录，针对每一个 endpoint / port，都会有单独的监听实例来进行处理。同时，社区也对 civetweb 的 port 参数进行了扩展，支持对多个 port 进行监听处理。

关于 endpoint 和 port 配置参数的具体说明：

> Beast
>
> =====
>
> endpoint
>
> Description:
>
> Sets the listening address in the form “address[:port]“,
>
> where the address is an IPv4 address string in dotted decimal
>
> form, or an IPv6 address in hexadecimal notation. The
>
> optional port defaults to 80. Can be specified multiple times
>
> as in “endpoint=::1 endpoint=192.168.0.100:8000“.
>
>  
>
>  port
>
>  Description:
>
>  Sets the listening port number. Can be specified multiple
>
>  times as in “port=80 port=8000“.

<br />

> Civetweb
>
> =======
>
> port
>
> Description:
>
> Sets the listening port number. For SSL-enabled ports, add an
>
> “s“ suffix like “443s“. To bind a specific IPv4 or IPv6
>
> address, use the form “address:port“. Multiple endpoints
>
> can either be separated by “+“ as in “127.0.0.1:8000+443s“,
>
> or by providing multiple options as in “port=8000 port=443s“.

<br />

**RGW 新增清除所有 usage 记录的处理**

rgw: add an option to clear all usage entries (https://github.com/ceph/ceph/pull/19322)

radosgw-admin 命令新增了 usage clear 选项来清除集群中的所有用户的所有 usage log 统计信息。

<br /> 

**RGW Bucket Policy 对 condition 配置相关处理功能的完善**

rgw: add support for tagging and other conditionals in policy (https://github.com/ceph/ceph/pull/17094)

在 AWS S3 中，bucket policy 是支持 condition 配置的，该 condition 域是用来指定 policy 生效的匹配条件的，只有在请求头中包含了满足 condition 域所有条件的请求内容，对应的 policy 才会生效，进而请求才会通过权限验证。具体见 S3 官方文档：

https://docs.aws.amazon.com/AmazonS3/latest/dev/amazon-s3-policy-keys.html

之前，RGW 的 Bucket Policy 是支持 condition 相关处理的，但是只支持针对 ListBucket 操作的

- s3:prefix
- s3:delimiter
- s3:max-keys
 
这几个基本的 condition 处理。在上面的提交中，社区对 RGW Bucket Policy 的 condition 相关处理进行了完善，新增了对 PutObj、PutACL、GetObj 等常用操作的

- s3:x-amz-canned-acl
- s3:x-amz-copy-source
- s3:x-amz-server-side-encryption
- s3:x-amz-server-side-encryption-aws-kms-key-id

以及 object tag 等常用 conditions 处理的支持。

<br />

**RGW-NFS 系统用量反馈信息修正**

RGW-NFS: Use rados cluster_stat to report filesystem usage (https://github.com/ceph/ceph/pull/20093)

之前，在通过 librgw && nfs-ganesha 来实现 NFS 会存在如下问题：

> df 命令返回的所有统计域的值均为 0

导致这一问题的原因见：

http://tracker.ceph.com/issues/22202

在上面的提交中，社区对这一问题进行了修正，通过获取 Rados 集群的各项统计信息来填充相应的统计字段，以提供给 df 显示正确的系统使用信息。

<br /> 

**新增 cache 相关操作的 admin 命令**

Cache Register! (https://github.com/ceph/ceph/pull/20144)

在上面的提交中，实现了对 RGW Cache 进行控制管理的相关命令，主要包括：

- cache list    –  list object cache, possibly matching substrings
- cache inspect   –  print cache element
- cache erase    –  erase element from cache
- cache zap    –   erase all elements from cache

<br />
<br />

# 块存储

**RBD 对 image 映射的相关处理命令进行统一**

rbd: unified way to map images using different drivers (https://github.com/ceph/ceph/pull/19711)

之前，在为 krbd、nbd、ggate 映射一个 RBD Image 时，需要分别使用三套命令，包括：

```
rbd map|unmap|showmapped
```

```
rbd ndb map|unmap|list
```

```
rbd ggate map|unmap|list
```

在上面的提交中，社区对这些 map image 相关的命令进行了清理和整合，将针对 krbd、nbd、ggate 的 map image 命令整合到一个统一的 command 中：

```
rbd device -t krbd|nbd|ggate|... map|unmap|list
```
<br />

**librbd clone v2 支持**

librbd: initial hooks for clone v2 support (https://github.com/ceph/ceph/pull/20176)

RBD 是支持 Snapshot Layering 功能的，即快照分层功能。该功能主要是支持对一个 image 的 snapshot 快照进行 clone 操作，生成一个新的 image，在快照这一层面形成一个父子关系的层级结构，如图：

![snapshot layer](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph-monthly-feb-1.png?raw=true)

整个功能的一个简化的处理流程为：

![snapshot layer workflow](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph-monthly-feb-2.png?raw=true)

在这里，把上述的针对 snapshot 的 clone 处理功能称为 clone v1。可以看到，在 clone v1 中，在对 snapshot 进行 clone 之前，需要先将 snapshot 设置为 protected，也就是说，在该 snapshot 所有相关的 child images (即 clone 自该 snapshot 的所有 child images) 被删除或解除父子关系 (rbd flatten 命令) 前，该 snapshot 是无法被删除的。

在上面的提交中，社区针对 snapshot 的 clone 处理实现了 v2 版本，在 clone v2 版本中，针对任何 snapshot 都可以进行 clone 处理，在 clone 之前不需要对 snapshot 进行 protect 的相关处理，也就是说，即使当前系统中存在 clone 自某一 snapshot 的 child image，依然可以删除该 snapshot ，但该删除操作不会真正的删除该 snapshot 的数据。只有在所有 clone 自该 snapshot 的 child images 与该 snapshot 解除关联后，才会真正的删除该 snapshot 对应的数据信息。

Clone v2 功能只有在 Mimic 及以上版本支持，可以通过  rbd_default_clone_format 配置项来配置所要使用的 clone 处理的版本。

<br />
<br />

# 统一存储层

**ceph osd stat 命令支持打印出 osd 的 epoch 信息**

osd/OSDMap: add osdmap epoch info when printing info summary (https://github.com/ceph/ceph/pull/20184)
 
<br />

**Ceph OSD 新增内置的 S.M.A.R.T. 检测**

osd: add ‘ceph tell osd.id smart’ (https://github.com/ceph/ceph/pull/19342)

S.M.A.R.T.，全称为“Self-Monitoring Analysis and Reporting Technology”，即“自我监测、分析及报告技术”，是一种自动的硬盘状态检测与预警系统和规范。通常，在 Linux 系统下，针对 S.M.A.R.T. 的相关处理操作均是通过 smartctl 命令来完成的。

在上面的提交中，社区实现了 ceph osd 的内置 SMART 检测。主要实现方式为，新增了 “ ceph tell osd.id smart ” 命令，在内部通过调用 smartctl 来进行 SMART 检测和结果数据的收集，然后在 mgr 中提供了了新的 smart 模块，通过该模块提供的 osd smart get 命令来获取检测信息给用户。

<br />

**Bluestore 新增 discard 处理来提升 SSD 的处理性能**

os/bluestore: add discard method for ssd’s performance (https://github.com/ceph/ceph/pull/14727)

在上面的提交中，社区为 Bluestore 新增了 discard 进行 trim 可以均摊 SSD GC 带来的负载，避免GC 造成响应时间不稳定。

<br />

**Bluestore 支持 Repair 功能**

os/bluestore: implement BlueStore repair (https://github.com/ceph/ceph/pull/19843)

Ceph 底层工具 ceph-objectstore-tool 支持通过 fsck 命令可以对受损OSD进行修复。

<br />

**PG 合并功能的实现**

WIP osd,mon: implement PG merging (https://github.com/ceph/ceph/pull/20469) <span style="color:red;">(未合并)</span>

集群扩容后，适当增大 PG 数量可以有效的提升集群性能。增加 PG 数量是通过 PG 分裂 ( Split ) 实现的。当大规模集群在缩容后，PG 数量过大导致占用过多计算资源，反而会影响集群性能。目前Ceph 是不支持减少 PG 数量的，通过实现 PG 合并 ( Merge ) 功能可以实现减少 PG 数量。

具体设计和实现计划见：

http://pad.ceph.com/p/pg-merging

<br />
<br />

# 集群管理

**Ceph 内置面板新增配置管理页面**

mgr/dashboard: add configuration setting browser (https://github.com/ceph/ceph/pull/20043)

![dashboard configuration setting browser](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph-monthly-feb-3.png?raw=true)

<br />

**mgr 新增对 python3 的支持**

mgr: fix py3 support (https://github.com/ceph/ceph/pull/20362)

<br />
<br />

# 基础库

无
