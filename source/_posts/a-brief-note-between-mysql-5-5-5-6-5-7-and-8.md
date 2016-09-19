title: 简要描述 MySql 5.5/5.6/5.7/8 的区别
date: 2016-09-18 23:58:14
tags: [MySQL]
categories: 系统
---

# 概述

中秋假期的前夕的9月12日，`MySQL 8.0.0` 放出了 [Development Milestone Release](http://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-0.html)。是开源数据库的一大新闻。

要知道MySQL的上一个版本号仅仅是 `5.7`。在MySQL归属于Oracle公司之后，版本号的飞速提升也开始了，按此趋势，预计不日将会赶超Oracle自家商业数据库产品……

发布之际，简要的了解和比较一下 MySQL 5.5/5.6/5.7/8 之间的区别。

# MySQL的开发周期

在比较之前，首先提一下[MySQL的开发周期](https://dev.mysql.com/doc/mysql-development-cycle/en/preface.html).

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

特性测试的阶段，从质量保证的角度出发，MySQL的测试工作将会有以下的关注点：

+ 完成新特性的基本测试工作
+ 未出现性能衰减/特性的倒退
+ 至少达到80%的代码覆盖率

一个新特性达到要能被合并到TRUNK之中，需要满足如下条件：

+ 没有任何已知的错误，包括那些微小的错误
+ 没有任何已知的性能衰减/特性的倒退
+ 代码覆盖率达到预期
+ 自动化测试集中需要有这一新特性的回归测试用例

## Performance Testing

作为应用的基础组件，性能是绝大多数开发人员都在关注的问题。在特性测试完成之后，将要进行的是性能测试。

性能测试主要关注两个指标：`吞吐量`和`响应时间`。

针对于`吞吐量`的测试，会有如下的特点：

+ 并发测试范围从`8`到`1024`个连接
+ 使用诸如`sysbench`这类开源软件，进行简单[OLTP](https://en.wikipedia.org/wiki/Online_transaction_processing)（基本的事务处理）的测试，每个测试时间在5-10min范围内，随着数据集以及系统配置而定
+ 针对特定场景也会有相应的测试用例
+ 测试数据会被存储到数据库中，以便比较或者确定测试的基准值

而对于`响应时间`的测试，则会有如下的特点：

+ 在单个被测试线程上完成测试
+ 准备两方面测试用例，分别考验计算复杂型场景和IO密集型场景

## Lab Releases

实验室发布是在用户对某一特性有着强烈的兴趣时，发布的快照版本，通常还没有合并到TRUNK之中。

实验室发布版本存在的特性，并不会保证在正式版等版本中存在。如果想要尝鲜，可以访问[labs.mysql.com](http://labs.mysql.com/)。

## Development Milestone Releases

上述步骤结束之后，到了DMR阶段，也就是本次 8.0 版本所处的阶段。

DMR将会是一个重复迭代的阶段，下一个DMR版本将会包含上一个DMR版本的新特性或者功能修复。并且，DMR了的MySQL将会支持所有的平台，DMR状态下的MySQL的代码质量是达到了可供发布的水准的。

DMR看起来像是游戏的公测版本，每3-6个月会发布一个版本，每个DMR版本的MySQL会有一个特性需求收集截止时间点，新增的特性会历经开发、测试的历程，合并到TRUNK之中，变为下一个DMR。

总之，DMR的目的就是能够时常发布，让用户和客户能够反馈需求，体验新版本。

## General Availability Release(GA)

终于，在n个DMR之后，MySQL表现稳定，需求“无可”增加，客户满意，质量过关。

基于最后一个DMR版本，会发布GA版本，即我们可以“放心”使用在生产环境中的MySQL。

每个GA版本的发布周期间隔在18-24个月。






