---
layout: post
title: Ceph 开发者月报 - 四月篇
date: 2018-04-24 00:00:00 +0300
img: ceph_monthly/April/ceph_monthly_april_1.png # Add image post (optional)
stickie: false
tags: [Ceph 月报] # add tag
---

# 对象存储

**LTTNG tracing 支持分析 RGW**

lttng: Trace rgw data transfer, bi entry and object header update processes (https://github.com/ceph/ceph/pull/20556)

为了实现对 Ceph 的相关处理过程进行追踪，以通过分析追踪结果对 Ceph 相关处理过程的性能进行优化，社区为 Ceph 引入了 LTTng。但之前社区主要是对 OSD、bluestore 等底层组件的处理过程进行了追踪。在上面的提交中，社区为 RGW 引入了追踪处理，主要是针对 RGW 中的数据迁移、bucket index 记录和 head object 的更新这几个主要的耗时处理过程进行追踪。

<br />

**对象存储前端 beast 支持 SSL**

rgw: add ssl support to beast frontend (https://github.com/ceph/ceph/pull/20464)

在上面的提交中，社区为 RGW 的 beast 前端添加了对 SSL 请求的支持。新增的相关配置选项包括：

> ssl_port        –    用来设置监听 SSL 请求的端口
>
> ssl_endpoint        –    用来设置监听来自指定地址的 SSL 请求
>
> ssl_certificate        –    用来指定 SSL 证书的系统路径
>
> ssl_private_key        –    用来指定 SSL 私钥文件的系统路径

<br />

**对象存储支持多因子认证**

rgw: mfa support (https://github.com/ceph/ceph/pull/19283)

MFA (Multi-Factor Authentication) ，顾名思义，多重要素认证，是 AWS S3 提供的一个提高数据安全性的功能。它能够在用户名称和密码之外再额外增加一层保护，当用户启用 MFA 功能后，在进行操作时，除了要提供用户名和密码外，还需要提供来自 MFA 设备的身份验证代码。AWS 官方支持的 MFA 设备包括：

1. 虚拟 MFA 设备
2. 硬件遥控钥匙 MFA 设备
3. 硬件显卡 MFA 设备
4. SMS MFA 设备
5. 适用于 AWS GovCloud (US) 的硬件遥控钥匙 MFA 设备

当前在 AWS S3 中，MFA 被主要应用在对启用了 versioning 功能的 bucket 中的 object 数据进行删除的处理场景。用户可以在配置启用 bucket versioning 功能时，对 MFA Delete 功能进行配置。如：

> <VersioningConfiguration xmlns=”http://s3.amazonaws.com/doc/2006-03-01/“>
>      <Status>VersioningState</Status>
>      <MfaDelete>MfaDeleteState</MfaDelete>
> </VersioningConfiguration>

在上面的提交中，社区为 RGW 新增了 MFA 功能的支持。主要是通过为 RGW 实现了一个虚拟 MFA 设备，在用户需要进行 MFA 认证时，为其生成 one-time passward 来完成身份认证。针对 MFA 功能，新增了如下 admin commands：

> Create a new MFA TOTP token
>
> —————————————————–
>
> $ radosgw-admin mfa create –uid=\<user-id\> –totp-serial=\<serial\> –totp-seed=\<seed\> [ –totp-seed-type=\<hex|base32\> ] [ –totp-seconds=\<num-seconds\> ] [ –totp-window=\<twindow\> ]
>
> <br />
>
> List MFA TOTP tokens
>
> ————————————————————
>
> $ radosgw-admin mfa list –uid=\<user-id\>
>
>
> Show MFA TOTP token
>
> ————————————————————
>
> $ radosgw-admin mfa get –uid=\<user-id\> –totp-serial=\<serial\>
>
>
> Delete MFA TOTP token
>
> ———————————————————-
>
> $ radosgw-admin mfa remove –uid=\<user-id\> –totp-serial=\<serial\>
>
>
> Check MFA TOTP token
>
> ———————————————————
>
> Test a TOTP token pin, needed for validating that TOTP functions correctly.
>
> # radosgw-admin mfa check –uid=\<user-id\> –totp-serial=\<serial\> –totp-pin=\<pin\>
>
>
> Re-sync MFA TOTP token
>
> ———————————————————
>
> In order to re-sync the TOTP token (in case of time skew). This requires
> feeding two consecutive pins: the previous pin, and the current pin.
>
> # radosgw-admin mfa resync –uid=\<user-id\> –totp-serial=\<serial\> –totp-pin=\<prev-pin\> –totp=pin=\<current-pin\>

<br />
<br />

# 块存储

**块存储设备映射支持禁止 discard 操作**

rbd: add notrim option to rbd map (https://github.com/ceph/ceph/pull/21056)

在上面的提交中，社区为 ‘rbd device map‘ 命令新增了 ‘notrim’ 选项。若在执行 rbd device map 命令对 image 进行映射时 enable 了该选项，则所有针对该 image 的 discard 请求都会返回 ‘-EOPNOTSUPP’ 错误，所有的写 ‘0’ 请求都被要求降级为手动执行，从而可以避免在映射一个 fully provisioned image 时对 image 进行擦除的错误处理。

<br />
<br />

# 统一存储层

**支持运行时使能 lz4 和 brotli 压缩算法**

osd,compressor: Expose compression algorithms via MOSDBoot. (https://github.com/ceph/ceph/pull/20558)

在 Ceph 当前的版本中，若要启用 lz4 和 brotli 两种压缩算法对应的压缩处理，需要在对 Ceph 仓库进行编译时显式指定。这往往就会导致如

（http://tracker.ceph.com/issues/20853）

中所 report 的问题，即若在编译过程中未显式指定，lz4 和 brotli 两种压缩算法对应的压缩处理默认是不可用的。如果后面为集群配置使用该压缩功能对数据进行处理，则会出错。在上面的提交中，社区通过为 OSD 新增了名为 supported_compression_algorithms 的元数据用于记录当前所有可用的压缩处理。

<br />
<br />

# 集群管理

**MGR 支持发送 PG最多和最少的 OSD 到 zabbix**

mgr/zabbix: Send max, min and avg PGs of OSDs to Zabbix (https://github.com/ceph/ceph/pull/21043) 

在实际的生产环境中，经常会发生有 OSD 宕掉了(状态为 down + out)，从而引起相关 PG 的重映射，即相关 PG 映射到其他健康的 OSD 上，以取代宕掉的 OSD，保证集群中相应数据的副本数。当宕掉的 OSD 数目非常多时，就可能会造成集群中某些 OSD 上的 PG 数目过多，从而消耗过多的系统资源。在上面的提交中，社区支持了发送 OSD 的最大、最小和平均 PG 数到 zabbix 中。这样就可以支持 admin 用户针对 OSD 上的 PG 数来创建 triggers，自定义当 OSD 对应的 PG 数过多时的一些处理操作。

<br />

**MGR 支持异步任务**

mgr/dashboard: asynchronous task support (https://github.com/ceph/ceph/pull/20870)

在上面的提交中，社区为 dashboard 增加了任务异步处理的支持

<br />

**MGR 支持收集所有的 op**

mon,mgr: make osd_metric more popular and report slow ops to mgr (https://github.com/ceph/ceph/pull/20660)

在上面的提交中，社区对 monitor 进行了完善，为 monitor 新增了 3 个 admin 命令：

> dump_historic_ops   –   用于罗列出当前的所有 op；
>
> dump_historic_ops_by_duration     –   用于罗列出当前的所有 op，并且按照 op 的处理时间对结果进行排序；
>
> dump_historic_slow_ops    –   用于罗列出当前所有处理较慢的 op；

那么如何判断哪些 op 被处理的较慢呢？这主要是通过引入一个阈值配置项来实现的，当 op 的处理时间大于该阈值时，就被认为是处理较慢的 op。在上面的提交中，引入的相关配置项包括：

> mon_enable_op_tracker           –         enable/disable MON op tracking
>
> mon_op_complaint_time          –        time in seconds to consider a MON OP blocked after no updates
>
> mon_op_log_threshold          –         max number of slow ops to display
>
> mon_op_history_size          –          max number of completed ops to track
>
> mon_op_history_duration         –        expiration time in seconds of historical MON OPS
>
> mon_op_history_slow_op_size          –          max number of slow historical MON OPS to keep
>
> mon_op_history_slow_op_threshold            –           duration time in seconds of an op to be considered as a historical slow op

<br >

**MON支持对 osdmap 进行 prune**

mon: osdmap prune (https://github.com/ceph/ceph/pull/19331)

在 Ceph 中存在配置项  mon_min_osdmap_epochs， 用于限制 monitor 能够保存的 osdmap epochs 的数量限制，默认为 500； 当 osdmap epochs 的数量超过该配置项限制的值时，较老的 osdmap epochs 将会被移除，以保证保存的 osdmap epochs 的数量始终在该配置项限制的范围内。在移除 osdmaps epoch 的处理过程中，实际上是存在一些限制的。这些限制被定义在 OSDMonitor::get_trim_to() 函数中。如果在移除处理过程中，其中的一个条件没有被满足，则不会进行移除操作，那么就会出现被保存的 osdmap epochs 的数量超过 mon_min_osdmap_epochs 配置项的限制的情况。如果当前集群环境在多次移除处理时都不满足上面的条件限制(e.g. unclean pgs)，则可能会造成 monitor 保存了过多的 osdmap epochs。这可能会对底层存储造成非常大的压力。

在上面的提交中，社区实现了针对上面保存 osdmap epochs 过多的情况下，对 osdmap 进行裁剪的处理，即只保留一段连续 osdmap epochs 记录的左右两个边界点，中间的记录会被裁剪掉。例如，若当前系统中保存了 50000 个 osdmap epochs，

![50000 osdmap epochs](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph_monthly/April/ceph_monthly_april_2.png?raw=true)

经过 osdmap prune 功能处理后，保存的记录为

![after osdmap prune](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph_monthly/April/ceph_monthly_april_3.png?raw=true)

在该提交中引入的主要配置项包括：

> mon_osdmap_full_prune_min        –    定义了一个阈值，当 osdmap epochs 记录超过该阈值后，就会进行 prune 处理，默认为 10000；
>
> mon_osdmap_full_prune_interval        –    用来定义裁剪操作的记录数量间隔，如上面的默认的 10；
>
> mon_osdmap_full_prune_enabled        –    是否启用 osdmap prune 功能

<br />

**MGR 引入 新插件 iostat**

mgr/iostat: implement ‘ceph iostat’ as a mgr plugin (https://github.com/ceph/ceph/pull/20100)

与

mgr/iostat: print output as a table (https://github.com/ceph/ceph/pull/21338) <span style="color:red;">(当前未合并)</span>

在上面的两个 PR 中，社区为 mgr 实现了 ceph iostat 插件，可用于对当前集群中客户端性能进行监测。

<br />
<br />

# 基础库

无
