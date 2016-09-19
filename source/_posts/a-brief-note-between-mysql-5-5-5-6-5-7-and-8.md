title: 简要描述 MySql 5.5/5.6/5.7/8 的区别
date: 2016-09-18 23:58:14
tags: [MySQL]
categories: 系统
---

# 概述

中秋假期的前夕的9月12日，`MySQL 8.0.0` 放出了 [Development Milestone Release](http://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-0.html)。是开源数据库的一大新闻。

要知道MySQL的上一个版本号仅仅是 `5.7`。在MySQL归属于Oracle公司之后，版本号的飞速提升也开始了，按此趋势，预计不日将会赶超Oracle自家商业数据库产品……

发布之际，简要的了解和比较一下 MySQL 5.5/5.6/5.7/8 之间的区别。

# MySQL的开发循环

在比较之前，首先提一下[MySQL的开发循环](https://dev.mysql.com/doc/mysql-development-cycle/en/preface.html).

MySQL一个大版本的开发，大致经历如下几个阶段：

+ Feature Development
+ Feature Testing
+ Performance Testing
+ Lab Releases
+ Development Milestone Releases
+ General Availability Release(GA)

## Feature Development

也即是所谓的特性开发阶段。这个阶段是常规的功能点确定，代码开发，完成CR以及QA等常规的开发流程。

在一系列的测试以及bugfix之后，当QA "signs off" 即测试通过之后，会合并到TRUNK之中。

## Feature Testing

即特性测试阶段。


