---
layout: post
title: Ceph 开发者月报 - 五月篇
date: 2018-05-30 00:00:00 +0300
img: ceph_monthly/May/ceph-monthly-may-1.png # Add image post (optional)
stickie: false
tags: [Ceph 月报] # add tag
---

# 对象存储

**完善 radosgw-admin bucket sync status 命令**

rgw: bucket sync status improvements, part 1 (https://github.com/ceph/ceph/pull/21788)

在上面的提交中，社区对

```
radosgw-admin bucket sync status
```

命令进行了重构。当前，执行该命令后，RGW 会将本地的同步状态与远端每一个 source zone 的 bilog 进行比较，以 shard 的粒度对比较后的同步状态信息进行统计、总结和展示。

例如，存在两个远端的 source zone，bucket index shard 为 8，则执行 “radosgw-admin bucket sync status”的输出为：

```
$ radosgw-admin bucket sync status –bucket BUCKET

  realm 43b972d8-9448-4bd5-a396-1415dfcc483a (dev)
  zonegroup 18b95c94-333f-465c-9869-cc06869eda1e (na)
  zone 1508260a-6c32-49db-8248-562aa1881c90 (na-2)
  bucket BUCKET[7559751c-8905-4c4b-bfab-83412318e701.4134.1]

  source zone 65ab7537-c7e9-48a3-97b2-e5808744140a (na-3)
              full sync: 0/8 shards
              incremental sync: 8/8 shards
              bucket is behind on 2 shards
              behind shards: [0,3]

  source zone 7559751c-8905-4c4b-bfab-83412318e701 (na-1)
              full sync: 8/8 shards
              full sync: 123 objects completed
              incremental sync: 0/8 shards
              bucket is behind on 8 shards
              behind shards: [0,1,2,3,4,5,6,7]
```

之前版本的 “radosgw-admin bucket sync status” 命令被更名为 “radosgw-admin bucket sync markers”，即现在执行

```
radosgw-admin bucket sync markers
```

命令，实现的是之前版本 “radosgw-admin bucket sync status” 命令的功能。

<br />

**RGW 支持 ES 新版本**

rgw: elastic: fixes for supporting Elasticsearch versions >= 5.0 (https://github.com/ceph/ceph/pull/21852) <span style="color:red;">(未合并)</span>

在上面的提交中，社区完善了 RGW ，对 ElasticSearch 5.0 以上的版本进行了支持。

<br />

**rgw 新增 librgw_admin_user 库**

new librgw_admin_user (https://github.com/ceph/ceph/pull/21439)

在上面的提交中，社区为 RGW 新增了一个 librgw_admin_user 库，以方便用户直接在其应用程序中进行调用来创建 RGW 中的用户。

<br />
<br />

# 块存储

**改进 rbd disk-usage 镜像容量的统计精度**

rbd: disk-usage can now optionally compute exact on-disk usage (https://github.com/ceph/ceph/pull/21912)

在上面的提交中，社区为 rbd disk-usage 命令引入了 –exact 参数，通过指定该参数，可以禁用fast-diff，以精确地计算当前 rbd快照的用量。

<br />

**librbd 深度拷贝支持扁平化镜像**

librbd: deep copy optionally support flattening cloned image (https://github.com/ceph/ceph/pull/21624)

在上面的提交中，社区为 librbd 实现了支持在进行深度拷贝过程中，解除克隆镜像间的父子关系的功能，以父镜像中的数据来填充克隆出来的子镜像。

<br />

**rbd-mirror 支持 active/active 模式**

rbd-mirror: optionally support active/active replication (https://github.com/ceph/ceph/pull/21915)

出于对 rbd-mirror 单点的高可用性，以及多点之间的可扩展性考虑，社区在上面的提交中，为 rbd-mirror 实现了对 active/active 模式的支持。

<br />
<br />

# 统一存储层

**OSD 支持在故障后强制重新创建 PG，而不再尝试恢复数据**

mon/OSDMonitor: require –yes-i-really-mean-it for force-create-pg (https://github.com/ceph/ceph/pull/21619)

在上面的提交中，社区为

```
osd force-create-pg
```

命令添加了 “–yes-i-really-mean-it” 确认选项，以保证该命令是在经过确认后执行的。

<br />

**mon metadata 命令支持显示系统可用压缩算法**

mon,osd: dump “compression_algorithms” in “mon metadata” (https://github.com/ceph/ceph/pull/21809)

Ceph 支持在编译时指定启用/禁用哪些压缩算法，但是却不支持对当前系统中所有可用的压缩算法进行汇总和展示。用户在使用操作集群的过程中，只能通过查阅文档了解当前 Ceph 支持哪些压缩算法，然后进行相关配置。这就会带来一个问题，就是用户配置使用的压缩算法在当前系统中是不可用的，导致用户的相关操作失败。

在上面的提交中，社区对这一问题进行了完善。在 “mon metadata” 的相关处理过程中，对当前系统中可用的压缩算法进行收集和展示。

<br />
<br />

# 集群管理

**完善 mgr module disable 命令**

mon,mgr: improve ‘mgr module disable’ cmd (https://github.com/ceph/ceph/pull/21188)

对 mgr module 命令进行了完善，对于：
1. disable 一个已经 disable 的 module；
2. disable 一个不存在的 module；
3. enable 一个已经 enable 的 module；
都会给出相应的错误信息。

例如

```
$ceph mgr module ls
{
    “enabled_modules”: [
        “balancer”,
        “dashboard”,
        “restful”,
        “status”
    ],

    “disabled_modules”: [
        {
            “name”: “dashboard_v2”,
            “can_run”: true,
            “error_string”: “”
        },
        {
            “name”: “influx”,
            “can_run”: false,
            “error_string”: “influxdb python module not found”
        },
        {
            “name”: “localpool”,
            “can_run”: true,
            “error_string”: “”
        },
        {
            “name”: “prometheus”,
            “can_run”: true,
            “error_string”: “”
        },
        {
            “name”: “selftest”,
            “can_run”: true,
            “error_string”: “”
        },
        {
            “name”: “smart”,
            “can_run”: true,
            “error_string”: “”
        },
        {
            “name”: “zabbix”,
            “can_run”: true,
            “error_string”: “”
        }
    ]
}

disable an already disabled module
$ceph mgr module disable smart
Error EINVAL: module smart is already disabled

disable an non existed module
$ceph mgr module disable non_exist_module
Error ENOENT: module non_exist_module does not exist

enable an already enabled module
$ bin/ceph mgr module enable balancer
Error EINVAL: module balancer is already enabled
```
<br />

**mgr 的集中式配置管理**

mgr: centralized setting/getting of mgr configs (https://github.com/ceph/ceph/pull/21442)

基于 Ceph 当前版本中 monitor 引入了集中式的配置管理来中心化存放整个集群的配置信息，在上面的提交中，社区对 mgr module configuration 针对中心化配置进行了相应的修改，主要是将config 和 store 进行了分离：

– mgr module 的配置选项主要是指在 module 的 “OPTIONS” 属性中进行声明的配置项信息。程序内部使用 “set_config” 和 “get_config” 来设置和获取 config 配置信息，外部可通过 “ceph config set”和 “ceph config get” 命令来完成对应的操作处理, 配置信息由 monitor 来集中进行存放和管理；

– 若 mgr module 有一些较大的数据需要保存，这种情况下，则需要使用 KV store 来进行保存。每个 mgr module 都有一个私有的 KV store, 程序内部可以使用 “set_store” 来将所需的数据存储在 KV store 中，同时可以使用 “get_store” 来获取 KV store 中保存的相应的数据。外部则可以通过 “ceph config-key [set\|get]” 来完成对应的操作处理.

Note:

任何在 ceph-mgr 外部对上面 KV store 中的 value 所做的修改，对正在运行的 module来说都是不可见的，直到下次 module 重启后，更改才会生效。

需要说明的是，以上对于 configuration options 和 KV store 的 set/get 操作都具有如下特性：

1. Reads are fast: ceph-mgr 会在本地内存中针对上述配置信息维护一个副本，因此多数情况下，get_config 和 get_store 处理会很快；
2. Write block: 若写操作访问的配置信息还未被持久化保存，则操作会被阻塞；

<br />

**mgr service daemons 注销功能支持**

mgr: allow service daemons to unregister from ServiceMap (https://github.com/ceph/ceph/pull/20761)

在上面的提交中，社区实现了 mgr 的 service daemons unregister 功能。主要是通过 service daemon 在关闭过程中，向 mgr 发送一个 MMgrClose 消息，mgr 在接收到消息后，会将该 service daemon 的相关信息从 ServiceMap 移除。

其主要的应用场景就是，在 service daemon 关闭过程中，对 mgr 的 ServiceMap 中的相关信息进行清除。

<br />

**dashboard 新增 TLS 协议支持**

mgr/dashboard: add TLS (https://github.com/ceph/ceph/pull/21627)

在上面的提交中，社区为 dashboard 增加了对 TLS 协议的支持。

<br /> 

**dashboard 新增 RGW 用户和存储桶管理特性**

mgr/dashboard: Add RGW user and bucket management features (https://github.com/ceph/ceph/pull/21351)

在上面的提交中，社区为 dashboard 新增了对集群中的用户和存储桶进行管理的新特性。

<br />

**mgr 新增 telegraf 模块**

mgr/telegraf: Telegraf module for Ceph Mgr (https://github.com/ceph/ceph/pull/21782)

Telegraf 是 InfluxData 下的一个子项目，由 Go 语言编写，是一个用以收集、聚合、统计、处理系统和服务指标数据的代理服务。Telegraf 具有内存占用小的特点，通过插件系统，可以非常方便的对其进行服务扩展。

在上面的提交中，社区为 mgr 实现了 Telegraf 模块，以支持将 Ceph 集群中的指标统计数据发送到 Telegraf 代理服务中，然后由 Telegraf 进行处理，将相关统计结果发送到配置的数据库服务中，如 InfluxDB、ElasticSearch、Graphite 等等。

因为当前这个模块主要是基于 Telegraf 的 socket_listener 插件进行工作，所以支持通过 UDP、TCP 以及本地 Unix Socket协议来发送数据。

<br />

**dashboard 引入通知组件**

mgr/dashboard: Display notification if RGW is not configured (https://github.com/ceph/ceph/pull/21785)

社区在这个 PR 中，主要为 dashboard 引入了两个组件：

– cd-info-panel

  用来显示通知信息；

– ModuleStatusGuardService

  一个路由监控服务。这个服务通过执行一个 GET “/api/<apiPath>/status” 的 REST API 调用来检查所请求的服务或模块是否可用，若不可用，则将用户的请求重定向到一个特定的 url 或是路由上。

通过上述的两个组件，实现了 dashboard 支持在检测到系统中没有配置 RGW 的情况下，显示通知提示信息。

<br />

**mgr 新增 Telemetry 模块**

mgr/telemetry: Add Ceph Telemetry module to send reports back to project (https://github.com/ceph/ceph/pull/21982)

在上面的提交中，社区为 Ceph 引入了 telemetry 模块。若启用该模块，则该模块将会以匿名的方式将本地 Ceph 集群的一些统计信息发送给 Ceph 官方，以便于社区更好的了解用户使用 Ceph 的行为方式。

需要注意的是，该模块不会统计发送本地集群的存储池名、对象名、对象内容等敏感信息；而是发送类似于 Ceph 集群的部署方式、所使用的 Ceph 的版本号、服务器所使用的操作系统版本等统计信息。以上数据将会以 HTTPS 请求发送到 https://telemetry.ceph.com/report 。

<br />
<br />

# 基础库

**QAT 压缩支持**

compressor: add QAT support (https://github.com/ceph/ceph/pull/19714)

QAT(Quick Assist Technology) 是Intel公司推出的一种专用硬件加速卡, 不仅对SSL非对称加解密
算法（RSA、ECDH、ECDSA、DH、DSA等）具有加速，而且对数据的压缩与解压也具有加速效果。

QATZip 是基于 QAT 用户层接口库而实现的专门针对压缩解压缩应用场景的用户层接口库
https://github.com/intel/QATzip

社区在上面的提交中，新增了对 QAT 压缩的支持。实际上，就是支持将压缩/解压的处理过程，由软件处理转换为专用硬件处理，这样可以大大的减小 CPU 的负载，同时可以很大的提升压缩/解压处理的效率。非常适合大数据的应用场景。

http://www.c114.com.cn/topic/images/4975/QATProductHandbooks-2016-CNFinalforPrint.pdf

<br /> 

**支持多条日志记录统一落盘**

log: disk write coalescing (https://github.com/ceph/ceph/pull/21847)

当日志等级被设置为很高时，每秒钟会生成大量的日志记录，这就会导致写日志操作成为影响系统性能的瓶颈之一。在上面的提交中，社区实现了多条日志记录的统一落盘处理，即不再针对每条日志记录单独执行系统调用写入磁盘，而是通过一个预分配的 buffer 对多条日志记录进行合并，然后统一调用一次系统调用进行落盘，减少了系统调用，提升了性能。

<br />
<br />

# 工具库

**完善 ceph-volume lvm list 命令**

ceph-volume include physical devices associated with an LV when listing (https://github.com/ceph/ceph/pull/21645)

在上面的提交中，社区实现了在执行

```
ceph-volume lvm list
```

命令时，支持列出与逻辑设备相关联的物理设备。当一个逻辑设备与多个物理设备相关联时，多个物理设备之间以 “,” 分隔。

<br /> 

**teuthology 整合 cosbench**

qa/tasks: run cosbench using the CBT task (https://github.com/ceph/ceph/pull/21656)

在上面的提交中，社区实现了将 cosbench 的 benchmark 测试整合到了 teuthology 的 CBT(Ceph Benchmarking Tool) 任务中。
