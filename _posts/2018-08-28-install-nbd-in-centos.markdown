---
layout: post
title: 在 CentOS 中安装 nbd.ko 驱动模块
date: 2018-08-28 00:00:00 +0300
img: linux/install_nbd_ko/install_nbd_ko_1.jpg # Add image post (optional)
stickie: false
tags: [Linux, NBD] # add tag
---

# 编译过程

## 1. 查看当前系统和内核版本

```
$ cat /etc/centos-release
```

输出为：

CentOS Linux release 7.5.1804 (Core)

```
$ uname -r
```

输出为：

3.10.0-862.el7.x86_64

<br />
<br />

## 2. 安装新内核和内核源码

```
$ yum install kernel kernel-devel kernel-headers -y
```

<br />
<br />

## 3. 安装编译工具

```
$ yum groupinstall" Development Tools"

$ yum install  elfutils-libelf-devel
```

<br />
<br />

## 4. 准备源码

```
$ wget http://vault.centos.org/7.5.1804/updates/Source/SPackages/kernel-3.10.0-862.11.6.el7.src.rpm

// 在 ~/rpmbuild/SOURCES 目录下生成 linux-3.10.0-862.11.6.el7.tar.xz

$ rpm -ihv kernel-3.10.0-862.11.6.el7.src.rpm

$ cd ~/rpmbuild/SOURCES/

$ mkdir /usr/src/kernels/fornbd

$ tar -Jxf linux-3.10.0-514.6.1.el7.tar.xz -C /usr/src/kernels/fornbd/
```

<br />
<br />

Note:
在这一过程中可能会遇到

```
warning: user builder does not exist - using root
warning: group builder does not exist - using root
```

这样的告警，主要是因为当前系统环境中没有 builder 用户，创建即可

```
$ useradd builder
```

## 5. 编译

```
# cd /usr/src/kernels/fornbd/linux-3.10.0-862.11.6.el7/

# cp /usr/src/kernels/3.10.0-862.11.6.el7.x86_64/Module.symvers .

# cp /boot/config-3.10.0-862.11.6.el7.x86_64 .config

# make oldconfig

# make prepare

# make scripts

# make CONFIG_BLK_DEV_NBD=m M=drivers/block
```

### 编译错误 1

```
drivers/block/nbd.c: In function ‘__nbd_ioctl’:
drivers/block/nbd.c:619:19: error: ‘REQ_TYPE_SPECIAL’ undeclared (first use in this function)
   sreq.cmd_type = REQ_TYPE_SPECIAL;
                      ^
drivers/block/nbd.c:619:19: note: each undeclared identifier is reported only once for each function it appears in
make[2]: *** [drivers/block/nbd.o] Error 1
make[1]: *** [drivers/block] Error 2
make: *** [drivers] Error 2
```

这是一个已知的 bug 

https://bugs.centos.org/view.php?id=12248

查找定义位置

```
$ grep "REQ_TYPE_SPECIAL" --color -R -n .
```

输出为：

