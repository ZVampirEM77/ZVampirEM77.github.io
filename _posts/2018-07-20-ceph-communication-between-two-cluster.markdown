---
layout: post
title: Ceph 开荒杂记(一) -- mon addr 和 iptables
date: 2018-07-20 00:00:00 +0300
img: ceph/common/communication_between_two_ceph_1.jpg # Add image post (optional)
stickie: false
tags: [Ceph, iptables] # add tag
---

# 起因

今天下午，找了两台测试服务器，假设 IP 地址分别为 166.166.666.178(A 服务器) 和 166.166.666.179(B 服务器)，想分别在其中启动一个 Ceph 测试集群，来测试 Ceph RBD-Mirror 功能。

环境准备妥当，并且分别在两台服务器上对 Ceph 源码进行编译完成后，分别在两台服务器上执行

```
$ MON=1 OSD=3 RGW=0 MDS=0 MGR=1 ../src/vstart.sh -n -l -b -x
```

来通过 vstart 启动两个本地测试集群。一切看起来都是那么的顺理成章。


# iptables

两个测试集群正常启动后，因为后面要对 RBD-Mirror 功能进行测试，所以必须要保证两个节点之间是可以正常通信访问的。因此，在进行接下来的配置操作前，先对两个集群之间的网络是否互通进行测试。

先进行互 ping 

```
$ ping 166.166.666.179    // 在 A 服务器上 ping B 服务器

$ ping 166.166.666.178    // 在 B 服务器上 ping A 服务器
```

确认得到两台服务器之间的网络是通的、正常的。

然后，分别在两台服务器上，尝试通过 telnet 来访问对方服务器上的 Ceph 集群的 mon 服务，以确认两台服务器之间是可以彼此访问到对方上面的 Ceph 集群服务。假设 A 服务器上的 Ceph 集群的 mon 服务监听在 40031 端口上；B 服务器上的 Ceph 集群的 mon 服务监听在 40669 端口上。

```
$ telnet 166.166.666.179 40669    // 在 A 服务器上 telnet 访问 B 服务器上的 Ceph 集群的 mon 服务

$ telnet 166.166.666.178 40031    // 在 B 服务器上 telnet 访问 A 服务器上的 Ceph 集群的 mon 服务
```

结果发现两台服务器上均报出了 <span style="color:red;">telnet: connect to address IP地址: No route to host</span> 。

遇到 <span style="color:red;">telnet: connect to address IP地址: No route to host</span> 错误，最先想到的是防火墙的问题。

![firewall](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/common/communication_between_two_ceph_2.jpg?raw=true)

于是，分别在两台服务器上，通过 iptables 来分别把 mon 服务监听的端口给暴露出来。

```
$ iptables -A INPUT -p tcp --dport 40031 -j ACCEPT   // 在 A 服务器上暴露 40031 端口

$ iptables -A INPUT -p tcp --dport 40669 -j ACCEPT   // 在 B 服务器上暴露 40669 端口
```

配置执行成功后，再分别进行 telnet，发现还是会报出 <span style="color:red;">telnet: connect to address IP地址: No route to host</span> 错误。回过头，通过执行

```
$ iptables -L
```

来查看防火墙配置是否配置成功，在检查过程中，便发现了问题

![iptables -L](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/common/communication_between_two_ceph_3.png?raw=true)

可以看到，确实已经成功添加了暴露 40031 和 40669 端口的相关配置，但是这条配置是以 -A 即 append 的方式添加进去的，在这条配置前，是一条 <span style="color:red;">"REJECT" all </span> 的配置，因此，当有从 40031 或是 40669 端口进来的访问请求经过防火墙时，会先被这条 REJECT 规则给拒绝掉，也就相当于我们添加的防火墙规则并没有生效。

知道了问题的根源后，我们便开始着手解决，这一次通过使用 -I 选项来向防火墙中插入一条规则

```
$ iptables -I INPUT -p tcp --dport 40031 -j ACCEPT   // 在 A 服务器上暴露 40031 端口

$ iptables -A INPUT -p tcp --dport 40669 -j ACCEPT   // 在 B 服务器上暴露 40669 端口
```

通过 iptables 的帮助信息，可以看到 -I 默认是将所要配置的防火墙规则插入到 rule chain 的第一个位置。插入成功后，再有从 40031 或是 40669 端口进来的访问请求经过防火墙时，都会因为最先判断这条规则而被成功接收处理。


# mon addr

在通过上面的处理，成功解决了防火墙的问题后，分别在两个节点上，通过 telnet 来访问对方节点上的 mon 服务，发现仍然报错，会报出 <span style="color:red;">telnet: connect to address IP地址: Connection refused</span> 。

由报错信息分析，现在请求应该是已经被对方节点上的 mon 服务接收到了，但是因为一些原因，请求被拒绝掉了。于是，先开始排查 mon 服务的配置，查看 ceph.conf 中 mon 相关的配置信息时，即发现了问题。正如我上面说到的，在通过 vstart 启动测试集群时，执行的命令是

```
$ MON=1 OSD=3 RGW=0 MDS=0 MGR=1 ../src/vstart.sh -n <span style="color:red;">-l</span> -b -x
```

其中，指定了 -l 参数，因此 mon addr 配置项被配置为了 <span style="color:red;">127.0.0.1:40031/40669</span>。什么意思？即 mon 只处理访问这个地址和端口的请求信息。若是配置为 <span style="color:red;">127.0.0.1:40031/40669</span>，则 mon 将只会处理本地的访问请求。而这也正是问题所在。

在启动项中删除 -l 参数，重新启动测试集群。分别在两个节点上，尝试通过 telnet 来访问对方节点上的 mon 服务，发现已经可以正常访问了。

![telnet ok](https://github.com/ZVampirEM77/ZVampirEM77.github.io/blob/master/assets/img/ceph/common/communication_between_two_ceph_4.png?raw=true)
