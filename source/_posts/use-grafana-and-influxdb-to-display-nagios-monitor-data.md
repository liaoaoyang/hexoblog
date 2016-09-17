title: 使用Grafana+InfluxDB展示Nagios监控数据
date: 2016-08-23 21:32:26
tags: [Nagios, InfluxDB, Grafana]
categories: 系统
---

# 概述

`Nagios`作为监控软件来说已经让人基本满意了，但是个人在使用中感觉还存在部分问题：

1. 不能绘图

可能有同学会说不是有[PNP4Nagios](https://docs.pnp4nagios.org/)这样的软件吗？这个软件Nagios并没有默认携带，需要自己手动安装配置。

另外`PHP4Nagios`虽然已经能满足显示上的需求，但是绘图效果比较古老（图片来自网络，侵删）：

![PNP4Nagios sample](http://blog.wislay.com/wp-content/uploads/2016/08/php4nagios-sample.png)

并且不能对某些数据进行对比（比如多台机器的负载情况在同一个图表中显示）。

2. 无法查看历史数据

Nagios只能看到“现在”的数据，无法回顾历史以及查看趋势。

譬如，如果使用现有的Nagios，想要回顾周日半夜任务对系统的消耗时，就会比较抓瞎了。

# 方案

## 绘图

先说绘图。

绘图作为一个附加的功能，可以考虑现有的一些优秀的数据展现软件，譬如[Kibana](https://www.elastic.co/products/kibana)和[Grafana](http://grafana.org/)。

个人认为，Kibana更适合展现日志计数类型的数据，并且本身没有权限控制等功能，这一点比较麻烦，虽然可以通过第三方插件或者HTTP Basic Auth来解决，但是总是增加了维护成本。

相比之下，Grafana带有权限控制，能支持相当多的时序数据库，从graphite、opentsdb、influxdb，当然也有ES。多条曲线同时展示时还可以看到具体数值，显示效果也相当不错。

![Grafana sample from it's website](http://blog.wislay.com/wp-content/uploads/2016/08/grafana-sample-small.jpg)

并且，Grafana在4.0后将会具有告警功能，玩法会变得更多。

最终选择Grafana作为展示端。

## 数据内容

提一下数据内容。

在这个方案中，我们展现的数据，实际上是Nagios插件产生的Performance Data（或者叫`PERFDATA`），如果自行编写的插件没有按照 `PERFDATA` 的格式输出信息，是不会被收集到InfluxDB中的。如果有自行编写业务监控插件，除了基本的返回码和错误信息之外，要考虑按照标准输出相关的信息，以便收集到InfluxDB中。

Nagios 3的插件多行输出格式为：

<font color="red">TEXT OUTPUT</font> | <font color="#FFAA10">OPTIONAL PERFDATA</font>
<font color="blue">LONG TEXT LINE 1</font>
<font color="blue">LONG TEXT LINE 2</font>
<font color="blue">...</font>
<font color="blue">LONG TEXT LINE N</font> | <font color="#FFAA10">PERFDATA LINE 2</font>
<font color="#FFAA10">PERFDATA LINE 3</font>
<font color="#FFAA10">...</font>
<font color="#FFAA10">PERFDATA LINE N</font>

如磁盘信息的收集，单行模式很好理解，通过 `|` 分隔输出信息以及 `PERFDATA`；多行模式的话，引用官网的例子：

<font color="red">DISK OK - free space: / 3326 MB (56%);</font> | <font color="#FFAA10">/=2643MB;5948;5958;0;5968</font>
<font color="blue">/ 15272 MB (77%);</font>
<font color="blue">/boot 68 MB (69%);</font>
<font color="blue">/home 69357 MB (27%);</font>
<font color="blue">/var/log 819 MB (84%);</font> | <font color="#FFAA10">/boot=68MB;88;93;0;98</font>
<font color="#FFAA10">/home=69357MB;253404;253409;0;253414</font>
<font color="#FFAA10">/var/log=818MB;970;975;0;980</font>

第一行仍然可以按照单行模式进行理解（<font color="red">红色</font>部分是输出信息，<font color="#FFAA10">橙色</font>部分是 `PERFDATA`），从第二行开始，每行一个输出信息，最后将剩余的 `PERFDATA` 紧接着输出信息的最后一行，通过 `|` 分隔，逐行输出。

每项 `PERFDATA` 格式为：

```
'label'=value[UOM];[warn];[crit];[min];[max]
```

引用上述例子的数据 `/=2643MB;5948;5958;0;5968`，可以看到对应关系为：


| label | / |
| --- | --- |
| value | 263MB  |
| warn | 5498 |
| crit | 5958 |
| min | 0 |
| max | 5968 |

`UOM` 表示 `unit of measurement`，即单位，可以是：

+ 不设定（即数字）
+ 秒`s`（或者`ms`或`us`）
+ 百分数`%`
+ 字节`B`（或者`KB`, `MB`, `TB`）

详情可以参阅 [Nagios Plugin Development Guidelines](https://nagios-plugins.org/doc/guidelines.html) 以及 [Nagios Plugin API](https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/pluginapi.html)。

## 数据落地

只有Grafana是不能完成期望的工作的，Grafana只是展现工具，需要有数据源才能展现。

目前主力使用的时序数据库是ES，但是可以看到Nagios官方网站上关于[绘图](https://exchange.nagios.org/directory/Addons/Graphing-and-Trending)这一主题推荐的插件中没有看到写入到ES中的插件（ES倒是可以把输出作为监控信息交给Nagios，[参见](http://kibana.logstash.es/content/logstash/plugins/output/nagios.html)）。

查阅资料，graphite性能可能存在问题，opentsdb需要有专业的运维同学维护（毕竟是Hadoop衍生产品），InfluxDB兼顾了性能和运维成本，考虑使用。

值得一提的是，InfluxDB提供了Web管理端，查询语句上类似SQL，便捷程度让人满意。

## 数据收集

落地方式确定之后，需要考虑如何将Nagios收集的诸如load、CPU、内存使用等信息存入InfluxDB了。

[graphios](https://github.com/shawn-sterling/graphios)可以相当便捷的完成这一工作。这是一个将Nagios perf data写入到InfluxDB中的工具。配置也相当便捷，基本上按照README即可完成。

[Nagflux](https://github.com/Griesbacher/nagflux)看起来也是不错，然而问题在于这个适用于`4.x`版本的Nagios，在不打算升级Nagios的情况下，只能放弃。

但是，凡事总有例外，使用软件的特定组合的情况下，会出现一些问题。

### 数据解析异常

graphios在Nagios官网上声称匹配`3.x`版本的Nagios，但是在与实际使用的Nagios `3.5`结合时，会出现部分perf data无法成功解析的情况，检查后发现Nagios `3.5`的部分数据在graphios读取时，并没有按数据协议组织，导致解析错误。

对此，针对输出数据，修改了解析所在部分[源码](https://github.com/shawn-sterling/graphios/blob/master/graphios.py)进行了匹配，筛选出了所需数据。

### InfluxDB版本

提示的InfluxDB版本较新，为`0.13`，然而，在graphios的配置文件中，backend部分，只有`enable_influxdb09`以及`enable_influxdb`选项，默认使用的json数据写入协议已经在`0.13`中[废弃](https://docs.influxdata.com/influxdb/v0.13/write_protocols/json/)。

上述问题，考虑到版本后续兼容性，将`0.13`视作`0.9`版本配置，并且配置数据写入协议为[`line`](https://docs.influxdata.com/influxdb/v0.13/write_protocols/line/)即可（即配置`influxdb_line_protocol = True`）。

为了让graphios永驻后台，可以考虑使用supervisord等工具。

## 展现样例

### load

![load](http://blog.wislay.com/wp-content/uploads/2016/08/nagios-load-data-in-grafana.jpg)

### CPU

![CPU](http://blog.wislay.com/wp-content/uploads/2016/08/nagios-cpu-data-in-grafana.jpg)

# 小结

使用`Nagios`+`graphios`+`InfluxDB`+`Grafana`可以将监控数据以更好的方式进行展现，并能够回顾历史，查看趋势，随着Grafana的升级，还可以有更多的玩法。

