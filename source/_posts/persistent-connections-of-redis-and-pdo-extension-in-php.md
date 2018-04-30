title: PHP中Redis/MySQL的长连接
date: 2018-04-30 17:29:47
tags: [PHP, PDO, MySQL, Redis]
categories: PHP
---

# TL;DR

PHP 中针对 Redis / MySQL 的长连接是生命周期级别的长连接，对于同一个进程的每一次相同目标的请求都不会释放当前连接对象。而针对 TCP Socket 级别的连接是否已断开，则交给操作系统维持。

使用 PDO 对 MySQL 开启持久连接，要注意 PHP 执行的进程数量，不能超过 MySQL 设定的最大连接数。

上述结论的前提是使用 phpredis 扩展，PHP 版本为 5.4.41。

<!-- more -->

# 背景

假设某同学使用 PHP 开发了一个队列消费 daemon，某天业务压力较大。实现上每次与 Redis 通信都新建连接，同时本地端口范围过小，导致用尽了本地端口，无法建立连接。

这个时候使用 pconnect 可以有效的减少重复建立连接的成本。使用 `ss` 等工具可以看到相关的连接数目。

编写如下脚本进行测试：

```
<?php
	$persist = isset($argv[1]) ? true : false;
	$cmd = "ss -ant | grep ESTAB | awk  'BEGIN{conn=0;}{if($5 == \"127.0.0.1:6379\"){++conn;}}END{print conn}'";

	echo "Before:\n";
	echo shell_exec($cmd) . "\n";

	$rs = [];

	for ($i = 1; $i <= 5; ++$i) {
		$r = new Redis();
		$rs[] = $r;

		if ($persist) {
			$r->pconnect('127.0.0.1', 6379);
		} else {
			$r->connect('127.0.0.1', 6379);
		}
	}

	echo "After:\n";
	echo shell_exec($cmd) . "\n";
```

结果：

```
root@vm:~# php test_redis_connect.php persist
Before:
0

After:
1

root@vm:~# php test_redis_connect.php
Before:
0

After:
5
```

可以看到使用了 pconnect 可以有效减少连接数。

## Redis 连接的实现

直接翻看 phpredis 扩展源码（2.2.7）。

总体而言，phpredis 通过给定的参数产生一个 `RedisSock*`，在这一个结构中包含一个 `php_stream` 结构，利用这一个流式成员，完成与服务端之间的网络通信。

连接时使用 `pconnect` 与 `connect` 的区别在于对于流式对象的管理方式。

### pconnect 实现

#### 参数传递

扩展中连接方法的入口是 `redis.c` 文件中的 `redis_connect` 函数。解析参数列表部分的逻辑为：

```
if (zend_parse_method_parameters(ZEND_NUM_ARGS() TSRMLS_CC, getThis(), "Os|ldsl",
			&object, redis_ce, &host, &host_len, &port,
			&timeout, &persistent_id, &persistent_id_len,
			&retry_interval) == FAILURE) {
	return FAILURE;
}
```

可以看到有一个可选的字符串参数（`|`之后的`s`）`persistent_id`，此处先留意这一个变量，后续会使用到。同时，也可以看到 `pconnect` 方法是可以在调用时指定这个参数的值的。

在后续的 `redis_sock_create` 函数中，会将当前的 `persistent_id` 赋值给返回 `RedisSock` 对象，以供后续创建连接使用。

#### 连接

建立与 Redis 服务器之间的连接工作实际上是由 `redis_sock_server_open` 函数完成的，即在这一个方法中，通过 `php_stream_xport_create` 函数将网络连接创建并赋值给 `RedisSock` 中的 `stream` 变量，实际上，后续的所有网络读写操作，都会通过这一个变量完成。

```
if (redis_sock->persistent) {
  if (redis_sock->persistent_id) {
    spprintf(&persistent_id, 0, "phpredis:%s:%s", host, redis_sock->persistent_id);
  } else {
    spprintf(&persistent_id, 0, "phpredis:%s:%f", host, redis_sock->timeout);
  }
}

redis_sock->stream = php_stream_xport_create(host, host_len, ENFORCE_SAFE_MODE,
						 STREAM_XPORT_CLIENT
						 | STREAM_XPORT_CONNECT,
						 persistent_id, tv_ptr, NULL, &errstr, &err
						);
```

这里判断 `persistent` 的逻辑，就用到了传递进来的 `persistent_id` 参数，如果没有设定，会根据 Redis 服务器的 IP 和当前设定的超时创建出一个字符串作为值。

这个 ID 用于标记这是一个需要保持的对象（持久性资源）。

#### PHP 持久性资源

首先，对于Socket/文件等对象，在 PHP 中都是资源对象，而 PHP 扩展中在实现上会将这一个资源对象记录到哈希表 `EG(regular_list)` 中，通过 `zend_list_delete` 等函数完成资源引用计数的操作，当引用计数为 0 时，认为资源已无效，删除资源。

