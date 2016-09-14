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


