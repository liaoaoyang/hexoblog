title: 基于数据库实现幂等接口
date: 2017-03-15 09:32:13
tags: [API]
category: 系统
---

# TL;DR

通过唯一编号确定同一请求，没有唯一编号的自行生成。
数据库记录操作状态，数据库事务保证数据一致性。

# 概述

通过HTTP API进行通信的系统，在支付或者只允许操作一次的相关场景中，对接口的幂等性有严格要求。

接口的幂等性体现在：

> 请求执行成功所得到的结果与次数无关

如果接口没有实现幂等性，对于转账的应用场景：

### A. 正常转账

1. A账户金额为￥200，B账户金额￥100
2. A调用API向B转账￥100，接口调用成功
3. A账户金额为￥100，B账户金额￥200

在这一场景下，整个流程正常，接口无论是否实现幂等性与否都对执行结果没有影响。

### B. 转账重试

1. A账户金额为￥200，B账户金额￥100
2. A调用API向B转账￥100，接口调用失败
3. A重试请求，本次成功
4. A账户金额为￥100，B账户金额￥200

在这一场景，重试成功的情况下，接口无论是否实现幂等性与否都对执行结果没有影响。

### C. 转账操作超时后重试

1. A账户金额为￥200，B账户金额￥100
2. A调用API向B转账￥100，接口调用超时，A不了解本次转账是否完成，服务端实际转账成功
3. A重试请求，本次成功
4. A账户金额为￥0，B账户金额￥300

在实际应用场景中，接口超时的情况并不罕见，接口超时不代表操作失败，可能存在的情况就有操作实际成功然而并没有返回数据。在这样一个场景之下，接口没有实现幂等性造成重复操作，对于系统的可靠性来说是不可容忍的。

### D. 重复的转账操作

1. A账户金额为￥200，B账户金额￥100
2. A调用API向B转账￥100，由于A的误操作发出了第二次同样的请求
3. A发出的两次请求均成功
4. A账户金额为￥0，B账户金额￥300

两次操作同时发出，并且都成功，接口没有实现幂等的情况下，两次转账操作都会成功，但是对于用户A来说，实际上这是同一次的转账意愿。

以上的场景还是在A与B账户均存在于同一个资源（一般为数据库）之上的操作，如果A与B账户处于两个资源，场景还会更加的复杂。

由上述的场景可以看出，实现接口幂等性的两个方向在于：

+ 定义同一次操作
+ 拒绝重复操作

# 实现

利用数据库实现上述两个需求十分方便。

> 定义同一次操作

使用数据库实现发号器，为每一次请求生成唯一编号

> 拒绝重复操作

通过数据库事务以及唯一索引，以请求编号作为依据，保证同一时间内只有一个请求进行操作，经过先查询后操作的方式，已完成操作不执行更改逻辑，保证请求值执行一次。

以MySQL为例，针对需要实现幂等的操作，可以建立如下的数据表：

```
CREATE TABLE `idempotent_op` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `op_no` char(32) NOT NULL DEFAULT '',
  `status` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `op_no` (`op_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

其中`op_no`列存在唯一索引。

针对上述转账的场景，设定A与B都处于同一数据库中，可以用如下伪代码表示上述的转账操作：

```

# 非事务读，判断是否当前请求已处理，减少事务数量
IF SELECT op_no_A IN idempotent_op AND (status OF op_no_A) == 'FINISHED' THEN
	RETURN 'TRANSFER HANDLED'

BEGIN TRANSACTION

# 在事务读中使用X锁，保证只有一个请求完成本次转账
IF SELECT op_no_A IN idempotent_op FOR UPDATE THEN
	IF (status OF op_no_A) == 'FINISHED' THEN
		ROLLBACK
		RETURN 'TRANSFER HANDLED'
	ELSE
		IF DO_TRANSFER(A, B, money) == SUCCESS THEN
			SET status OF op_no_A FROM 'CREATED' TO 'FINISHED'
			COMMIT
		ELSE
			ROLLBACK
			RETURN 'TRANSFER FAILED'
ELSE
	# 数据库唯一键保证同时到达的多个新请求只有一个可以进行转账操作
	result = INSERT (op_no=op_no_A, status='CREATE') INTO idempotent_op

	IF result = FAIL THEN
		ROLLBACK
		RETURN 'TRANSFER INSERT RECORD FAILED'

	IF DO_TRANSFER(A, B, money) == SUCCESS THEN
		SET status OF op_no_A FROM 'CREATED' TO 'FINISHED'
		COMMIT
	ELSE
		ROLLBACK
		RETURN 'TRANSFER FAILED'
```

以上。

