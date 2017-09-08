date: 2014-11-09 00:00:00
title: Oracle ORA27102 问题
categories: Database
tags: [Oracle]
---

<style>
img {
	max-width:600px;
}
</style>

# 问题背景

在处理Oracle因为CI的持续连接引起Oracle进程用尽的问题之后，更新代码时忘记修改配置，导致了再次出现进程耗尽的问题，在处理时发现居然无法关闭数据库，同时sqlplus提示如下：

![ora27102][1]

`Out of memory`这个提示真是让人费解，内存感觉还是非常充足的样子。

<!-- more -->

所用的机器是两台阿里云的8核心16GB ECS服务器，Oracle数据库为`11G R2`。

这里是作为[`上一篇`][2]的补充笔记。

# 分析

## 查看内存需求

首先通过对应实例的ora文件查看内存需求：

```
cat /u01/app/oracle/product/11R2/dbs/spfileorcl.ora
```

由下图可知，1000进程需要大约`6.7GB`内存：

![no-enough-mem][3]

但是由`free -l`得知当前系统只有`4.6GB`内存可用，那么问题就变得简单了，暂时关闭其他进程，腾出内存空间，修改完成之后可以通过降低进程数减少内存占用。

# 解决

分析得知php-fpm当前占用了很多的内存，首先可以先停止php-fpm，之后完成减少Oracle进程数的操作。


[1]: https://blog.wislay.com/wp-content/uploads/2014/11/ora27102.png
[2]: http://www.liaoaoyang.com/articles/2014/11/09/ci-may-not-use-pconnect-with-oracle.html
[3]: https://blog.wislay.com/wp-content/uploads/2014/11/no-enough-mem.png