```
grep: ./configs: No such file or directory
./drivers/block/nbd.c:619:        sreq.cmd_type = REQ_TYPE_SPECIAL;
./drivers/block/paride/pd.c:445:    if (pd_req->cmd_type == REQ_TYPE_SPECIAL) {
./drivers/block/paride/pd.c:728:    rq->cmd_type = REQ_TYPE_SPECIAL;
./drivers/ide/ide-atapi.c:96:    rq->cmd_type = REQ_TYPE_SPECIAL;
./drivers/ide/ide-atapi.c:480:        if (rq->cmd_type == REQ_TYPE_SPECIAL) {
./drivers/ide/ide-cd.c:802:    case REQ_TYPE_SPECIAL:
./drivers/ide/ide-cd_ioctl.c:307:    rq->cmd_type = REQ_TYPE_SPECIAL;
./drivers/ide/ide-devsets.c:169:    rq->cmd_type = REQ_TYPE_SPECIAL;
./drivers/ide/ide-eh.c:150:    if (rq && rq->cmd_type == REQ_TYPE_SPECIAL &&
./drivers/ide/ide-floppy.c:100:    if (rq->cmd_type == REQ_TYPE_SPECIAL)
./drivers/ide/ide-floppy.c:249:        if (rq->cmd_type == REQ_TYPE_SPECIAL) {
./drivers/ide/ide-floppy.c:268:    case REQ_TYPE_SPECIAL:
./drivers/ide/ide-io.c:138:    u8 drv_req = (rq->cmd_type == REQ_TYPE_SPECIAL) && rq->rq_disk;
./drivers/ide/ide-io.c:356:        } else if (!rq->rq_disk && rq->cmd_type == REQ_TYPE_SPECIAL)
./drivers/ide/ide-ioctls.c:225:    rq->cmd_type = REQ_TYPE_SPECIAL;
./drivers/ide/ide-park.c:37:    rq->cmd_type = REQ_TYPE_SPECIAL;
./drivers/ide/ide-park.c:54:    rq->cmd_type = REQ_TYPE_SPECIAL;
./drivers/ide/ide-tape.c:579:    BUG_ON(!(rq->cmd_type == REQ_TYPE_SPECIAL ||
./drivers/ide/ide-tape.c:856:    rq->cmd_type = REQ_TYPE_SPECIAL;
./include/linux/blkdev.h:89:    REQ_TYPE_SPECIAL,        /* driver defined type */
```

实际上，可以看到，在 nbd.c 文件中已经引用了 blkdev.h 文件，查看 include/linux/blkdev.h 文件

```
enum rq_cmd_type_bits {
  REQ_TYPE_FS             = 1,
  REQ_TYPE_BLOCK_PC,
  REQ_TYPE_SENSE,
  REQ_TYPE_PM_SUSPEND,
  REQ_TYPE_PM_RESUME,
  REQ_TYPE_PM_SHUTDOWN,
#ifdef __GENKSYMS__
  REQ_TYPE_SPECIAL,
#else
  REQ_TYPE_DRV_PRIV,
#endif
...
}
```

对源码进行修改，修改后的内容为

```
enum rq_cmd_type_bits {
  REQ_TYPE_FS             = 1,
  REQ_TYPE_BLOCK_PC,
  REQ_TYPE_SENSE,
  REQ_TYPE_PM_SUSPEND,
  REQ_TYPE_PM_RESUME,
  REQ_TYPE_PM_SHUTDOWN,
  REQ_TYPE_SPECIAL = 7,
  REQ_TYPE_DRV_PRIV = 7,
...
}
```

或者是直接修改 drivers/block/nbd.c 中的代码 (推荐，因为 include/linux/blkdev.h 是内核源码中的通用头文件，修改的话可能会影响其他模块的使用。当然，因为我们只需要使用 nbd 驱动模块，所以对于我们来说影响是不存在的)

从 ./include/linux/blkdev.h 头文件中的 rq_cmd_type_bits 定义可知 REQ_TYPE_SPECIAL 实际上为 7，所以，可以直接在 drivers/block/nbd.c 中对代码进行修改，修改 

```
sreq.cmd_type = REQ_TYPE_SPECIAL; --> sreq.cmd_type = 7;
```

即可解决编译错误问题。

<br />

### 编译错误 2

另外，可能在编译过程中，执行

```
make CONFIG_BLK_DEV_NBD=m M=drivers/block
```

后会报出如下错误

```
make[1]: *** No rule to make target `tools/objtool/objtool', needed by `drivers/block/floppy.o'. Stop.
make: *** [_module_drivers/block] Error 2
```

此时，可以观察一下，是否是上面

```
make scripts
```

过程发生错误。果然，发现在 make scripts 过程中，报出了如下错误

```
Makefile:913: "Cannot use CONFIG_STACK_VALIDATION, please install libelf-dev or elfutils-libelf-devel"
```

产生这个错误的原因就是当前系统中没有安装 libelf-dev 或是 elfutils-libelf-devel。安装之后，从

```
make prepare 
```

重新开始执行。

<br />
<br />

## 6. 安装

```
# cp drivers/block/nbd.ko /usr/lib/modules/3.10.0-862.11.6.el7.x86_64/kernel/drivers/block/
# depmod
# modprobe nbd
# lsmod | grep nbd
nbd                    17603  0
```


