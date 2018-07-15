---
layout: post
title: Ceph 开发者月报 - 一月篇
date: 2018-01-31 00:00:00 +0300
img: ceph_monthly/January/ceph-monthly-jan.png # Add image post (optional)
stickie: false
tags: [Ceph 月报] # add tag
---

# 对象存储

**存储桶支持使用多个存储池**

Multiple Data Pool Support for a Bucket (https://github.com/ceph/ceph/pull/19890)

即一个存储桶的多池支持。当前 Ceph 中，一个存储桶只能对应一个存储池，在实际生产使用场景中，用户可能会遇到如下痛点：

- 当存储池容量满了，进行容量扩容时，数据重平衡以及 Recovery 的过程会极大的影响集群的性能；

- 随着集群扩容时，OSD 数量的增加，相当于每个 OSD 上 PG 的数量被稀释了，这将可能会造成数据分布不均。为了解决这个问题，可以根据具体 OSD 的数据量通过 reweight 处理来调整不同 OSD 的权重；或者也可以增加 PG 数量。但上述处理过程同样会造成数据迁移，影响集群性能；

但是，若一个存储桶可以对应多个存储池，则上述痛点都可以被解决，即若当前存储池容量已被打满，不需要为该存储池新增 OSD 进行扩容 ，直接指定相应存储桶将数据存储到其他存储池上，避免了数据迁移对集群性能的影响。

<br />

**对象存储 HTTP Admin API 新增修改 bucket quota 的 API**

rgw: Admin API Support for bucket quota change (https://github.com/ceph/ceph/pull/18324)

<br />

**librgw支持多租户**

librgw: export multitenancy support (https://github.com/ceph/ceph/pull/19358)


<br />
<br />

# 块存储

**块存储支持给一致性组打快照**

librbd group snapshots (https://github.com/ceph/ceph/pull/11544)

Consistency Group，即一致性组，把一批存在公共操作的卷，在逻辑上划分为一个组，用户可以很方便的通过操作该组，来操作组中所有的卷，而不需要一个个的去操作这些卷，主要是出于数据保护和容灾的考虑。在这个 PR 中，开发者实现了 librbd Consistency Group 的支持，包括为 Consistency Group 打快照的功能。

<br /> 

**块存储支持镜像的深度拷贝**

rbd: add deep cp CLI method (https://github.com/ceph/ceph/pull/19996)


<br />
<br />


# 统一存储层

**OSD chunk级去重推进**

相关 PR

- osd,librados: add manifest, operations for chunked object (https://github.com/ceph/ceph/pull/15482)
- osd: flush operations for chunked objects (https://github.com/ceph/ceph/pull/19294)
- osd, librados: add a rados op (TIER_PROMOTE) (https://github.com/ceph/ceph/pull/19362)
- osd: refcount for manifest object (redirect, chunked) (https://github.com/ceph/ceph/pull/19935) <span style="color:red;">(未合并)</span>

在 http://pad.ceph.com/p/deduplication_how_do_we_store_chunk 和 http://pad.ceph.com/p/deduplication_how_dedup_manifists 中社区针对实现 OSD 支持 chunk 级对象数据去重功能进行了讨论。上面的 4 个 PR 即是开发者基于讨论的结果提交的代码实现。当前这个功能的实现已经基本趋于完整，针对 chunk 级去重的数据上、下行处理也基本包含于上面的提交中，可能还有一些 bug 和细节有待完善。

<br />

**librados支持异步处理接口**

librados add async interface for use with Networking TS (https://github.com/ceph/ceph/pull/19054)

以 C++ Networking 技术规范为标准，开发者为 librados 添加了异步处理接口，当前这套接口只支持 async_read()，async_write()，以及 async_operate() (当然 async_operate() 当前也只支持 read 和 write 两种操作)。具体接口的使用方法可以参照 PR 中对应提交的测试用例中的使用方法。详见

https://github.com/ceph/ceph/pull/19054/files#diff-b5fd6691c22ab2c09a283b8bdda0e9ec

PS：
《Working Draft, C++ Extensions for Networking》，即 C++ Networking 技术规范草稿
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4711.pdf

<br />

**scrub 支持抢占式的调度**

osd/PG: allow scrub preemption (https://github.com/ceph/ceph/pull/18971)

之前，当 OSD 正在进行 scrub 处理时，若此时有写请求进来，OSD 会相应的中断 scrub 处理，转而处理客户端发来的写请求。这会导致一个问题，若客户端的写请求很频繁，则会导致 OSD 迟迟无法完成 scrub 处理，这会对数据一致性的保证产生影响。sage 在这个 PR 中引入了配置项 osd_scrub_max_preemptions 用于配置在被客户端的 IO 请求中断多少次之后，scrub 处理线程会进行抢占，优先进行 scrub 处理。

<br />

**OSD异步恢复推进**

OSD Async Recovery (https://github.com/ceph/ceph/pull/19811) <span style="color:red;">(未合并)</span>

Ceph OSD Recovery 的性能问题一直是 Ceph 被人所诟病的缺陷之一。在宕掉的 OSD 重启进行 Recovery 处理的过程中，Ceph 对客户端的 IO 响应会出现数量级上的下降。而对 Ceph 的性能进行优化是社区后面的开发重心之一，因此 Recovery 的性能问题是必须要解决的。社区在这个 PR 中提交了 Async Recovery 功能，实现了在保证满足存储池最小副本数的情况下，将 Recovery 处理过程异步化，从而大大提升 OSD Recovery 处理过程中，Ceph 对客户端的 IO 响应性能。

<br />

**引入集中式的配置管理**

mon: centralized config (https://github.com/ceph/ceph/pull/20172) <span style="color:red;">(未合并)</span>

当前 Ceph 主要是通过配置文件 ceph.conf 来实现对集群各组件的配置管理，这无疑给系统的管理和维护，尤其是分布式系统，带来极大的难度和负担。因此，社区当前正在做的一件事情，就是为 Monitor 组件赋予配置中心的属性，即以后 Ceph 将会使用 Monitor 来中心化存放整个集群的配置信息，其他组件会从 Monitor 中获取对应的配置信息。sage 在这个 PR 中即实现了 Monitor 配置中心的特性。

<br />
<br />

# 集群管理

**内置面板新增对象存储页面**

mgr/dashboard: RGW page (https://github.com/ceph/ceph/pull/19512)

![dashboard rgw page](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph_monthly/January/ceph-monthly-jan-1.png?raw=true)

<br />
<br />

# 基础库

新增对 Google Brotli 压缩算法的支持

compressor: Add Brotli Compressor (https://github.com/ceph/ceph/pull/19549)
