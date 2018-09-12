title: java-c1000k-notes
date: 2018-09-10 23:52:54
categories: Java
tags: [Java, NIO, 并发, 网络编程]
---

# TL;DR

Java 实现 C1000K 需要对服务器进行一定的调整，同时也需要选择合理的编程方式。

个人对 Java 实现 C1000K 的学习笔记。

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