在 `redis.c` 中的 `redis_connect` 方法中，当通过 redis_sock_create 创建成功资源对象后，通过 `zend_list_insert` 完成了当前资源对象的记录：

```
// redis.c
	redis_sock = redis_sock_create(host, host_len, port, timeout, persistent, persistent_id, retry_interval, 0);

	if (redis_sock_server_open(redis_sock, 1 TSRMLS_CC) < 0) {
		redis_free_socket(redis_sock);
		return FAILURE;
	}

#if PHP_VERSION_ID >= 50400
	id = zend_list_insert(redis_sock, le_redis_sock TSRMLS_CC);
#else
	id = zend_list_insert(redis_sock, le_redis_sock);
#endif
```

```
// zend_list.c
ZEND_API int zend_list_insert(void *ptr, int type TSRMLS_DC)
{
	int index;
	zend_rsrc_list_entry le;

	le.ptr=ptr;
	le.type=type;
	le.refcount=1;

	index = zend_hash_next_free_element(&EG(regular_list));

	zend_hash_index_update(&EG(regular_list), index, (void *) &le, sizeof(zend_rsrc_list_entry), NULL);
	return index;
}
```

回到 phpredis 扩展连接创建过程，在创建过程中的 `_php_stream_xport_create` 函数中，当设定了 `persistent_id` 之后，会首先在另一个全局哈希表 `EG(persistent_list)` 中通过 `persistent_id` 查找到对应的资源对象是否存在，如果存在还会将其尝试注册到 `EG(regular_list)` 中，减少了无谓的创建过程。

```
PHPAPI int php_stream_from_persistent_id(const char *persistent_id, php_stream **stream TSRMLS_DC)
{
	zend_rsrc_list_entry *le;

// 注意
	if (zend_hash_find(&EG(persistent_list), (char*)persistent_id, strlen(persistent_id)+1, (void*) &le) == SUCCESS) {
		if (Z_TYPE_P(le) == le_pstream) {
			if (stream) {
				HashPosition pos;
				zend_rsrc_list_entry *regentry;
				ulong index = -1; /* intentional */

				/* see if this persistent resource already has been loaded to the
				 * regular list; allowing the same resource in several entries in the
				 * regular list causes trouble (see bug #54623) */
				zend_hash_internal_pointer_reset_ex(&EG(regular_list), &pos);
				while (zend_hash_get_current_data_ex(&EG(regular_list),
						(void **)&regentry, &pos) == SUCCESS) {
					if (regentry->ptr == le->ptr) {
						zend_hash_get_current_key_ex(&EG(regular_list), NULL, NULL,
							&index, 0, &pos);
						break;
					}
					zend_hash_move_forward_ex(&EG(regular_list), &pos);
				}
				
				*stream = (php_stream*)le->ptr;
				if (index == -1) { /* not found in regular list */
					le->refcount++;
					(*stream)->rsrc_id = ZEND_REGISTER_RESOURCE(NULL, *stream, le_pstream);
				} else {
					regentry->refcount++;
					(*stream)->rsrc_id = index;
				}
			}
			return PHP_STREAM_PERSISTENT_SUCCESS;
		}
		return PHP_STREAM_PERSISTENT_FAILURE;
	}
	return PHP_STREAM_PERSISTENT_NOT_EXIST;
}
```

随后会检查连接的可用性，由于对 Redis 服务器是一个 TCP 连接，会通过 `xp_socket.c` 文件中的 `php_sockop_set_option` 中 `PHP_STREAM_OPTION_CHECK_LIVENESS` 对应分支的逻辑进行连接可用性的检查：

```
static int php_sockop_set_option(php_stream *stream, int option, int value, void *ptrparam TSRMLS_DC)
{
	int oldmode, flags;
	php_netstream_data_t *sock = (php_netstream_data_t*)stream->abstract;
	php_stream_xport_param *xparam;
	
	switch(option) {
		case PHP_STREAM_OPTION_CHECK_LIVENESS:
			{
				struct timeval tv;
				char buf;
				int alive = 1;

				if (value == -1) {
					if (sock->timeout.tv_sec == -1) {
						tv.tv_sec = FG(default_socket_timeout);
						tv.tv_usec = 0;
					} else {
						tv = sock->timeout;
					}
				} else {
					tv.tv_sec = value;
					tv.tv_usec = 0;
				}

				if (sock->socket == -1) {
					alive = 0;
				} else if (php_pollfd_for(sock->socket, PHP_POLLREADABLE|POLLPRI, &tv) > 0) {
					if (0 >= recv(sock->socket, &buf, sizeof(buf), MSG_PEEK) && php_socket_errno() != EWOULDBLOCK) {
						alive = 0;
					}
				}
				return alive ? PHP_STREAM_OPTION_RETURN_OK : PHP_STREAM_OPTION_RETURN_ERR;
			}
			//...
```

