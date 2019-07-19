---
layout: post
title: 为什么 Ceph RGW radosgw-admin gc process 不生效
date: 2019-07-19 00:00:00 +0300
stickie: false
tags: [Ceph, RGW, GC] # add tag
---

#背景

用过 Ceph 对象存储的同学应该都支持，在 RGW 中，如果一个对象的数据大小大于 4M，则在 Ceph 内部处理过程中，会将对象进行条带化存储，即将对象分为一个个 4M 大小的条带分别进行存储，在 Ceph 中，我们将其称为 rados object，这也正是 Ceph 的核心实现思想是对象存储中的 "对象" 的真正含义。

与此同时，对于 rados object 对象来说，实际上是分为 head object 和 tail object 两种角色的：

- head object 中包含了对象的元数据信息，以及对象实际数据的前 4M 大小的数据；
- tail object 中则纯粹存储的是对象的数据信息；

在 Ceph RGW 中，当删除一个对象时，实际上是立即删除了该对象对应的 rados objects 中的 head object，剩下的其对应的 tail objects 则是通过 GC ，即垃圾回收机制，来在固定的时间点，统一进行实际的删除处理的。

RGW 中提供了两个 gc 相关的命令行工具

```
radosgw-admin gc list [--include-all]
radosgw-admin gc process
```

前者用于列出当前所有正在进行以及待进行垃圾回收处理的 rados object 对象信息，后者用于手动进行垃圾回收处理。

但用过 radosgw-admin gc process 命令的同学应该都有体会，我们预想的是，执行该命令后，所有通过 gc list 列出的待清理的对象数据应该都会被删除清理，但现实往往会给你一巴掌，我们发现在执行了 radosgw-admin gc process 之后，貌似什么都没有发生，世界还是那个世界，集群还是那个集群，存储空间用量还是那个用量。

Why?


# 这是为什么呢？

通过阅读 RGW GC 相关的代码，我们就能找到答案。

首先，让我们仔细观察 radosgw-admin gc list --include-all 返回的信息

```
[
    {
        "tag": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.1090\u0000",
        "time": "2019-07-19 05:04:06.0.015255s",
        "objs": [
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__multipart_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.1",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.1_1",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.1_2",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.1_3",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__multipart_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.2",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.2_1",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.2_2",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.2_3",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__multipart_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.3",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.3_1",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.3_2",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.3_3",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__multipart_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.4",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.4_1",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.4_2",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_20M.2~iMG95JcnPCanqV1hDEAtT3QUmgLWGGm.4_3",
                "key": "",
                "instance": ""
            }
        ]
    },
    {
        "tag": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.1078\u0000",
        "time": "2019-07-19 04:46:05.0.837063s",
        "objs": [
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__multipart_16M.2~HSrK9yhsKrcYl01AFyG-JkWOitZhYpT.1",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_16M.2~HSrK9yhsKrcYl01AFyG-JkWOitZhYpT.1_1",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_16M.2~HSrK9yhsKrcYl01AFyG-JkWOitZhYpT.1_2",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__shadow_16M.2~HSrK9yhsKrcYl01AFyG-JkWOitZhYpT.1_3",
                "key": "",
                "instance": ""
            },
            {
                "pool": "default.rgw.buckets.data",
                "oid": "5eed88ba-8354-4417-b921-6f74fa3c85bf.4120.2__multipart_16M.2~HSrK9yhsKrcYl01AFyG-JkWOitZhYpT.2",
                "key": "",
                "instance": ""
            }
        ]
    }
]
```

有没有发现什么我们之前忽略的事情？是的，就是返回信息中，除了包含所有待进行垃圾回收的对象列表信息外，还包含 time 字段。而且上面两个对象对应的 rados object 待删除列表中， time 字段的值还是不同的。

```
"time": "2019-07-19 05:04:06.0.015255s",
"time": "2019-07-19 04:46:05.0.837063s",
```

实际上，关键就在这里。

上面 time 字段的含义是，rados object 对象真正过期的时间。这也正是当前 RGW GC 垃圾回收机制的设计思想之一。在 Ceph RGW 中存在 rgw_gc_obj_min_wait 配置项，正是该配置项决定了上面的 time 字段的值。该配置项用于控制对象在能被 GC 删除之前需要等待的最小时间，这主要是考虑到，当删除一个对象时，可能会有其他服务正在读取对应的象数据信息，为了保证不对其他服务的操作处理造成影响，因此在删除对象数据后，需要等待一段时间，才能真正的对对象数据进行删除。

有了上面对 time 字段的说明，相信大家已经能够联想到 radosgw-admin gc process 不生效的原因了。

是的，实际上并不是 radosgw-admin gc process 不生效，而是执行该命令时，只能删除已经过期了的，但是还没有到 gc 处理周期进行删除的对象数据，即当前时间已经超过了上面 time 字段执行的时间，但还没有被删除的 tail object 数据。
