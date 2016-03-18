title: 一组PHP7在Nginx与Apache下的测试数据
date: 2016-03-18 01:46:29
categories: 系统
tags: [PHP,PHP7,Apache,Nginx]
---

# 概述

PHP7作为迄今为止性能最强的PHP版本，希望通过一些测试来了解它在Nginx与Apache下的表现。数据仅供参考。

由于个人水平有限，对测试的理解可能有所偏差，如有错误麻烦指出。

# 运行环境

测试环境一共分为两台机器，机器A用作PHP运行环境，机器B作为压测机。A与B的CPU均为*12 x Xeon E5* + *16GB RAM*。

两台机器位于同一网段。

测试工具使用`ab`。

为了便于测试，调整了内核参数：

```
net.ipv4.tcp_syncookies = 0
net.ipv4.tcp_tw_reuse = 1
```

测试使用的软件版本为：

+ Apache 2.2
+ Nginx 1.2.7
+ PHP 7.0.4

Apache在`httpd-mpm.conf`中的配置为：

```
ServerLimit 512

<IfModule mpm_prefork_module>
    StartServers          25
    MinSpareServers       25
    MaxSpareServers       25
    MaxClients            512
    MaxRequestsPerChild   10000
</IfModule>
```

由于`Apache+modphp7`与`Nginx+PHP-FPM`二者的工作模式差别较大，想要均衡的配置出一个均衡的测试比较环境很困难，从简单出发，考虑到Apache在上述配置下最大会fork出512个进程处理请求，将PHP-FPM最大进程数也设成512个，即:

```
pm = dynamic
pm.max_children = 512
pm.start_servers = 25
pm.max_spare_servers = 25
pm.max_requests = 10000
```

# 测试方法

测试主要针对三种操作进行：`输出Hello World`，`Redis KV读取操作`，`输出phpinfo()`。

选择`Redis KV读取操作`的目的主要想测试下这一常用扩展的在Apache与Nginx作为WebServer环境下的表现。

测试分为2组，分别在25并发以及512并发下进行，测试时长为1min，同时记录服务器的load情况。

关注的数据为：

+ 最小响应时间
+ 平均响应时间
+ 平均每秒请求数
+ load
+ 相关进程数

# 测试结果

测试从`性能指标`以及`系统负载`两个方面进行观察。

## 性能指标

### 最小响应时间

![min_response_time.png](http://blog.wislay.com/wp-content/uploads/2016/03/min_response_time.png)

针对最小响应时间，并发数高低对最小响应时间没有显著的影响。在测试初期系统负载较低，测试数据应该表现最佳的阶段，推断二者在此阶段差距不大。

### 平均每秒请求数

![requests_per_sec.png](http://blog.wislay.com/wp-content/uploads/2016/03/requests_per_sec.png)

针对平均每秒请求数，`Nginx+PHP-FPM`的组合处理能力稍低于`Apache+modphp7`的组合，考虑到Nginx与PHP-FPM之间有网络通信(测试中采用了tcp socket)的开销，这个结果是可以接受的。高并发下考虑系统开销较大，请求处理能力会稍有下降。

### 平均响应时间

![mean_response_time.png](http://blog.wislay.com/wp-content/uploads/2016/03/mean_response_time.png)

针对平均响应时间，高并发下同样由于系统开销较大的缘故，平均的请求处理响应时间升高也是符合预期的。


## 系统负载

从性能指标上看，`Nginx+PHP-FPM`的组合与`Apache+modphp7`的组合在表现上没有巨大的差距，但是在系统负载上，`Nginx+PHP-FPM`的配置在高并发的情况下远优于`Apache+modphp7`的组合。
在512并发时，在测试Hello world时，系统load的表现就有巨大的差距：

![load_hello_world.png](http://blog.wislay.com/wp-content/uploads/2016/03/load_hello_world.png)

同样的情况还出现在512并发之下对`Redis KV操作`的测试case之下：

![load_redis_kv.png](http://blog.wislay.com/wp-content/uploads/2016/03/load_redis_kv.png)

即便是在低并发（25并发）下，Redis KV此类需要操作网络资源的操作，`Apache+modphp7`在系统负载上的表现也差于`Nginx+PHP-FPM`：

![load_redis_kv_c25.png](http://blog.wislay.com/wp-content/uploads/2016/03/load_redis_kv_c25.png)

对于这类需要操作I/O的操作来说，`Apache+modphp7`的组合表现差于使用了异步I/O的Nginx与PHP-FPM的组合是符合预期的。

在测试phpinfo()输出时，虽然二者在load上的区别不大，但是`Nginx+PHP-FPM`的组合在进程数这一指标上**完全占优**：

![processes_count_phpinfo.png](http://blog.wislay.com/wp-content/uploads/2016/03/processes_count_phpinfo.png)

考虑到`Nginx+PHP-FPM`的组合无需fork出新的子进程处理新到的客户端请求，以及phpinfo()的执行时间较长这两个因素，同时fork子进程等操作属于消耗系统资源较大的操作，这个现象是符合预期的。

# 总结

使用php7时，在性能上`Apache+modphp7`的组合与`Nginx+PHP-FPM`的组合相差无几，但对于系统负载上来说，`Nginx+PHP-FPM`组合综合表现优于`Apache+modephp7`的组合，可以推断`Nginx+PHP-FPM`的组合对于构建高并发的服务更有优势。

综合考虑，`Nginx+PHP-FPM`表现较优。

# TODO

+ 尝试阅读ab源码，了解其测试原理
+ 比对fpm在动静模式之间的区别


