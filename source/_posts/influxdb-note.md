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

[手册](https://docs.influxdata.com/influxdb/v1.0/concepts/glossary)中对这些名词有了详细的定义，这里稍微提及一下。

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

## series

`Series` 表示`Measurement`、`Retention Policy`、`Tag`都相同（tag的名称与值）的一组数据，`Field`不在考虑衡量范围之内。

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

InfluxDB 启动后，会默认监听 `8086` 端口，在这一端口上可以通过 HTTP POST 的方式，请求 `/write` ，写入数据。

写入方式[手册](https://docs.influxdata.com/influxdb/v1.0/guides/writing_data/#creating-a-database-using-the-http-api)已很清楚，即将 Line Protocol 格式的的数据提交到 InfluxDB 之上。

需要注意的是，InfluxDB 在通过HTTP方式操作时，需要注意下HTTP的状态码，尤其要注意`204`这一状态码，InfluxDB在返回这一状态码时认为请求合理但是并未完成，当返回这一个数值是需要注意HTTP的body段中附加的错误信息。至于`4xx`与`5xx`则是真的存在错误的情况。

#### 时序相关

既然是时序数据库，时间作为重要的维度需要单独一提。

通过 `Line Protocol` 写入数据时，如果没有指定时间，将会根据系统时间指定时间，所以手册中提到：

>  If you do not specify a timestamp for your data point InfluxDB uses the server’s local nanosecond timestamp in UTC.

这个和 ES 的默认行为不太相同，所以这里需要注意。

如果要人工指定时间，需要按照 [RFC3389](https://www.ietf.org/rfc/rfc3339.txt) 里提及的格式指定，即指定一个`ns`级别的时间戳，形如`1465839830100400200`。

### RETRIEVE

与 `CREATE` 这一动作相同，数据的获取同样可以通过在 CLI 中执行InfluxQL（即InfluxDB中定义的类 SQL 语言），或者通过 HTTP API 提交查询语句的方式进行。通过 HTTP API 访问的话，InfluxDB会返回JSON格式的[数据](https://docs.influxdata.com/influxdb/v1.0/guides/querying_data/#querying-data-using-the-http-api)。

InfluxDB使用类 SQL 语言进行查询，确实是降低了使用的门槛，类似 SQL 的 `SELECT` `GROUP BY` `LIMIT` `ORDER BY` 操作的用法参见[手册](https://docs.influxdata.com/influxdb/v1.0/query_language/data_exploration/)。

#### GROUP BY

值得一提的是，作为时序数据库，InfluxDB在查询上有一些时间相关的操作，例如 `GROUP BY` 操作可以按照时间戳进行聚合，获得每个时间片级别上的聚合数据。即通过 `GROUP BY time(interval)` 方法进行聚合，`interval` 即支持的时间片，支持的范围有：

| 时间片单位 | 范围 |
| --- | --- |
| u | 微秒 |
| ms | 毫秒 |
| s | 秒 |
| m | 分钟 |
| h | 小时 |
| d | 天 |
| w | 周 |

比如 `time(3d)` 表示按照`3天`作为周期进行聚合。

同时 `fill()` 方法也是一个有意思的功能，使用 `GROUP BY` 之后，如果某个聚合的结果内没有值，可以通过 `fill()` 设定为自己需要的值。更详细的用法参见[手册](https://docs.influxdata.com/influxdb/v1.0/query_language/data_exploration/#the-group-by-clause-and-fill)。

#### INTO

如果说上述的的一些操作看起来和平常使用的关系型数据库没有特别大的区别的话，我想从 `INTO` 开始，InfluxQL就体现出时序数据库的一些特点了。

比如机器性能数据的采集，数据点多了之后，很可能我们期望能够看到的不是单一时间点上的数据，而是机器阶段性的表现和趋势，这样的数据自然可以通过聚合具体的采集数据获得，但是为什么不直接读取已经聚合完成的数据并展现呢？重复的计算是对机器性能的巨大浪费。

通常情况下，我们可能需要编写程序完成这样的工作。但是在InfluxDB中，可以通过内置的 `INTO` 语句来完成，无需额外的工作。

结合[手册](https://docs.influxdata.com/influxdb/v1.0/query_language/data_exploration/#the-into-clause)中的例子：

```
SELECT MEAN("water_level") INTO "average" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)
```

即在`2015-08-18T00:00:00Z`到`2015-08-18T00:30:00Z`的时间范围内，将每12分钟在`santa_monica`区域的水位的平均值存入`average`表中。

如是，如果要了解平均值，无需重复计算原始数值。

对于实际业务来说，统计接口PV/UV的场景很适合使用这一个子句，因为阶段统计中，并不关心具体的访问用户，而是期望了解趋势（对于UV的定义需要结合实际的业务场景）。

#### LIMIT & SLIMIT

`LIMIT` 限制的是数据的行数，而 `SLIMIT` 限制的则是返回的 `Series` 的个数，如果二者都要限制，则先设定 `LIMIT`，之后再设定 `SLIMIT`。

#### OFFSET & SOFFSET

分页是查询数据的一个基本方式，InfluxQL中通过 `OFFSET` 以及 `SOFFSET` 进行。

类似 LIMIT & SLIMIT， `OFFSET` 是针对数据行的翻页，`SOFFSET` 是针对 `Series` 的翻页。

#### 时序相关

时序数据库，自然查询的数据经常会与时间有关。前面提及的时间单位，在选定时间范围时，可以通过组合简化时间区间的确定。

例如选出当前时间前3天到5小时前的数据，那么只需要指定条件为`time >= now() - 3d AND time < now() - 5h`即可。

如果你愿意，也可以与 [RFC3339](https://www.ietf.org/rfc/rfc3339.txt) 格式的时间做对比。

### UPDATE & DELETE

InfluxDB不存在常规意义上的 `UPDATE` 和 `DELETE` 这些动作！需要注意！

InfluxDB的设计初衷是为了记录时序相关的数据，这些数据被认为是不常变的，并且是仅仅有一次写入操作的（瞬时应当是唯一的）。

综上，InfluxDB期望通过牺牲了数据的灵活性，换取了更好的写入与查询性能。


# 实例：grahios使用InfluxDB作为后端

[grahios](https://github.com/shawn-sterling/graphios) 是一个将 Nagios 性能数据，使用不同的后端数据收集工具（or 时序数据库）进行收集的工具。在之前的笔记 [使用Grafana+InfluxDB展示Nagios监控数据](http://www.liaoaoyang.com/articles/2016/08/23/use-grafana-and-influxdb-to-display-nagios-monitor-data/) 之中有所提及。

graphios 使用 InfluxDB 作为存储后端，首先完成的步骤，是将 PERFDATA 解析，之后按照 `Line Protocol` 拼装，发送到后端。

由于 InfluxDB 是 Schemaless 的，所以 graphios 无需进行提前建表这一操作，直接发送数据即记录。

比较让我关心的是 `Line Protocol` 的运用，参见提交 sha 值为 [02098f](https://github.com/shawn-sterling/graphios/commit/02098f3dd08ffa2f649016c4489d6665900bdc65) 的 `graphios_backends.p` 的文件的667行的 `format_metric` 方法：

```
def format_metric(self, timestamp, path, tags, value):
    if not self.influxdb_line_protocol:
        return {
            "timestamp": timestamp,
            "measurement": path,
            "tags": tags,
            "fields": {"value": value}}
        return '%s,%s value=%s %d' % (
	         path,
	         ','.join(['%s=%s' % (k, v) for k, v in tags.iteritems() if v]),
	         value,
	         timestamp * 10 ** 9
	    )
```

## 生成Measurement

使用了 `Line Protocol` 的情况下，输入变量`path`将会作为 `Measurement`的名称，这里的 `path` 即 Nagios 中的 `HOSTCHECKCOMMAND` 或者 `SERVICEDESC` Macro 的值，即检测服务的名称。例如磁盘就是 `Disk`。

## 生成Tags

所有的检测数据都会作为 tags 存在，目的应是便于查询。检测数据的格式形如：

<font color="red">DISK OK - free space: / 3326 MB (56%);</font> | <font color="#FFAA10">/=2643MB;5948;5958;0;5968</font>
<font color="blue">/ 15272 MB (77%);</font>
<font color="blue">/boot 68 MB (69%);</font>
<font color="blue">/home 69357 MB (27%);</font>
<font color="blue">/var/log 819 MB (84%);</font> | <font color="#FFAA10">/boot=68MB;88;93;0;98</font>
<font color="#FFAA10">/home=69357MB;253404;253409;0;253414</font>
<font color="#FFAA10">/var/log=818MB;970;975;0;980</font>

<font color="#FFAA10">橙色</font>部分是 `PERFDATA`，将会被解析成Tags所需的格式：

`/=2643MB,/boot=68MB,/home=69357MB,/var/log=818MB`

由于 Nagios 中 `/` 一般被设定为需要转义的字符，所以，写入数据库时，可能会被转换为：

`_=2643MB,_boot=68MB,_home=69357MB,_var_log=818MB`

至于如 PERFDATA 中的 `WARNING` `CRITICAL` 等标识去哪里了，可以看到提交 sha 值为[8aa93b](https://github.com/shawn-sterling/graphios/commit/8aa93b6ea3895777c5d8e372d72d253c4f511a42) 的 `graphios.py` 文件的358行的 `process_log`方法，文件387行开始：

```
(nobj.LABEL, d) = metric.split('=')
v = d.split(';')[0]
u = v
nobj.VALUE = re.sub("[a-zA-Z%]", "", v)
nobj.UOM = re.sub("[^a-zA-Z]+", "", u)
```

说明了实际上原始日志处理过后，只会有等号后的第一个数据存在。

## 生成Fileds

由于 `Fileds` 是InfluxDB数据不可缺少的一部分，但是实际上，由于没有索引的原因，查询时的成本比较大，所以 graphios 把 `PERFDATA` 的value作为数据的一个field `value`记录，个人任务因为这个value并没有足够的细节，具有足够的细节的数据已都做为tag存在。

## 生成时间戳

由于需要ns级别的时间戳，所以需要在检测的时间戳上乘以特定倍数即可。

## 查询

查询的工作，则交给Grafana这类专业的数据展现工具完成。

# 小结

以上是InfluxDB的基础使用笔记，数据库的诸如`认证`，`部署`，`集群`，以及使用上的如`降采样(Downsampling)`预计将会在后续笔记中提及。


