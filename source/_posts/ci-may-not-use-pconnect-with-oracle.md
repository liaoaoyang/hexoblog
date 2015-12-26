date: 2014-11-09 00:00:00
title: CI可能不应该使用持久连接Oracle数据库
categories: PHP
tags: [Oracle,CI]
---

<style>
img {
	max-width:600px;
}
</style>

# 问题背景

看过CI框架用法应该会看到，在配置CI框架连接数据库时，默认会开启[`持久连接`][1]，即类似这样的配置`$db['test']['pconnect'] = TRUE;`，使用MySQL时会调用`mysql_pconnect`方法实现这一个功能，而`oci8`扩展恰巧也有类似的方法[`oci_pconnect`][2]:

![oci_pconnect][3]

方法的用处文档上说的很清楚：

> oci_pconnect() 创建一个到 Oracle 服务器的持久连接并登录。持久连接会被缓冲并在请求之间重复使用，可以降低每个页面加载的消耗。

那么按道理来说这样的功能应该是会提升处理能力的，但是问题在于，持久连接会增加Oracle的进程数，一旦进程数耗尽，那么新的连接请求可能会被拒绝，反而会使得处理能力下降。

今天遇到了这样的一个问题，当双机各自开启1024个php-fpm进程时，使用sqlplus连接数据库被拒绝，同时各种操作都被拒绝执行。

所用的机器是两台阿里云的8核心16GB ECS服务器，Oracle数据库为`11G R2`，PHP版本为`5.4`，CI版本为`2.2`。

# 表现

首先是一次压测结束后，发现通过sqlplus连接不上Oracle数据库，即首先`sqlplus /nolog`之后使用`conn`命令无法连接成功，发现报错信息为：

![max-proc-exceeded][4]

居然没有可用的进程了！

那么目前需要做的就是释放这些进程，直接`SHUTDOWN IMMEDIATE`也该是最简单粗暴也能达到目的的方法，但是目前根本不能按照原先的`sqlplus /nolog`之后使用`conn /as sysdba`登录，解决的方式可以直接在shell中使用`sqlplus / as sysdba`即可完成登录。

# 分析

## 增加进程数

登录完成之后进行`SHUTDOWN IMMEDIATE`以及`STARTUP`操作，重启之后修改可用进程数：

```
alter system set processes=1000 scope=spfile;
alter system set sessions=1105 scope=spfile;
alter system set transactions=1206 scope=spfile;

```

关于这几个数值的关系，参考了这篇[`博文`][5]，博文中作者提到这三者可以按照如下关系配置：

```
processes=x
sessions=x*1.1+5
transactions=sessions*1.1
```

使用spfile作为scope的参数原因是这几个参数并不能修改后立即生效，需要重启数据库之后才能生效。

修改完成后重启数据库，重新进行压测，`ps -ef | grep oracle | wc -l`发现进程数果然上来了，但是在压测结束后，发现再次出现了这一问题。那么这一问题并不单纯是进程不足的问题。

## 测试环境检查

Oracle进程数增加却被完全消耗，这个情况确实让人觉得奇怪。

检查CI的配置时，发现数据库配置文件中使用了pconnect属性，查阅oci_pconnect的文档，注意到如下一句话：

> 一个典型的 PHP 应用程序对于每个 Apache 子进程（或者 PHP FastCGI/CGI 进程）会有一个打开的持久连接到 Oracle 服务器

那么，目前一共有两台机器，每台机器开启了1024个php-fpm的进程，总计2048个，那么Oracle的进程数仅仅为1000个，自然是不够用的。

持久连接虽然减少了连接的成本，然后却使得没有进程可用，显然问题出在CI的持久连接上。

# 解决

解决的方式自然是关闭CI的持久连接配置，在设定`$db['test']['pconnect'] = FALSE;`之后，通过`ab`进行压力测试，在`10000`并发的情况下，所用的Oracle进程数长期维持在`30`个左右，基本上解决了这一问题。

# 结论

CI在使用持久连接时，可能需要考虑Oracle的可用进程数（设为`X`）以及php-fpm的个数（设为`Y`）之间的关系，当`X>Y`时，个人认为是可以考虑使用持久连接的。

否则，个人认为可以牺牲一部分性能来建立连接，使得Oracle维持有足够的可用进程数。



[1]: http://codeigniter.org.cn/user_guide/database/configuration.html
[2]: http://php.net/manual/zh/function.oci-pconnect.php
[3]: http://blog.wislay.com/wp-content/uploads/2014/11/oci_pconnect.jpg
[4]: http://blog.wislay.com/wp-content/uploads/2014/11/max-proc-exceeded.png
[5]: http://nimishgarg.blogspot.com/2012/05/ora-00020-maximum-number-of-processes.html
