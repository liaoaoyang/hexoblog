title: 简要了解 MySQL 5.5/5.6/5.7/8 出现的新特性
date: 2016-09-18 23:58:14
tags: [MySQL]
categories: 系统
---

# 概述

中秋假期的前夕的9月12日，`MySQL 8.0.0` 放出了 [Development Milestone Release](http://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-0.html)。是开源数据库的一大新闻。

要知道MySQL的上一个版本号仅仅是 `5.7`。在MySQL归属于Oracle公司之后，版本号的飞速提升也开始了，按此趋势，预计不日将会赶超Oracle自家商业数据库产品……

发布之际，简要的了解和比较一下 MySQL 5.5/5.6/5.7/8 之间的区别和特性。

如有错误，烦请指出。

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

# 版本比较

对于这些MySQL版本，如果想要了解之间的差异，一个可行的办法是阅读GA版本的Release Notes。

MySQL 的近几个版本发行时间间隔并不太大：


| 版本 | GA 版本发布时间 |
| --- | --- |
| 5.5.8 | 2010-12-03 |
| 5.6.10 | 2013-02-05 |
| 5.7.9 | 2015-10-21 |

大约2年会有一个较大更新的版本会发出。

## 新特性

对于使用者而言，新特性应该是关注的第一焦点。下面会针对版本列出一些个人认为有特点的新特性。

### 5.5

#### InnoDB 作为默认存储引擎

InnoDB 因为支持事务、行级别锁而广为人知，并广泛应用。但是在之前的版本中，InnoDB并不是默认的存储引擎。在5.5中，InnoDB成为了默认的存储引擎。


#### 半同步复制

半同步复制（[Semisynchronous Replication](https://dev.mysql.com/doc/refman/5.5/en/replication-semisync.html)）在MySQL 5.5中被支持（以插件形式实现）。

默认的MySQL通过异步模式进行复制，主库写入binlog之后，从库不一定能够被读取并处理，因为写入成功只是说明在主库上成功。主从不同步带来的问题相当之多，提升了开发难度。

而半同步复制则是主库需要有至少一个半同步从库，当一次写入操作进行之后，至少在主库和至少一个半同步从库上都完成了写入之后，用户才会收到已成功的信息。

半同步复制在这一程度上提高了数据的安全性。

#### 复制心跳

从库如何尽早发现主库宕机？

通过5.5新增的复制心跳机制即可。复制过程中主库会定期发送心跳到从库，当从库SHOW STATUS时可以看到连接情况。

#### 增删索引优化

InnoDB中新（Add）增删除（Drop）索引在这一代中不会复制表中的原始数据，这个操作会在一定程度上提高索引的操作效率。

### 5.6

MySQL 5.6 的主要变化在性能优化方面。有一些小的新特性也值得关注。

#### 表中可以设置多个Timestamp属性

MySQL 5.5 中，如果设定多个Timestamp的属性为 `ON UPDATE CURRENT_TIMESTAMP` 时，这样的操作是不能完成的，这样的需求，通常要在业务代码中完成。

而到了 MySQL 5.6 中，这样的操作可以直接通过设定字段的属性即可完成。

#### InnoDB 支持全文索引

[全文索引](http://dev.mysql.com/doc/refman/5.6/en/innodb-fulltext-index.html) MyISAM 存储引擎之前相对于InnoDB的一个“优势”特性，在MySQL 5.6中不复存在。

针对字符串型的字段(`CHAR`，`VARCHAR`或者`TEXT`)，可以选择在创建表时增加这个类型的索引。也可以后续添加。

InnoDB的全文索引也使用的是倒排索引的设计，分词完成的词汇将会存储在独立的索引表之中。当包含全文索引的字段插入之后，会进行分词，同时先将分词结果进入内存缓存，之后再刷入索引表中，避免一次写入带来的大量附加的小规模的更改操作。

#### 多线程复制

在MySQL 5.6中，会针对每一个数据库开启一个独立的复制线程，如果数据库压力平均的话，对于主从同步延迟会有一定的改善。但是如果数据操作都在一个数据库上，就不会有太多显著的效果了。

#### 加入全局事务ID（GTID）

在MySQL 5.6前，如果从库宕机，重启之后需要进行同步，需要知道binlog文件名已经位置。

在MySQL 5.6中，加入了GTID（[global transaction identifier](http://dev.mysql.com/doc/refman/5.6/en/replication-gtids-concepts.html)）。GTID由source_id和transaction_id构成，source_id标识主库，transaction_id标识在数据库上进行的事务，格式即`GTID = source_id:transaction_id`。

在加入GTID之后，重启从库之后，不需要重新进行位置的指向，只需要连接到主库即可，剩下的步骤将会是自动的。

### 5.7

#### InnoDB

InnoDB地位进一步增强，这一次系统表已然变成了基于InnoDB存储引擎的表。并且也不能禁用InnoDB存储引擎了。

#### 增强的多线程复制

在5.6中添加的多线程复制的增强版，针对每个数据库可以增加线程数进行同步，对5.7.9版本，在实际使用中，在机械盘的服务器上，原有业务高峰时主从同步延迟在10-30分钟左右，使用5.7.9之后基本实现了数据上的同步。

#### 多源复制

即将多个主库的数据归并到一个从库的实例上。

之前的MySQL，每个从库都只能拥有一个主库，如今MySQL提供了官方的解决方案，用于将多个主库的数据同步到一个从库之上。

多源复制有一个关键概念，即频道（channel）。频道指代一个主从库之间用于同步binlog的连接，通过新增的`FOR CHANNEL`子句，指定一个非空的频道名称，按照先前版本的连接主库的方法，即可实现多源复制功能。

需要注意的是，当多个主库均写入同一张表时，需要自行处理主键冲突。

#### JSON数据类型操作

PostgreSQL 9.3开始，PostgreSQL中JSON成为了内置的数据类型。

作为被广泛使用的数据组织格式，之前版本的只能讲JSON格式数据按照字符串形式进行存储。

到了5.7之后，JSON支持也被加入。

JSON中的字符串在MySQL中会被转化成`utf8mb4`的字符集，给携带诸如emoji字符的数据的存储带来了方便。

对于JSON数据的结构特性，MySQL中对JSON的查询需要借助path expression以及`JSON_EXTRACT`方法进行查询。path expression的简要要点如下：

+ 以`$`符号开头
+ `.`符号紧接着的是对象中的key
+ `[n]`中表示的是数组中的第n个元素，n>=0
+ `.[*]`表示一个key下的所有对象
+ `[*]`表示一个key下所有的数组
+ `exp_a**exp_b`则表示path中带有`exp_a`与`exp_b`的值
+ key如果包含特殊字符，需要通过双引号包裹起来

更多操作参见[手册](http://dev.mysql.com/doc/refman/5.7/en/json.html)。

#### innodb_buffer_pool_size参数动态修改

在之前的版本中，`innodb_buffer_pool_size`调整之后，需要重启数据库实例，这个对于线上业务几乎是不可接受的。硬件性能强悍的服务器，调整这一参数之后，MySQL的表现会有较为客观的提升。

到了MySQL 5.7，这一参数终于可以在线调整了。

#### 初始化工具

在之前的版本中，初始化系统表一般都会使用`mysql_install_db`脚本，到MySQL 5.7之后建议使用`mysqld --initialize`完成实例初始化。

在通过`mysqld --initialize`进行初始化时，需要加上`--initial-insecure`才能实现空密码登录，否则会将初始化的默认密码写入到错误文件中。

初始化完成之后，还需要使用MySQL 5.7版本的客户端登录，并且修改默认密码。

### 8.0

作为版本号突飞猛进的一个版本，在MySQL 8.0中新增了如下的特性：

#### 用户角色

8.0中将会增强账号管理的功能，提供`角色`这一概念，即能组合权限，批量授权给某一用户。使用步骤可以参见[博文](https://yq.aliyun.com/articles/60654)或[官方文档](http://dev.mysql.com/doc/refman/8.0/en/roles.html)。

#### 增强的InnoDB

+ 自增id会写入到redo log中，这一改动使得数据库重启之后的自增值能恢复到重启前的状态
+ 增加了死锁检测开关`innodb_deadlock_detect`，可以在高并发系统中动态调整这一特性，提升性能

#### 增强的JSON操作

+ 增加了`->>`操作符，使用这一操作符等同于对`JSON_EXTRACT`的结果进行`JSON_UNQUOTE`操作，简化了SQL语句（PostgreSQL也有类似的操作符）
+ 增加了按JSON数据组织返回数据操作的两个方法：`JSON_ARRAYAGG`与`JSON_OBJECTAGG`。`JSON_ARRAYAGG`将某列的值按照一个JSON数据返回，而`JSON_OBJECTAGG`将列A作为键，列B作为值，返回一个JSON对象格式的数据

后续将会继续更新本文。

# 相关

\[1]: [What’s New in MySQL 5.7? (So Far)](http://mysqlserverteam.com/whats-new-in-mysql-5-7-so-far/)

\[2]: [MySQL 5.7版本新特性连载（一）](http://imysql.com/2015/06/23/mysql-57-new-feature-part-1.shtml)









