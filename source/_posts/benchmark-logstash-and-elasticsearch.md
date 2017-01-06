title: 一组Logstash与Elasticsearch的压测数据
date: 2016-03-16 22:55:02
categories: 系统
tags: [Logstash, Elasticsearch]
---

# 概述

组内的日志系统基于ELK搭建，本文中的数据在生产环境中进行测试得到，仅供参考。

# 系统构成

系统可以简要的分为：

+ 日志接收机
+ 日志数据队列
+ 日志数据处理机
+ ES集群

在`日志接收机`上通过一个 *Logstash* 进程 parse 日志数据，将 parse 后的结构写入由 *Redis List* 实现的`日志数据队列`中，之后在`ES集群`前，再使用一个`日志处理机` *Logstash* 进程从 *Redis* 中 pop 出数据写入`ES集群`中。

使用 *Redis List* 的原因是在于当 parse 能力大于`ES集群`的处理能力时，缓存数据。

# 运行环境

`日志接收机`为CPU为4 \* Xeon E5强IO型机器，*Redis List* 与向ES集群写入的 *Logstash* 位于同一机器上，即`日志数据处理机`，CPU为12 \* Xeon E5。

处理日志为Nginx access日志，记录了如时间、域名、访问IP、URL、HTTP Method、响应时间、返回体长度等，约10+字段。

# 参数配置

日志接收机上的 *Logstash* 配置成了10个线程，output 中的 redis 的参数配置为：

```
output {
    redis {
        host => "192.168.1.101"
        port => '6379'
        data_type => "list"
        key => "Logstash_benchmark"
        type => "nginx_access"
        threads => 5
    }
}
```

日志数据处理机上的 input 对已配置为：

```
input {
    redis {
        host => "192.168.1.101"
        port => '6379'
        data_type => "list"
        key => "Logstash_benchmark"
        type => "nginx_access"
        threads => 5
    }
}
```

而 output 配置为：

```
output {
    if ([type] == "nginx_access") {
        elasticsearch_http {
            host => "192.168.2.101"
            port => "9527"
            flush_size => 10000
            workers => 10
            index => "Logstash_benchmark"
        }
    }
}
```

# 测试方法

测试分为测试 *Logstash* 分析日志能力，以及`ES集群`写入能力。

*Logstash* 的分析能力通过每秒取样 *Redis List* 中的新增队列长度获得（`日志接收机`上的 *Logstash* 生产）。

`ES集群`的写入能力通过每秒取样 *Redis List* 中减少的队列长度获得（`日志数据处理机
`上的 *Logstash* 消费）。

在测试一个阶段时，会关闭另一端的 *Logstash*。

# 数据

平均数据后，可供参考的数据为：

*Logstash* 分析速度为：**2352.94** lines/s

`ES集群`写入速度为：**9345.79** records/s
