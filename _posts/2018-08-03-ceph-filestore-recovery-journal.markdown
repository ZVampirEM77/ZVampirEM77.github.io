---
layout: post
title: Ceph 开荒杂记(二) -- Ceph filestore journal 分区写坏修复过程
date: 2018-08-03 00:00:00 +0300
img: ceph/osd/filestore/journal/the_journal.jpg # Add image post (optional)
stickie: false
tags: [Ceph, osd, filestore] # add tag
---

# 环境准备

- 启动测试集群

```
$ MON=1 OSD=3 RGW=0 MDS=0 MGR=1 ../src/vstart.sh -n -x -d

$ ./bin/ceph -s
```

![ceph-s](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_1.png?raw=true)

<br />
<br />

- 准备测试数据

```
$ ./bin/ceph osd lspools

$ ./bin/ceph osd pool create test 64 64

$ ./bin/ceph osd lspools

$ ./bin/rbd create --size 1G test/test_image

$ ./bin/rbd -p test ls
```

![rbd_ls](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_3.png?raw=true)

<br />
<br />

# 实验

- 查看 ceph-osd 进程

```
$ ps aux |grep ceph-osd
```

![ps](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_2.png?raw=true)

<br />
<br />

- 写坏 journal

```
$ cd dev/osd.1

$ echo > journal
```

![write_journal](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_4.png?raw=true)

<br />
<br />

- 重启 osd.1 进程

```
$ kill -9 $OSD_PROC

$ ./bin/ceph-osd -i 1 -c ./ceph.conf

$ ps aux |grep ceph-osd
```

![ps](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_5.png?raw=true)

![ceph-s](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_8.png?raw=true)


可以看到，osd.1 启动失败了。查看 osd.1 的日志

![osd.1.log](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_6.png?raw=true)


可以看到，的确是因为 journal 分区坏掉了，导致 osd.1 挂掉了。

<br />
<br />

- 修复 journal

```
# 查看journal指向的磁盘，清空旧journal
# dd if=/dev/zero of=/dev/disk/by-partuuid/793b35bc-a0ba-4aae-ae62-4c63558e199c bs=1M count=100

$ ./bin/ceph-osd --mkjournal -i 1
```

其中，

*--mkjournal*

initialize a new journal

*-i*

set ID portion of my name，实际上就是 osd 的 ID

![mkjournal](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_7.png?raw=true)

<br />
<br />

- 重新启动 osd.1

```
$ ./bin/ceph-osd -i 1 -c ./ceph.conf
```

![ps](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_9.png?raw=true)

可以看到 osd.1 成功启动了。查看集群状态

![ceph-s](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/osd/filestore/journal/ceph_filestore_journal_10.png?raw=true)

可以看到，集群一切恢复正常。


# Reference

https://ceph.com/geen-categorie/ceph-recover-osds-after-ssd-journal-failure/
