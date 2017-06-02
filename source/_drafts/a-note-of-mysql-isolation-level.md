title: MySQL事务隔离级别
date: 2017-05-29 22:09:34
tags: [MySQL]
category: LAMP/LNMP
---

# ANSI SQL-92标准

事务隔离级别在[ANSI SQL-92](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)标准中有了明确的定义，即我们熟知的`READ UNCOMMITTED`/`READ COMMITTED`/`REPEATABLE READ`/`SERIALIZABLE`。级别由低到高，并发度依次下降。

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

**不可重复读（Non-repeatable read）**

```
2) P2 ("Non-repeatable read"): SQL-transaction T1 reads a row. SQL-
   transaction T2 then modifies or deletes that row and performs
   a COMMIT. If T1 then attempts to reread the row, it may receive
   the modified value or discover that the row has been deleted.
```

两个或多个事务中有一个事务更改了数据，只有提交之后，才会读取到数据的改动。

**幻读（Phantom）**

```
 3) P3 ("Phantom"): SQL-transaction T1 reads the set of rows N
    that satisfy some <search condition>. SQL-transaction T2 then
    executes SQL-statements that generate one or more rows that
    satisfy the <search condition> used by SQL-transaction T1. If
    SQL-transaction T1 then repeats the initial read with the same
    <search condition>, it obtains a different collection of rows.
```

幻读发生的前提是一个事务改动并提交处于其他某一事务查询条件范围内，并且其他事务重复同一条件进行查询。

