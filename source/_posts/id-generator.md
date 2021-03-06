title: 发号器设计简述
date: 2018-12-25 23:00:00
tags: [Snowflake]
categories: 系统
---

# TL;DR

发号器用于为系统中各个模块提供ID生成服务。

设计一个发号器至少需要考虑ID唯一性，有序性，性能，可用性，使用成本，以及一定的数据安全问题。

作为一个极其成熟的业务功能，发号器有大量的实现，有海量的文章描述各类要点，按需选择即可，无需重复造轮子。

<!-- id-generator -->
<!-- more -->

# 为什么不使用UUID？

[UUID](https://zh.wikipedia.org/wiki/%E9%80%9A%E7%94%A8%E5%94%AF%E4%B8%80%E8%AF%86%E5%88%AB%E7%A0%81) 算法上使用部分硬件信息，从前面提到的几个维度来分析一下：

+ 唯一性上差强人意（虚假的硬件信息等）
+ 有序性上不完全能够保证
+ 性能上由于可以是本地生成没有太大问题
+ 可用性亦然
+ 使用成本上开发成本低，然而占用空间等较大
+ 安全性上不能被遍历，高版本的生成算法可以让机器无法回溯

综上，使用 UUID 带来的问题首先就是有序性上考量让人质疑，同时占用的空间较为巨大，生成的ID也较难理解，使用了不同算法实现的UUID需要深入实现才能人工制造出一个同类型的UUID。

最直接的影响，就是当前大量系统使用数据库，UUID 作为主键会[严重影响性能](http://imysql.com/2014/09/14/mysql-faq-why-innodb-table-using-autoinc-int-as-pk.shtml)。

并不是说不能使用UUID，而是需要慎重考虑，属于架构上的取舍。

# 解决问题

合理设计的发号器可以解决如下问题（部分）：

## 分库分表的分区键

业务在初始阶段往往是一个单体应用，往往选择依赖数据库生成自增ID来标示新增数据。

随着业务增长，单表数据增长到千万及以上，或者出现了激烈的竞争条件，需要考虑进行分库分表操作。

此时的ID成了问题，通过发号器为每条记录生成一个全局唯一的ID，可以避免不必要的主键冲突等问题，让数据各得其所。

## 自增ID的数据安全性

例如订单相关的业务，每日的流水属于业务的机密数据，如果直接选用自增ID作为订单号，竞对很容易直接估算出每天的下单量（例如间隔24小时下单）。

通过特定算法实现的发号器，一定程度可以规避这一问题。

## ID的可读性

业务上的ID如果单纯是数字的信息量就太小了。

可以在ID中包含一些业务信息，帮助快速的定位问题。


# 常见架构

## 生成算法

大量的发号器设计上都使用 [Snowflake](https://github.com/twitter-archive/snowflake) 作为ID生成算法。

核心就是`时间戳`+`节点信息`+`业务数据`，通过位运算组合，在10进制模式下，数字基本单调递增而不连续。

生成的 ID 是64位数字，存储空间和表现力上来说对于绝大多数系统足够。

### 时间戳

默认的算法是41位用于毫秒级别的时间戳，通常还会搭配一个起始时间（默认是时间戳的0点），可供使用`相对`的69年时间，日常使用中可以考虑缩减，腾出更多空间用于业务数据，毕竟6年之后的事情都难以预知。

### 节点信息

一般每个服务器分配一个编号，可以结合 ZK / Etcd 等分布式协同组件管理编号，机器本身也可以配置一个默认编号用于降级，多台服务器使用一个编号主要带来的问题就是冲突。

### 业务数据

业务数据包含两个方面，一个是业务区分信息，即表示这一类型的ID是何种ID，便于查询问题，能够迅速判断应当从哪一系统入手。

另一个方面自然是核心的编号，也就是自增的数字序列。

虽然已经做了`时间`（ID 中携带时间戳）以及`空间`（ID 中的节点信息)上的隔离，但是限于 ID 的数据长度，1秒（或者1毫秒）内同一节点上生成的是有限个的，取决于数据长度。

## 架构设计

为了实现全局唯一，需要一个中心节点。

从对中心节点的依赖程度出发，可以分为两类实现，

### 纯中心实现

对于强依赖类型的发号器，每个 ID 的生成需要从中心节点获取。

最常见的实现方式就是直接使用数据库的自增 ID 生成序列，或者基于 Redis 这类 NoSQL 的内置特性完成。

一般采用这类方案的问题在于存在单点问题以及性能有上限（然而又有多少系统可以达到系统性能的上限呢？），项目中的订单系统实战过自增 ID + Snowflake的方案。

基于 Redis 的自增 ID 方案可以参考 [唐福林](https://weibo.com/tangfl) 老师的博文《6行代码实现一个 id 发号器》，结合 Lua 实现，言简意赅。由于原文链接已经失效，这里摘抄一下：

```
id 发号器的问题， @一乐 的这篇文章说的很透彻了： http://weibo.com/p/1001603800404851831206 但参考实现就显得有些复杂。最近在雪球工作中正好需要用到发号器，于是用 Lua 在 Redis 上实现了一个最简单的：


-- usage: redis-cli -h 10.10.201.100 -p 10401 EVAL "$(cat getID.lua)" 1 XID:01:02 
local arg = KEYS[1] 
-- project id must has 2 digits, 01 - 15 
local pid = tonumber(string.sub(arg, 5, 6)) 
-- instance id must has 2 digits, 01 - 15 
local iid = tonumber(string.sub(arg, 8, 9)) 
local idnum = redis.call("INCR", "ID_IDX") % 65536 
local sec = redis.call("TIME")[1] - 1420041600 
return sec*16777216 + pid*1048576 + iid*65536 + idnum 

解释：

id 总长度 52bit，为了兼容 js，php，flex 等语言 long 类型最长只能 52 bit
最高 28 bit 为秒级时间戳，因为位数限制，不能从 1970.1.1 开始，-1420041600 表示从 2015.1.1 开始，大约可以使用10年（3106天）
接下来4个bit为 project id，客户端传入，区分业务
再接下来4个bit为 instance id，HA 用的，支持最多16个instance。如果业务只需要“秒级粗略有序”，比如发微博或发评论，则可以多个 instance 同时使用，不需要做任何特殊处理；如果业务需要“因果有序”，比如某个user短期内快速做的多个操作必须有因果顺序（程序化交易，必须先卖再买，几个毫秒内完成），那么就需要做一些特殊处理，要么用 uid 做一致性hash，或者像我们这样偷懒：一段时间内固定使用一个 instance
最低16个bit是 sequence id，每个 instance 支持每秒 65535 个 id。bit数再大，redis 该成为瓶颈了。
twitter和微博都是把 instance id 写死到 server 端，这样 server端就变成有状态的了：16个instance，每个 instance 都与其它的配置不一样。在雪球我们不希望 server 端有状态，于是设计成 instance id 由客户端传入，server 端退化成一个普通的 redis cache server （不需要 rdb，不需要 aof，宕机重启即可）
几个注意点：宕机不能自动立即重启，必须间隔1秒以上，避免 sequence id 重复导致 id 重复；迁移时必须先 kill 老的 instance，再启动新的，也是为了避免 sequence id 重复。
id 发号器说简单也确实挺简单。任何一个技术点，只要理解了本质，大约都是这么简单罢。
```

这类方案几乎能满足中小业务的需求了，成本上可以复用现有基础设施，如果是做成了公共服务，成本会进一步降低。

纯中心实现有一个优势，就是所有ID能保证全局单调递增。

### 中心-客户端实现

这部分又可以分为`序列中心分配`和`序列客户端自行生成`两类：

#### 序列中心分配

这类方案可以参考 [Leaf](https://github.com/Meituan-Dianping/Leaf) 。简单来说，有如下一些特点：

+ 通过预先分配 ID 区间解决性能问题，每次按照特定步长分配ID，客户端缓存区间本地生成 ID
+ 通过为不同客户端分配不同区间解决 ID 唯一性问题，在客户端内实现 ID 的单调递增
+ 结合 Snowflake 算法解决 ID 信息安全性问题

在某些数据库中间件中也会提供类似的方案，专门的发号系统还会增加如下的一些组件，解决部分问题：

+ 使用 ZK/Etcd 保证节点信息的唯一性
+ 引入时钟修正机制解决节点时间错乱问题，保证 ID 全局唯一
+ 预先获取下一区间降低TP999，减少系统响应时间毛刺
+ 同机房优先调用/中心节点自动切换等运维手段提升中心的可用性

这类方案仍然需要中心节点实现高可用，但是极大程度的降低了中心节点的压力。

然而，这类方案也存在一些问题：

+ 难于在线调整步长
+ 闰秒的处理（几乎所有基于时间的方案都需要注意）
+ 对序列数据直接使用仍然有可能猜测出业务量级
+ ID 只是局部单调递增
+ 有大量组件依赖，适合做成公共服务

由于需要客户端缓存，对于 PHP 这类非常驻进程的语言，可以考虑引入共享内存完成功能。

#### 序列客户端自行生成

如果固定了序列的位数，同时引入分布式协同组件，为客户端提供时间校正以及分配节点ID，结合 Snowflake 算法，可以在客户端自定生成序列。这类方案的优点在于：

+ 去除了数据库等序列生成中心组件的依赖

这一类方案的问题是，1个时间片内（1s/1ms）序列使用完毕后，不能再生成新的ID，为了防止覆盖已生成ID，需要等到下一个时间片内才能继续生成。

另一个问题是，单个节点上存在多个应用时，面临着多个应用竞争同一个 ID 的情况，需要通过一些锁定的手段，保证序列正常发放。当然，如果节点的概念是每个应用，则这个问题可以忽略，但是带来的问题就是分配的节点信息的位数可能需要增加。

虽然去除了数据库的依赖，仍然存在 ZK / Etcd 等协同组件的依赖，对于小型业务系统来说，在有限节点的情况下，可以考虑直接使用 IP 转化为节点 ID，结合 NTP 服务器，去除分布式协同组件的依赖，彻底实现分布式的 ID 生成。

# 其他

+ [业务系统需要怎样的全局唯一ID？](http://ericliang.info/2015/01/18/what-kind-of-id-generator-we-need-in-business-systems.html)
+ [Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/2017/04/21/mt-leaf.html)
+ [baidu/uid-generator](https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md) 
+ [微信序列号生成器架构设计及演变](https://www.infoq.cn/article/wechat-serial-number-generator-architecture)
+ [分布式ID生成系统怎么做](http://www.10tiao.com/html/773/201712/2247487246/2.html)
+ [如何做一个靠谱的发号器](https://tech.youzan.com/id_generator/)
+ [PHP ramsey/uuid](https://github.com/ramsey/uuid)



