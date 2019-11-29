---
layout: post
title: Linux 磁盘分区管理工具 -- gdisk 
date: 2019-11-29 00:00:00 +0300
stickie: false
tags: [Linux, gdisk] # add tag
---

nux 下的磁盘分区管理工具，大家可能第一个想到的就是 fdisk，但是 fdisk 的历史太“悠久”了，只能支持 MBR(Master Boot Record) 分区，并不支持 GPT (GUID Partition Table) 分区，无法操作超过 2T 的磁盘，因此现在比较常用的是 gdisk 磁盘分区管理工具。

# 使用方法

首先，以系统中的磁盘 /dev/sdg 为例

```bash
# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda             8:0    0 278.9G  0 disk
├─sda1          8:1    0   200M  0 part /boot
└─sda2          8:2    0 278.7G  0 part
  ├─umos-root 253:0    0   200G  0 lvm  /
  └─umos-var  253:1    0  78.7G  0 lvm  /var
sdb             8:16   0 278.9G  0 disk
sdc             8:32   0 278.9G  0 disk
sdd             8:48   0 278.9G  0 disk
sde             8:64   0 278.9G  0 disk
sdf             8:80   0 476.4G  0 disk
sdg             8:96   0 476.4G  0 disk
```

进入针对磁盘 /dev/sdg 的交互模式

```bash
# gdisk /dev/sdg
GPT fdisk (gdisk) version 0.8.10

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help):
```

## 查看帮助信息

```bash
gdisk /dev/sdg
GPT fdisk (gdisk) version 0.8.10

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): ?
b    back up GPT data to a file
c    change a partition's name
d    delete a partition
i    show detailed information on a partition
l    list known partition types
n    add a new partition
o    create a new empty GUID partition table (GPT)
p    print the partition table
q    quit without saving changes
r    recovery and transformation options (experts only)
s    sort partitions
t    change a partition's type code
v    verify disk
w    write table to disk and exit
x    extra functionality (experts only)
?    print this menu
```


## 创建分区

创建分区，打算将磁盘 /dev/sdg 分为两个分区，第一个分区 50GB。通过 n 命令来创建分区，需要填入:

+ 分区 ID
+ 起始扇区
+ 结束扇区

起始扇区可以直接回车采用默认值，描述结束扇区时，可以直接用 +50G 这种 humanable-read 的方式来定义

```bash
Command (? for help): n
Partition number (1-128, default 1): 1
First sector (34-999030750, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-999030750, default = 999030750) or {+-}size{KMGTP}: +50G
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300):
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sdg: 999030784 sectors, 476.4 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 4B1D34BD-ACDC-40E8-B030-024E3BE24918
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 999030750
Partitions will be aligned on 2048-sector boundaries
Total free space is 894173117 sectors (426.4 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048       104859647   50.0 GiB    8300  Linux filesystem
```


可以看到我们已经创建好了第一个分区，接下来，把剩下的磁盘空间划分为第二个分区

```bash
Command (? for help): n
Partition number (2-128, default 2): 2
First sector (34-999030750, default = 104859648) or {+-}size{KMGTP}:
Last sector (104859648-999030750, default = 999030750) or {+-}size{KMGTP}:
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300):
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sdg: 999030784 sectors, 476.4 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 4B1D34BD-ACDC-40E8-B030-024E3BE24918
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 999030750
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048       104859647   50.0 GiB    8300  Linux filesystem
   2       104859648       999030750   426.4 GiB   8300  Linux filesystem

Command (? for help):
```

可以看到，我们已经将磁盘 /dev/sdg 划分为两个分区，且第一个分区的大小为 50GB。


## 保存磁盘分区信息

在上面的操作中，我们已经对磁盘进行了分区操作，在分区表正式生效前，需要对分区信息进行保存

```bash
Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sdg.
The operation has completed successfully.
```

这样，分区表正式生效，我们已经成功对磁盘 /dev/sdg 进行分区。查看下分区后的结果


```bash
# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda             8:0    0 278.9G  0 disk
├─sda1          8:1    0   200M  0 part /boot
└─sda2          8:2    0 278.7G  0 part
  ├─umos-root 253:0    0   200G  0 lvm  /
  └─umos-var  253:1    0  78.7G  0 lvm  /var
sdb             8:16   0 278.9G  0 disk
sdc             8:32   0 278.9G  0 disk
sdd             8:48   0 278.9G  0 disk
sde             8:64   0 278.9G  0 disk
sdf             8:80   0 476.4G  0 disk
sdg             8:96   0 476.4G  0 disk
├─sdg1          8:97   0    50G  0 part
└─sdg2          8:98   0 426.4G  0 part
```


## 删除磁盘分区信息

使用命令 d 通过磁盘分区 ID 来删除对应的磁盘分区，首先我们先删除分区 ID 为 2 的磁盘分区

```bash
# gdisk /dev/sdg
GPT fdisk (gdisk) version 0.8.10

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): p
Disk /dev/sdg: 999030784 sectors, 476.4 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 4B1D34BD-ACDC-40E8-B030-024E3BE24918
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 999030750
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048       104859647   50.0 GiB    8300  Linux filesystem
   2       104859648       999030750   426.4 GiB   8300  Linux filesystem

Command (? for help): d
Partition number (1-2): 2

Command (? for help): p
Disk /dev/sdg: 999030784 sectors, 476.4 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 4B1D34BD-ACDC-40E8-B030-024E3BE24918
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 999030750
Partitions will be aligned on 2048-sector boundaries
Total free space is 894173117 sectors (426.4 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048       104859647   50.0 GiB    8300  Linux filesystem
```

可以看到磁盘的分区 ID 为 2 的分区已经被删除了。删除分区 ID 1，以擦除磁盘的所有分区

```bash
Command (? for help): d
Using 1

Command (? for help): p
Disk /dev/sdg: 999030784 sectors, 476.4 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 4B1D34BD-ACDC-40E8-B030-024E3BE24918
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 999030750
Partitions will be aligned on 2048-sector boundaries
Total free space is 999030717 sectors (476.4 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
```

至此，我们已经删除了磁盘的所有分区信息，保存并退出

```bash
Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sdg.
The operation has completed successfully.
# lsblk
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda             8:0    0 278.9G  0 disk
├─sda1          8:1    0   200M  0 part /boot
└─sda2          8:2    0 278.7G  0 part
  ├─umos-root 253:0    0   200G  0 lvm  /
  └─umos-var  253:1    0  78.7G  0 lvm  /var
sdb             8:16   0 278.9G  0 disk
sdc             8:32   0 278.9G  0 disk
sdd             8:48   0 278.9G  0 disk
sde             8:64   0 278.9G  0 disk
sdf             8:80   0 476.4G  0 disk
sdg             8:96   0 476.4G  0 disk
```

可以看到磁盘 /dev/sdg 的所有分区都已经被擦除了。
