title: InfluxDB使用笔记
date: 2016-09-11 23:55:44
tags: [InfluxDB]
categories: 系统
---

# 概述

[InfluxDB](https://www.influxdata.com/time-series-platform/influxdb/)是 `InfluxData` 公司发布的一款开源时序数据库产品。

关于时序数据库，除了常用的ElasticSearch之外，InfluxDB也是一个选择。

InfluxDB 使用 go 语言编写。个人认为几个外在的优点在于：

1. 无特殊依赖，几乎开箱即用（如ES需要Java）；
2. 自带HTTP管理界面，免插件配置（如ES的kopf或者head）；
3. 自带数据过期功能；
4. 类SQL查询语句（再提ES，查询使用自己的DSL，虽然也可以通过sql插件来使用类SQL进行查询）；
5. 自带权限管理，精细到“表”级别；

这些好处[官网](https://docs.influxdata.com/influxdb/v1.0/)都有提及。

# 关键概念

## time

`Time` 在 InfluxDB 中的每一行数据中都会存在。

`Time` 即数据生成时的时间戳。

类比更为熟悉的 MySQL 来说，InfluxDB 中的 `Time` 几乎可以看做是主键的代名词了。

## field

`Field` 在 InfluxDB 中是必须的数据的组成部分。

`Field` 未被索引。这意味着如果查询field的话将会扫描所有数据，找出匹配项；手册上也建议不要在field中存储将会频繁查询的数据（存储区分度更高的数据）。

类比 MySQL，`Field` 可以看做没有索引的列。

## tag

`Tag` 在 InfluxDB 中是可选的数据组成部分。

不同于 `Filed`，`Tag` 的数据会被索引。

类比 MySQL，`Tag`就是有索引的列。

## measurement

InfluxDB 是时序数据库，时间数据 `time` 与 `Tag` 以及 `Field` 组合成了 `Measurement`。

类比 MySQL，`Measurement`可以看做的是表。

## retention policy

应用程序记录日志的情况下，如果没有关注，很有可能会出现打爆硬盘的情况出现。运维同学一般会在服务器上部署自动清理脚本。

在InfluxDB中，合理的设置了 `Measurement` 的 `Retention Policy` 的情况下，无需太过担心磁盘被打爆的情况。

`Retention Policy` 可以看做是数据清理策略。基础应用个人认为应该关注如下的一些参数：

+ `DURATION`，即保存时间，有 `m` `h` `d` `w` `INF`几种，即分，小时，天，周以及无限（Infinity）；
+ `REPLICATION`，即副本数，即会有多少份副本保留在集群之中。

## point

`Point` 即同一时间戳产生的数据的集合。

## 实例：grahios使用InfluxDB作为后端

[grahios](https://github.com/shawn-sterling/graphios) 是一个将 Nagios 性能数据，使用不同的后端数据收集工具（or 时序数据库）进行收集的工具。在之前的笔记 [使用Grafana+InfluxDB展示Nagios监控数据](http://www.liaoaoyang.com/articles/2016/08/23/use-grafana-and-influxdb-to-display-nagios-monitor-data/) 之中有所提及。

graphios 使用 InfluxDB 作为存储后端。

# 基本操作

基本操作简单提及数据的 `CURD` 以及数据库管理方面的一些使用方式。

## CURD

### CREATE

在已创建数据库的前提下，新增数据，可以通过两种基本的方式进行数据的写入：

+ CLI
+ InfluxDB HTTP API

#### CLI

CLI 自然与 MySQL 类似，通过 InfluxDB 提供的 CLI 工具，使用类 SQL 语句进行写入。
但是 CLI 毕竟不是使用 InfluxDB 作为后端存储的好方式，InfluxDB 也提供了非常方便的HTTP API 供开发者们使用。

InfluxDB 的特点是无结构的，即文档中提到的 `schemaless`，可以随时的动态增添数据域和表，所以使用时直接写入，无需考虑关系型数据库中类似建表等过程。

不过，CLI 虽然不能做到随时随地的的使用，但是类 SQL 语句还是在特定场景下还是有应用空间的（例如使用默认的Web管理端）。

CLI 写入数据时，语法相当简单，只需要使用：`INSERT 数据` 这样的语句即可。但是数据部分有InfluxDB自己的一些协议，目前在使用的规则叫做 [`Line Protocol`](https://docs.influxdata.com/influxdb/v1.0/write_protocols/write_syntax/#line-protocol)。

##### Line Protocol

文档中已有详细描述，这里总结一下，简单来说，个人认为可以关注如下的特点：

+ 第一个字段是表名
+ 如果有tag，表名之后使用`英文逗号`分隔，以k=v的方式逐个列出
+ field在前两部分之后，用空格分隔开，之后所有的field，也使用`英文逗号`分隔，以k=v的方式逐个列出
+ 如果要指定时间戳，那么在field之后，通过空格分隔，将纳秒级别的时间戳写入
+ 在 `tag` 的键，值，以及 `field` 的键中包含`,` 、`=` 以及空格的，需要进行转义（变为`\,` `\=` `\ `），`field` 的值可以通过 `英文双引号` 包裹起来（但是双引号决定了这个值必然是一个字符串）
+ 数据分为 `string` `int64` `float64` `boolean` 几种类型
+ `string` 最大存储的长度为64KB
+ `float64` 是默认的的类型，如果整数值要存成整数，需要在数字后通过字母 `i` 标明

#### InfluxDB HTTP API



