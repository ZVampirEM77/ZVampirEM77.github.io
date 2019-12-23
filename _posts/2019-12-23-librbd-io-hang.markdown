---
layout: post
title: 记一次 librbd io hang 问题调查
date: 2019-12-23 00:00:00 +0300
stickie: false
tags: [Ceph, librbd] # add tag
---

# 背景

今天客户的集群环境遇到了 librbd io hang 的问题。

具体的问题为，在客户的 OpenStack 控制节点，执行 rbd import 向 Ceph 集群中导入操作系统镜像时，会出现 io hang 的问题。

具体执行命令为

```bash
rbd import Centos7 volumes/Centos7 --id volumes
```

写入一段时间会，会出现 io hang，io hang 住时的 debug 信息显示为

```bash
2019-12-23 16:25:06.922229 7f2bfc3f8700 20 librbd::object_map::UpdateRequest: 0x7f2be8002150 handle_update_object_map: r=0
2019-12-23 16:25:06.922237 7f2bfc3f8700 20 librbd::object_map::UpdateRequest: 0x7f2be8002150 update_in_memory_object_map:
2019-12-23 16:25:06.922241 7f2bfc3f8700 20 librbd::object_map::Request: 0x7f2be8002150 should_complete: r=0
2019-12-23 16:25:06.922245 7f2bfc3f8700 20 librbd::ObjectMap: 0x7f2be8004cf0 handle_detained_aio_update: cell=0x7f2be8003640, r=0
2019-12-23 16:25:06.922248 7f2bfc3f8700 20 librbd::BlockGuard: 0x7f2be8004d90 release: block_start=24, block_end=25, pending_ops=0
2019-12-23 16:25:06.922254 7f2bfc3f8700 20 librbd::io::ObjectRequest: 0x7f2be8016d30 handle_pre_write_object_map_update: r=0
2019-12-23 16:25:06.922256 7f2bfc3f8700 20 librbd::io::ObjectRequest: 0x7f2be8016d30 write_object:
2019-12-23 16:25:06.924787 7f2bfc3f8700 20 librbd::object_map::UpdateRequest: 0x7f2be800a200 handle_update_object_map: r=0
2019-12-23 16:25:06.924814 7f2bfc3f8700 20 librbd::object_map::UpdateRequest: 0x7f2be800a200 update_in_memory_object_map:
2019-12-23 16:25:06.924823 7f2bfc3f8700 20 librbd::object_map::Request: 0x7f2be800a200 should_complete: r=0
2019-12-23 16:25:06.924832 7f2bfc3f8700 20 librbd::ObjectMap: 0x7f2be8004cf0 handle_detained_aio_update: cell=0x7f2be8003690, r=0
2019-12-23 16:25:06.924836 7f2bfc3f8700 20 librbd::BlockGuard: 0x7f2be8004d90 release: block_start=25, block_end=26, pending_ops=0
2019-12-23 16:25:06.924843 7f2bfc3f8700 20 librbd::io::ObjectRequest: 0x7f2be801af70 handle_pre_write_object_map_update: r=0
2019-12-23 16:25:06.924845 7f2bfc3f8700 20 librbd::io::ObjectRequest: 0x7f2be801af70 write_object:
```

也就是说，会卡在 write_object 这里。


但是在 Ceph 存储集群的 mon 节点上进行 import 时，是没有问题的。


# 解决过程记录

在刚开始拿到这个问题时，主要的怀疑点是是否是 OpenStack Controller 节点上的 librbd 的版本与我们后端存储集群的版本不一致，通过执行 

```bash
yum list installed | grep librbd
```

发现，控制节点上的 librbd 版本与后端存储集群的版本是一致的。

排除版本不一致的问题后，就需要具体看看 io hang 时，具体执行了什么操作。修改 Controller 节点上的配置文件，将 librbd 的日志等级调大，同时启用 admin socket 机制。 (这次调试过程中，admin socket 帮了大忙)

```bash
...
[client]
debug client = 20/20
debug rbd = 20/20
admin socket = /etc/ceph/ceph-client-rbd.asok
...
```

添加好上面的配置后，再次执行 import 操作，发现还是会出现 io hang 的问题。这时通过查看 admin socket 的帮助信息

```bash
ceph --admin-daemon /etc/ceph/ceph-client-rbd.asok help
```

发现可以通过执行 objecter_requests 命令来查看当前有哪些处理处于 in progress 的状态。于是查看之

```bash
ceph --admin-daemon /etc/ceph/ceph-client-rbd.asok objecter_requests
```

发现当前写入到某几个 osd 的请求正处于 in progress 状态。

反复执行 import 操作，反复观察，发现 io 都会 hang 在向某个固定节点的 osd 中写入数据的处理。由此基本可以断定，可能是当前控制节点到该节点的网络有问题。在控制节点上 ping 该节点，发现果然无法 ping 通，由此基本可以断定是网络问题导致的。

经过相关人员排查后，发现果然网关配置有问题，修复之后，import 操作即可以顺利完成数据的写入操作。
