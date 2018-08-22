---
layout: post
title: 为本地系统新增 yum 源
date: 2018-08-23 00:00:00 +0300
img: linux/yum_update/yum_update_1.png # Add image post (optional)
stickie: false
tags: [Linux] # add tag
---

# 新增 repo 配置文件

repo 配置文件中会保存有对应 yum 源的基本配置信息。例如，在 /etc/yum.repos.d 目录下，新增 emmm.repo 配置文件

![add_repo_file](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/linux/yum_update/yum_update_2.png?raw=true)

<br />
<br />

# 清理当前 yum 缓存

```
$ yum clean all
```

<br />
<br />

# 重新创建 yum 缓存信息

```
$ yum makecache
```

<br />
<br />

# 列出当前系统中所有可用 yum 源

```
$ yum repolist
```

![repolist](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/linux/yum_update/yum_update_3.png?raw=true)

<br />
<br />

# 列出当前系统中某一个 yum 源中所有可用的安装包

```
$ yum --disablerepo="*" --enablerepo="remi" list available
```

![list_available](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/linux/yum_update/yum_update_4.png?raw=true)
