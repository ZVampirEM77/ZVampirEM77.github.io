---
layout: post
title: Ceph 开荒杂记(三) -- radosgw-admin bucket stats 统计对象数与 list 的不一致
date: 2018-12-19 00:00:00 +0300
stickie: false
tags: [Ceph, radosgw] # add tag
---

# 背景

今天下午，测试同学在测试 UMStor 的过程中，发现了一个奇怪的现象：

通过

```
$ s3cmd ls s3://cc2
```

能够看到，存储桶 cc2 中有一个对象

<br />

![s3cmd ls](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/rgw/object-num-inconformity.png?raw=true)

<br />

但是通过

```
$ radosgw-admin bucket stats --bucket cc2
```

能够看到，存储桶中的 object num 为 5

<br />

![bucket stats](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/rgw/object-num-inconformity-2.png?raw=true)

<br />

为什么 bucket stats 返回的存储桶中的对象数和通过 s3cmd 看到的存储桶中的对象数不同呢？

<br />

想到的解决方法就是查看 bucket index 记录的存储桶中的对象列表来查看存储桶中到底有哪些对象。通过执行

```
$ radosgw-admin bucket stats --bucket cc2
```

从中获取到存储桶的 id 为 29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1

再执行

```
$ rados ls -p default.rgw.buckets.index |grep "29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1" | awk '{print "rados listomapkeys -p default.rgw.buckets.index "$1 }'|sh -x
```

来打印出存储桶的 bucket index 中记录的存储桶中所有对象，结果如下：

```
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.391
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.390
IMG_5741.jpg
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.178
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.106
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.16
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.221
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.231
…
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.241
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.254
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.510
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.93
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.483
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.317
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.144
_multipart_ubuntu-15.10-server-amd64.iso.2~myIwQyZP4UXG2pm-8ADCXLerI0TPpIw.meta
_multipart_ubuntu-15.10-server-amd64.iso.2~qgWv0FnHPbHcaerV-wj_G-U0Cwof52y.meta
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.419
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.458
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.253
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.69
…
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.414
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.40
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.206
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.108
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.25
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.292
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.137
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.1
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.2
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.3
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.4
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.meta
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.331
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.139
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.200
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.321
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.375
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.245
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.249
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.293
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.97
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.485
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.15
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.199
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.314
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.20
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.473
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.310
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.304
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.32
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.201
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.323
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.462
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.171
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.27
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.507
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.58
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.457
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.119
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.250
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.209
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.126
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.308
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.333
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.152
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.150
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.177
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.417
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.470
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.411
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.179
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.173
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.161
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.481
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.406
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.154
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.268
+ rados listomapkeys -p default.rgw.buckets.index .dir.29b4947b-d038-4511-bd36-9b2c9a2f45e7.271450.1.492
```

<br />

到这里，我们就看出了问题的原因所在，原来存储桶中，还存在

```
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.1
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.2
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.3
_multipart_ubuntu-14.04.3-server-amd64.iso.2~mfAysNmQgucoeAZ6CtjEvOiBOKkgbUh.4
```

这样几个 multipart 的分片数据对象。

通过和测试同事沟通，了解到，原来之前他们通过 multipart upload 上传过对象，但中途给 ctrl-C 了。

至此，找到了对象数据不一致的根本原因。
