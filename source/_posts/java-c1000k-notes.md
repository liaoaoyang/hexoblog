<!--
title: Java C1000K 笔记
date: 2018-09-10 23:52:54
categories: Java
tags: [Java, NIO, 并发, 网络编程]
---
-->

# TL;DR

Java 实现 C1000K 需要对服务器进行一定的调整，同时也需要选择合理的编程方式。

个人对 Java 实现 C1000K 的学习笔记。虽然早已不是难事，终须自己实现一遍。

<!-- more -->
<!-- java-c1000k-notes -->

# 前提

## C1000K or 1000K QPS？

首先需要说明，C1000K 并不指的是单机 1000W QPS。

并发的连接数不全是活跃连接。

## 服务器

8核16G ECS，操作系统CentOS 7。

# 配置

## 内核参数

### 文件

并发百万连接早已不是难事，但是服务器默认的一些配置仍然需要配置，默认情况下的配置并不足以支撑这一需求。

Linux 一切皆文件，Socket 连接也是文件。

内核对可以打开的文件数做了限制，默认较小，肯定无法达到百万量级。不调整就使用，会提示 `Too many open files`，这个问题在 ES 的使用过程中很容易遇到，同学启动集群时不做任何内核参数配置，索引增多之后无法建立连接查询ES，会在错误日志中看到相关的错误信息。

日常使用 `ulimit` 进行设置，在内核限制的情况下，即便设置成功页只对当前会话终端有效。

在 `/etc/sysctl.conf` 中配置文件打开的内核参数，在 `/etc/security/limits.conf` 中配置进程可以打开的文件参数。

主要限制文件打开数的有如下内核参数：

+ fs.file-max
+ fs.nr_open

关于二者的定义可以在[内核文档](https://www.kernel.org/doc/Documentation/sysctl/fs.txt)上查到：

```
==============================================================

file-max & file-nr:

The value in file-max denotes the maximum number of file-
handles that the Linux kernel will allocate. When you get lots
of error messages about running out of file handles, you might
want to increase this limit.

Historically,the kernel was able to allocate file handles
dynamically, but not to free them again. The three values in
file-nr denote the number of allocated file handles, the number
of allocated but unused file handles, and the maximum number of
file handles. Linux 2.6 always reports 0 as the number of free
file handles -- this is not an error, it just means that the
number of allocated file handles exactly matches the number of
used file handles.

Attempts to allocate more file descriptors than file-max are
reported with printk, look for "VFS: file-max limit <number>
reached".
==============================================================

nr_open:

This denotes the maximum number of file-handles a process can
allocate. Default value is 1024*1024 (1048576) which should be
enough for most machines. Actual limit depends on RLIMIT_NOFILE
resource limit.

==============================================================
```

二者的定义是 `fs.file-max` 决定了内核可以打开的文件数量(不同于文件描述符，这里指的是指向实际文件结构的对象，[参见](http://www.linuxvox.com/post/what-are-file-max-and-file-nr-linux-kernel-parameters/) )，`fs.nr_open` 决定了单个进程可以打开的最大文件数量。但是一般对应文件对象的都具有一个文件描述符，所以姑且可以认为二者数目相等。

再看 ECS 默认的 `/etc/security/limits.conf`:

```
root soft nofile 65535
root hard nofile 65535
* soft nofile 65535
* hard nofile 65535
```

soft 和 hard 的区别是 hard 值表示是参数的最大值，soft 值为设定值，在 hard 值的范围以内可以随意修改。

此处需要注意，如果 `/etc/security/limits.conf` 设定的值大于 `fs.nr_open` 会引起无法登录服务器的问题，一旦修改错误，如果还没有断开连接，立即调低当前的值，使之小于 `fs.nr_open`，否则，只能进入单用户模式恢复了。

综上，`fs.file-max` >= `fs.nr_open` >= `configs in /etc/security/limits.conf`。

服务端配置如上参数，压测客户端也需要配置，否则无法模拟出大量连接。

### 网络

#### 服务端

Echo Server 使用 TCP 连接，服务端需要先 bind 一个端口，之后 accept 新连接。

TCP 三次握手无需多言：

1) 客户端发出 `SYN_(a)`
2) 服务端收到 `SYN_(a)`，服务端返回 `SYN_(b) + ACK(a + 1)`
3) 客户端收到 `SYN_(b) + ACK(a + 1)`，客户端返回 `ACK(b + 1)`

在步骤`1`之后本次网络连接就是半连接状态，会进入一个队列，这个队列的大小由内核参数 `net.ipv4.tcp_max_syn_backlog` 决定。

在步骤`3`之后，会将连接放入Accept队列，队列的大小由内核参数 `net.core.somaxconn` 决定。

这两个队列至关重要，只有调用 `accept()` 方法之后，整个连接才是可以收发数据的状态，如果长时间不调用 `accept()`，客户端已经认为连接建立，发送数据会出现 `client fooling` 问题，长时间得不到回应会进入重试，一段时间过后客户端会主动发 `FIN` 断开连接，导致连接不成功。