实际上是通过 `poll` 系统调用判断 socket 对应的 fd 是否可读并且 `recv` 系统调用返回可读长度小于等于0实现的。

对于 `EG(persistent_list)` 这一个哈希表，为什么能够实现针对于PHP应用程序持久性的连接呢？答案很简单，这个哈希表只有在 **MSHUTDOWN** 阶段时才会清理：

```
// zend.c
void zend_shutdown(TSRMLS_D) /* {{{ */
{
#ifdef ZEND_SIGNALS
	zend_signal_shutdown(TSRMLS_C);
#endif
#ifdef ZEND_WIN32
	zend_shutdown_timeout_thread();
#endif
	zend_destroy_rsrc_list(&EG(persistent_list) TSRMLS_CC);
	// ...
```

而 `EG(regular_list)` 在 **RSHUTDOWN** 阶段就会被清理（这一阶段进程尚未退出）：

```
// zend.c
void zend_deactivate(TSRMLS_D) /* {{{ */
{
	/* we're no longer executing anything */
	EG(opline_ptr) = NULL;
	EG(active_symbol_table) = NULL;

	zend_try {
		shutdown_scanner(TSRMLS_C);
	} zend_end_try();

	/* shutdown_executor() takes care of its own bailout handling */
	shutdown_executor(TSRMLS_C);

	zend_try {
		shutdown_compiler(TSRMLS_C);
	} zend_end_try();

	zend_destroy_rsrc_list(&EG(regular_list) TSRMLS_CC);
	// ...
```

哪怕是显式的调用了 `close` 方法，也只是向服务器发送 `QUIT` 指令并将当前对象的状态设定为已断开状态，实际上并未清理对应的流：

```
// redis.c
PHP_REDIS_API int redis_sock_disconnect(RedisSock *redis_sock TSRMLS_DC)
{
    if (redis_sock == NULL) {
	    return 1;
    }

    redis_sock->dbNumber = 0;
    if (redis_sock->stream != NULL) {
			if (!redis_sock->persistent) {
				redis_sock_write(redis_sock, "QUIT" _NL, sizeof("QUIT" _NL) - 1 TSRMLS_CC);
			}

			redis_sock->status = REDIS_SOCK_STATUS_DISCONNECTED;
            redis_sock->watching = 0;
			if(redis_sock->stream && !redis_sock->persistent) { /* still valid after the write? */
				php_stream_close(redis_sock->stream);
			}
			redis_sock->stream = NULL;

			return 1;
    }

    return 0;
}
```

## MySQL PDO的连接实现

MySQL PDO的连接实现与 Redis 类似，也是在传递了持久连接的参数后，会将连接对象保存到持久性资源对象的哈希表中。

构建持久化标记ID的方式看起来比 phpredis 扩展稍好一些，用到了 `服务器Host/服务器端口/用户名/密码` 等元素。

```
// pdo_dbh.c
// ...
		if (SUCCESS == zend_hash_index_find(Z_ARRVAL_P(options), PDO_ATTR_PERSISTENT, (void**)&v)) {
			if (Z_TYPE_PP(v) == IS_STRING && !is_numeric_string(Z_STRVAL_PP(v), Z_STRLEN_PP(v), NULL, NULL, 0) && Z_STRLEN_PP(v) > 0) {
				/* user specified key */
				plen = spprintf(&hashkey, 0, "PDO:DBH:DSN=%s:%s:%s:%s", data_source,
						username ? username : "",
						password ? password : "",
						Z_STRVAL_PP(v));
				is_persistent = 1;
			} else {
				convert_to_long_ex(v);
				is_persistent = Z_LVAL_PP(v) ? 1 : 0;
				plen = spprintf(&hashkey, 0, "PDO:DBH:DSN=%s:%s:%s", data_source,
						username ? username : "",
						password ? password : "");
			}
		}
// ...		
```

然而如果启用了持久连接，PDO并没有给出主动清理 `EG(persistent_list)` 的方法，所以，如果要使用这一个特性，需要格外注意不要启动过多的进程，以至于超过 MySQL 设定的最大连接数。

# 小结

PHP 中如果使用了 Redis/MySQL 的持久连接功能，PHP 内核会通过将资源对象存储到 **MSHUTDOWN** 阶段才会清理的全局 HashTable 中，实现针对同一个服务器进行请求时，在 PHP 执行的整个生命周期之内保证了正常情况下不会重新创建连接。

# 相关

+ [PHPBOOK-7.1 函数的参数](https://github.com/walu/phpbook/blob/master/7.1.md)
+ [PHPBOOK-9.2 PHP中的资源类型](https://github.com/walu/phpbook/blob/master/9.2.md)
+ [PHPBOOK-14.1 14.1 流的概览](https://github.com/walu/phpbook/blob/master/14.1.md)
+ [Linux Programmer's Manual-POLL(2)](http://man7.org/linux/man-pages/man2/poll.2.html)




