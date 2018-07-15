---
layout: post
title: Ceph 开发者月报 - 三月篇
date: 2018-03-30 00:00:00 +0300
img: ceph_monthly/March/ceph-monthly-mar.png # Add image post (optional)
stickie: false
tags: [Ceph 月报] # add tag
---

# 版本发布

社区发布了 Ceph Luminous 版本的第四个 bugfix 版本 v12.2.4，该版本中的具体变动详见：

https://ceph.com/releases/v12-2-4-luminous-released/

<br />
<br />

# 对象存储

**RGW 添加配置 Multisite ElasticSearch module 文档**

doc: rgw add some basic documentation for sync plugins & ES (https://github.com/ceph/ceph/pull/15849)

社区在说明文档中新增了对 sync module 和 ElasticSearch sync module 的介绍说明。

<br />

**RGW GC 引入 AIO 提升效率**

rgw: gc use aio (https://github.com/ceph/ceph/pull/20546)

之前，在社区的邮件列表中，有用户反馈，其生产环境的分布式存储集群中，在有大量的待删除对象数据需要清除时，因为 rgw gc 的处理性能低下，导致无法及时的进行删除处理、释放集群的存储空间，详见：

http://lists.ceph.com/pipermail/ceph-users-ceph.com/2017-October/021825.html

在上面的提交中，社区对 rgw gc 的处理过程进行了修改，使用 rados aio (异步 IO 处理) 来进行 gc 处理，以提升 rgw gc 的处理效率。

<br />

**RGW Multisite 支持查看对象级别的同步状态**

rgw: improve sync status: display lagging objects (https://github.com/ceph/ceph/pull/20733) <span style="color:red;">(未合并)</span>

在这个 PR 中，社区为 radosgw-admin bucket sync status 命令新增了 --sync-detail 参数，用来列出 miltisite 数据同步处理过程中还有哪些对象数据没有被同步，以及还没有被同步处理的原因。

上面的提交与之前的两个提交：

rgw: improve sync status (https://github.com/ceph/ceph/pull/19573)

rgw: improve sync status: display behind bucket shards (https://github.com/ceph/ceph/pull/20027) <span style="color:red;">(未合并)</span>

相结合，共同实现了 multisite 同步处理状态信息获取和展示的细粒度化。

<br />

**RGW 改进 cache 一致性**

Cheque Caching (https://github.com/ceph/ceph/pull/20883) <span style="color:red;">(未合并)</span>

在上面的 PR 中，社区主要对 rgw cache 在数据不一致问题上的处理进行了完善，完善机制主要包括：

1. 为 rgw cache 的数据一致性处理引入了 X-RGW-Cache-Epoch 请求头，通过新引入的配置项 rgw_cache_epoch_header 对是否启用该请求头进行控制。若启用该请求头，则每当有 cache 分发失败时，都会更新该请求头中所记录的 cache epoch ，并通知给所有其他的缓存节点。其他节点在收到该请求头后，和自己本地保存的 cache epoch 进行对比，若发现请求头中的 cache epoch 更新，则清空当前所有的缓存数据，以重新更新缓存。但需要注意的是，需要引入一个 proxy 或是 LB 来为所有请求附加上这一请求头；

2. 对 rgw cache notify 机制进行完善，增加了重试机制。

<br />

**RGW 改进 list bucket 操作性能**

rgw: ability to list bucket contents in unsorted order for efficiency （https://github.com/ceph/ceph/pull/21026) <span style="color:red;">(未合并)</span>

在上面的 PR 中，社区对 list bucket 的性能进行了优化，即支持在进行 list bucket 操作时，指定不要对返回结果进行排序。众所周知，当某一存储桶中的对象数过多时，在通过 list bucket 操作来列出存储桶中的所有对象时，性能非常低下，而在这过程中，在 RGW 内部处理时，还会对查询得到的结果进行排序操作，这无疑是雪上加霜。在上面的 PR 中，社区通过为 Get Bucket 操作的 Restful API 新增了 allow_unordered (for Swift) 和 allow-unordered (for S3) 参数来允许用户在进行 list bucket 的操作中指定不要对查询得到的结果进行排序操作，以提升 list bucket 操作处理的效率。

<br />

**RGW Beast 前端支持异步流控**

rgw: add async Throttle for beast frontend (https://github.com/ceph/ceph/pull/20990) <span style="color:red;">(未合并)</span>

在前面的 Ceph 月报中，我们介绍过，社区为 RGW 引入了 Beast 前端以实现 RGW IO 处理的异步化。将 IO 处理过程实现为异步化固然能够提高对 IO 请求的处理效率，但若当前系统中存在过多的异步 IO 处理时，无疑会消耗大量的系统资源。在上面的 PR 中，社区为 RGW 的 Beast 前端添加了相应的限制，通过引入 rgw_max_concurrent_requests 配置项来限制 RGW 能够同时处理的异步 IO 数，进而对系统资源消耗进行限制。

<br />

**RGW 支持重置特定用户的状态信息**

rgw: add an option to recalculate user stats (https://github.com/ceph/ceph/pull/20853) <span style="color:red;">(未合并)</span>

在上面的 PR 中，社区为 radosgw-admin user stats 引入了 –reset-stats 参数，用于根据当前 user.bucket 的 omap 记录信息对 user 的状态信息进行重置(或者说更新)。

<br />
<br />

# 块存储

**RBD 一致性组支持重命名**

rbd: add group rename methods (https://github.com/ceph/ceph/pull/20577)

在上面的提交中，社区新增了对 Consistency Group，即 RBD 一致性组，进行 rename 重命名操作的支持，并为 rbd 命令新增了如下选项来进行 group rename 操作：

> group rename         —        Rename a group within pool.
>
> rbd help group rename
>
> usage: rbd group rename [–pool <pool>] [–group <group>]
>
> [–dest-pool <dest-pool>] [–dest-group <dest-group>]
>
> <source-group-spec> <dest-group-spec>
>
> Rename a group within pool.
>
> Positional arguments
>
> <source-group-spec>    source group specification
>
> (example: [<pool-name>/]<group-name>)
>
> <dest-group-spec>        destination group specification
>
> (example: [<pool-name>/]<group-name>)
>
> Optional arguments
>
> -p [ –pool ] arg               source pool name
>
> –group arg                     source group name
>
> –dest-pool arg               destination pool name
>
> –dest-group arg             destination group name

但当前的实现并不支持跨存储池的 rename 重命名操作，也就是说，目前不支持在进行 rename 重命名操作过程中修改 group 所对应的存储池。

<br />

**RBD 支持删除所有未设置 protected mode 的快照**

rbd: allow remove all unprotected snapshots (https://github.com/ceph/ceph/pull/20608)

之前，如果用户为一个 rbd image 创建了多个 snapshots，只要这些 snapshots 中有一个被设置为 protected mode，则该 image 对应的所有 snapshots 均无法被删除，这种处理方式，在很多应用场景中都显得不够灵活。

在上面的提交中，社区对这一处理过程进行了完善，允许删除所有未被设置为 protected mode 的 snapshots。

<br /> 

**RBD children 命令支持指定 snap-id**

rbd: children list should support snapshot id optional (https://github.com/ceph/ceph/pull/20966)

在上面的 PR 中，社区为 rbd children 命令新增了 –snap-id 参数。rbd children 命令用于列出在 Snapshot Layering 中某一 Image 的所有 Child Images 。之前若要查询某一特定 Snapshot 的所有 Child Images 只能通过指定该 Snapshot 的名字来进行查询，在支持 –snap-id 参数后，也可以通过指定 Snapshot 的 snapshot ID 来进行查询。

<br />
<br />

# 统一存储层

**OSD 支持异步 Recovery**

osd: async recovery (https://github.com/ceph/ceph/pull/19811)

Ceph 社区后面的工作重点之一，就是对 Ceph 的性能进行提高。众所周知，Ceph 的处理性能一直是其被诟病的问题之一。现在社区每周都会举行 Ceph Performance Weekly，可见社区确实已经开始重视起 Ceph 的性能问题了。Youtube 上有 Ceph Performance Weekly 的会议录像，感兴趣的同学可以观看学习。

Ceph Recovery 故障恢复处理在整个 Ceph 框架体系中，是至关重要的一部分。但是由于处理机制的问题，导致 Ceph 在进行 Recovery 处理过程中，对客户端的 IO 请求处理性能低到了惨不忍睹的地步，如图是我们之前的测试结果：

![ceph osd recovery](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph_monthly/March/ceph-monthly-mar-1.png?raw=true)

在上面的提交中，社区为 Ceph 引入了 Async Recovery 处理的机制，即将 Recovery 的处理过程异步化。

之前我们也尝试通过引入 Async Recovery 来解决 Ceph Recovery 处理过程中的性能问题。之前我们尝试 Async Recovery 测试结果如图<span style="color:red;">(特别说明并非对该提交的测试结果)</span>：

![ceph osd async recovery](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph_monthly/March/ceph-monthly-mar-2.png?raw=true)

<br />

**MON 集中收集 Ceph 组件配置**

mon: centralized config (https://github.com/ceph/ceph/pull/20172)

在之前的 Ceph 月报中我们提到过，当前社区正在引入集中式的配置管理。现在，monitor 的 centralized config 特性已经被合并到了 master 主线分支，以后 monitor 组件将会作为 Ceph 集群的配置中心来中心化存放整个集群的配置信息，其他组件会从 monitor 中获取对应的配置信息，这无疑给集群的管理和维护工作带来了极大的提升。

<br />

**MON 保留配置变更历史，以支持 diff 和 rollback 操作**

mon: keep history of config changes; implement diff and rollback (https://github.com/ceph/ceph/pull/20760) <span style="color:red;">(未合并)</span>

针对上面提到的，被 merge 到 master 主线分支的 集中式配置管理 (Centralized Config) 功能，社区在这个 PR 中，为 monitor 组件新增了两个用于配置管理的 CLI 选项：

- config log name=num,type=CephInt,req=False

- config reset name=num,type=CephInt

其中，

config log 选项用来显示 config 的修改历史信息，可类比 git log

config reset 选项用来对修改后的 config 配置信息进行回滚，恢复到之前的配置状态

后面，社区可能还会添加

config diff 和 config revert 等命令选项

<br />
<br />

# 集群管理

**MGR 合并默认面板支持**

mgr/dashboard_v2: Initial submission of a web-based management UI (replacement for the existing dashboard) (https://github.com/ceph/ceph/pull/20103)

dashboard_v2 被合并到了 master 主线分支中。以后 Ceph 就有自己默认的管理面板了。

前一版本的 dashboard 是 read-only 的，更多的是作为一个当前 Ceph 集群运行时状态信息和性能参数指标的展示页面的。dashboard_v2 的设计目标则主要集中在为 Ceph 集群的系统管理员提供一个基于 Web 的 UI 管理界面。dashboard_v2 的 WebUI 是基于 Angular 和 TypeScript 实现的。

<br />
<br />

# 基础库

**vstart.sh 支持创建集群不启用面板**

vstart: fix option (due to quotes) and allow disabling dashboard (https://github.com/ceph/ceph/pull/20986)

在开发过程中，使用 vstart.sh 创建测试集群超级方便，但是默认创建 dashboard 还是挺烦人的。

在上面的 PR 中，社区为 vstart.sh 新增了 –without-dashboard 参数以支持在通过 vstart.sh 启动测试集群时通过指定该参数来禁用 dashboard。
