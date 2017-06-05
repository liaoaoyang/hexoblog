title: MySQL InnoDB事务隔离级别笔记
date: 2017-06-05 08:14:30
tags: [MySQL]
category: LAMP/LNMP
---

# TL;DR

MySQL InnoDB可以设定ANSI SQL-92中规定的四个事务隔离级别，事务并发度和事务隔离级别成反比，事务隔离级别越高，并发度越低。

是否有锁操作，取决于当前的读取是快照读，还是当前读，快照读读取的是可见的历史版本，无需上锁，简单的读操作是快照读（SELECT无附加语句），其他都是当前读，需要加锁。

对无索引的数据，InnoDB会锁定全表的记录，但是会在扫描过程中释放不符合筛选规则的记录的锁定。

对于有索引的数据，需要具体分析，在RR这一级别，InnoDB通过GAP锁机制避免了幻读问题。

<!-- more -->

# ANSI SQL-92标准

事务隔离级别在[ANSI SQL-92](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)标准中有了明确的定义，即我们熟知的`READ UNCOMMITTED`/`READ COMMITTED`/`REPEATABLE READ`/`SERIALIZABLE`，级别由低到高。

在大量博文中经常提到的比如`脏读（Dirty read）`，`不可重复读（Non-repeatable read）`，`幻读（Phantom）`，标准也有提及。

**脏读（Dirty read）**

```
1) P1 ("Dirty read"): SQL-transaction T1 modifies a row. SQL-
   transaction T2 then reads that row before T1 performs a COMMIT.
   If T1 then performs a ROLLBACK, T2 will have read a row that was
   never committed and that may thus be considered to have never
   existed.
```

简而言之，即两个或多个事务中有一个事务更改了数据，无论提交与否，都会读取到数据的改动。

在ANSI SQL-92标准中，`READ UNCOMMITTED`会可能出现脏读。

**不可重复读（Non-repeatable read）**

```
2) P2 ("Non-repeatable read"): SQL-transaction T1 reads a row. SQL-
   transaction T2 then modifies or deletes that row and performs
   a COMMIT. If T1 then attempts to reread the row, it may receive
   the modified value or discover that the row has been deleted.
```

两个或多个事务中有一个事务更改了数据，只有提交之后，才会读取到数据的改动。

在ANSI SQL-92标准中，`READ COMMITTED`不会出现脏读，但是会可能出现不可重复读。

**幻读（Phantom）**

```
3)  P3 ("Phantom"): SQL-transaction T1 reads the set of rows N
    that satisfy some <search condition>. SQL-transaction T2 then
    executes SQL-statements that generate one or more rows that
    satisfy the <search condition> used by SQL-transaction T1. If
    SQL-transaction T1 then repeats the initial read with the same
    <search condition>, it obtains a different collection of rows.
```

幻读发生的前提是一个事务改动（新增）并提交处于其他某一事务查询条件范围内，并且其他事务重复使用`同一条件`进行查询。

在ANSI SQL-92标准中，`REPEATABLE READ`不会出现脏读，不会出现不可重复读，但是会可能出现幻读。

在最高的事务隔离级别`SERIALIZABLE`中，以上问题都不会出现。


# MySQL InnoDB的RR事务隔离级别

InnoDB已经是MySQL首选存储引擎，内容会针对默认的隔离级别展开。

MySQL InnoDB实现了标准中的4个事务隔离级别，默认的事务隔离级别是`REPEATABLE READ`：

```
mysql>  SELECT @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
```

和标准不同的是，MySQL在这一级别通过GAP锁消除了幻读问题。

MySQL的并发度和事务隔离级别成反比，事务隔离级别越高，并发度越低。

## 幻读的解决方案：GAP锁

GAP锁用于锁定记录的间隙，防止事务并发时同一搜索条件下，出现事务中前后两次读取结果不同的情况。

来看测试表`test`:

```
mysql> desc test;
+-------+------------------+------+-----+---------+----------------+
| Field | Type             | Null | Key | Default | Extra          |
+-------+------------------+------+-----+---------+----------------+
| id    | int(11) unsigned | NO   | PRI | NULL    | auto_increment |
| v1    | int(11)          | NO   | UNI | 0       |                |
| v2    | int(11)          | NO   | MUL | 0       |                |
| v3    | int(11)          | NO   |     | 0       |                |
+-------+------------------+------+-----+---------+----------------+
```

现有数据：

```
mysql> SELECT * FROM test;
+----+-----+-----+-----+
| id | v1  | v2  | v3  |
+----+-----+-----+-----+
|  1 |   1 |   1 |   1 |
|  3 |   2 |   2 |   2 |
|  4 |   3 |   3 |   4 |
|  5 |   4 |   4 |   5 |
|  8 | 100 | 100 | 101 |
| 12 | 200 | 200 | 200 |
+----+-----+-----+-----+
```

事务A（ID 381797）尝试读取v1=10的记录：

```
mysql> SELECT * FROM test WHERE v1=10 FOR UPDATE;
Empty set (0.01 sec)
```

这是一个当前读，由于`v1`是一个唯一索引，v1=10这一个条件处于id=5,v1=4与id=8,v1=100的记录之间的间隙，所以为了防止两次读取数据不一样，需要在这之间增加GAP锁。

同时事务B（ID 381798）尝试写入一条记录：

```
mysql> INSERT INTO test(v1,v2,v3) VALUES(6,6,6);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

v1=6处于间隙之中，通过查询当前锁情况：

```
mysql> SELECT * FROM information_schema.INNODB_LOCKS\G
*************************** 1. row ***************************
    lock_id: 381798:1437:4:6
lock_trx_id: 381798
  lock_mode: X,GAP
  lock_type: RECORD
 lock_table: `learn`.`test`
 lock_index: v1
 lock_space: 1437
  lock_page: 4
   lock_rec: 6
  lock_data: 100
*************************** 2. row ***************************
    lock_id: 381797:1437:4:6
lock_trx_id: 381797
  lock_mode: X,GAP
  lock_type: RECORD
 lock_table: `learn`.`test`
 lock_index: v1
 lock_space: 1437
  lock_page: 4
   lock_rec: 6
  lock_data: 100
```

后来事务B需要等待先来事务A释放这一区间上的GAP锁，最终锁等待超时。

## 快照读

通过锁定，是可以实现RR事务隔离级别的。关键在于，对于需要改动的数据或者数据范围，不要过早的释放锁，在`事务完成之后`才进行释放。

简单读取操作是S锁，其他操作可以是X锁，S锁和X锁不能同时存在。

[《事务处理原理》](https://book.douban.com/subject/5412835/)一书中提到`两阶段锁定`（2PL，two-phase locking），即一个事务必须在释放它的任何锁之前获得它所有的锁。如果用锁定的方式实现RR，在整个事务中，其他事务的简单读操作也会被阻塞。

InnoDB实现了MVCC的方式，可以实现事务中简单读不需锁定的效果，即实现了快照读。

# 参考

[ANSI SQL-92](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)
[Innodb中的事务隔离级别和锁的关系](http://tech.meituan.com/innodb-lock.html)
[MySQL 加锁处理分析](http://hedengcheng.com/?p=771)
[InnoDB’s gap locks](https://www.percona.com/blog/2012/03/27/innodbs-gap-locks/)
[深入分析事务的隔离级别](http://www.hollischuang.com/archives/943)
[MySQL 5.7 Reference Manual / 14.5.1 InnoDB Locking](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html)