以上问题的详情可以阅读博文 [《关于TCP 半连接队列和全连接队列》](http://jm.taobao.org/2017/05/25/525-1/) 了解更多。

回到 Echo Server 上，由于 `accept()` 操作不能马上完成，需要一定时间，在突发请求，或者压测的情况下瞬间建立大量连接，会导致队列拥塞，最后仍然会引起无法建立连接的问题。所以，适度提高队列大小有助于 Echo Server 以及实际网络服务器的开发。

```
# 最大值，原因参见 https://stackoverflow.com/questions/23862410/invalid-argument-setting-key-net-core-somaxconn
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
```

`net.core.somaxconn` 这个参数还会影响应用代码中 `backlog` 的取值，取值为 `min(net.core.somaxconn, backlog)`，这部分在服务端编码中再细致说明。

如果开启了 `iptables`，还需要注意 `net.*.nf_conntrack_max` 参数需要超过 1000K。

#### 客户端

作为压测客户端，需要注意的是，TCP请求可以看做一个四元组：

`客户端IP-客户端端口-服务端IP-服务端口`

想要获得大量的客户端连接，首先就需要足够多的端口。

端口范围的内核参数 `net.ipv4.ip_local_port_range` 默认值范围不大，大约在30000个左右，我们可以将其放宽到保留端口附近的范围，这样，就能产生超过60000个客户端连接了。

# 开发

## 客户端

考虑到开发的难度以及趣味性，选择 Go 进行开发（代码见[GitHub](https://github.com/liaoaoyang/LearningJava/blob/master/scripts/simple_nio_test_echo_client.go)）。

核心内容如下：

+ 每个请求产生一个协程，模拟一个客户端
+ 并发程度通过 channel 控制，由于 channel 可以阻塞住操作，可以通过产生与并发度相同大小的队列，当一个请求完成时读取一个数据，起到控制并发度的效果
+ 同样通过 channel 完成请求数的计数操作

考虑到单网卡难以模拟出百万级别连接（单网卡对应服务端端口只能使用约60000个端口），可以使用 docker 模拟出 16+ 客户端访问 `docker0` 设备上侦听端口的服务端程序，或者选择服务端开启 16 个以上端口。

## 服务端

服务端不会主动断开连接，TIME-WAIT 问题暂且不用处理。

通过 BIO + 多线程模式在客户端数量较少时没有问题，Java 目前了解到一个用户线程对应一个内核线程，客户端数目多时，为了降低系统消耗，考虑使用 NIO 实现（代码见[GitHub](https://github.com/liaoaoyang/LearningJava/blob/master/src/main/java/co/iay/learn/learningjava/nio/SimpleNIOEchoServerMT.java)）。

NIO 的核心就是一个线程管理多个连接，通过 Selector 实现对多个连接读写事件的监听，在读写时间触发时处理 IO 操作。

选用 Java NIO 实现 C1000K 服务器，实际上是 Reactor 模式的具体实现。

为了提高处理速度，考虑将 accept 操作与 IO 操作分开。

IO 操作即便是在非阻塞的 Channel 里也是占用 CPU 时间的，为了充分利用多核心，可以考虑将线程数调成CPU个数加一个。

# 效果

服务端选择启动多个端口：

```
java -jar -Dbacklog=40000 -Dhostname=192.168.16.1 -Dport=`seq 30001 30020|tr "\n" ","|sed 's/,$//'` SimpleNIOEchoServerMT.jar
```

客户端选择压测多个端口，并使用长连接，每 30+rand(0,30) 秒发送一个数据包，平均每秒活跃连接数30000左右：

```
go run simple_nio_test_echo_client.go -h=192.168.16.1 -P=`seq 30001 30020|tr "\n" ","|sed 's/,$//'` -i 30 -r 10 -c 60000
```

使用 ss 查看：

```
➜  ~ ss -s
Total: 1153054 (kernel 1153152)
TCP:   1174832 (estab 1055390, closed 92005, orphaned 22030, synrecv 0, timewait 0/0), ports 0

Transport Total     IP        IPv6
*         1153152   -         -
RAW       0         0         0
UDP       9         8         1
TCP       1082827   1082827   0
INET      1082836   1082835   1
FRAG      0         0         0
```

以上是 NIO 实现 C1000K 服务器的摘要。

# 参考

+ [强烈推荐余锋老师的博文](http://blog.yufeng.info/archives/category/network/)
+ [不要在linux上启用net.ipv4.tcp_tw_recycle参数](https://www.cnxct.com/coping-with-the-tcp-time_wait-state-on-busy-linux-servers-in-chinese-and-dont-enable-tcp_tw_recycle/)
+ [关于TCP 半连接队列和全连接队列](http://jm.taobao.org/2017/05/25/525-1/)
+ [构建C1000K的服务器(1)](http://www.ideawu.net/blog/archives/740.html)
+ [iptables 的 conntrack 连接跟踪模块](https://awen.me/post/59062.html)
+ [Node版单机100w连接（C1000K）是如何达成的](https://www.jianshu.com/p/e0b52dc702d6)


