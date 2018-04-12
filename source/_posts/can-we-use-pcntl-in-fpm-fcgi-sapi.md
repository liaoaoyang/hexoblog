title: 在PHP-FPM中使用pcntl扩展
date: 2018-03-31 22:09:25
tags: [pcntl, PHP]
categories: PHP
---

# TL;DR

在 PHP-FPM 中使用 pcntl 扩展在一定情况下是可行的，但是并不是合理的。

<!-- more -->

# pcntl 扩展

[pcntl](http://php.net/manual/zh/intro.pcntl.php) 扩展主要用于Unix方式的进程创建, 程序执行, 信号处理以及进程的中断。

手册上告知：

> Process Control should not be enabled within a web server environment and unexpected results may happen if any Process Control functions are used within a web server environment.

但是，这个扩展真的不能在 LNMP 环境中使用吗？让我们来验证一下。

# 多进程 fork 验证

pcntl 最常用的功能之一应该就是 fork 新进程了。通过 `pcntl_fork()` 函数可以使用 PHP 进行[多进程](https://www.liaoaoyang.cn/articles/2017/08/22/php-daemon/)应用程序的开发工作。

## 环境

还是使用 Docker 进行环境的准备工作。使用 OpenResty + PHP 5.6.35。

默认的 `php:5.6.35-fpm-alpine3.4` 镜像并没有开启 `pcntl` 扩展，所以需要自行修改镜像，Dockerfile 如下：

```
FROM php:5.6.35-fpm-alpine3.4
RUN apk add --no-cache --virtual .build-deps g++ make autoconf \
	&& docker-php-source extract \
	&& cd /usr/src/php \
	&& docker-php-ext-configure pcntl \
	&& docker-php-ext-install pcntl \
	&& docker-php-ext-enable pcntl \
	&& apk del --purge .build-deps
```

构建镜像 `php:5.6.35-fpm-pcntl-alpine3.4`。

使用 docker-compose 编排 OpenResty 和 PHP-FPM，`docker-compose.yml`如下：

```
version: '3'
networks:
    pcntl_test:
services:
    pcntl_test_phpfpm:
        image: php:5.6.35-fpm-pcntl-alpine3.4
        networks:
            - pcntl_test
        volumes:
            - /data/www/htdocs/pcntl_test:/var/www/html
            - /data/www/logs:/var/www/logs
        expose:
          - "9000"

    pcntl_test_openresty:
        image: openresty/openresty:alpine
        networks:
            - pcntl_test
        depends_on:
            - pcntl_test_phpfpm
        links:
            - pcntl_test_phpfpm
        ports:
            - "9527:80"
        volumes:
            - /data/www/etc/pcntl_test/conf/nginx:/etc/nginx/conf.d/:ro
            - /data/www/logs:/usr/local/openresty/nginx/logs
            - /data/www/htdocs/pcntl_test:/var/www/html
```

## 测试代码

编写一个最简单的 fork 样例：

```
<?php
	if (extension_loaded('pcntl')) {
		echo "true\n";
	}

	for ($i = 0; $i < 5; ++$i) {
		$pid = pcntl_fork();

		if (0 == $pid) {
			echo "child:" . posix_getpid() . "\n";
			exit();
		} else if ($pid > 0) {
			echo "parent:" . posix_getpid() . " {$pid}\n";
		} else {
			echo posix_getpid() . " error\n";
		}
	}
```

## 表现

测试开始。

```
curl http://127.0.0.1:9527/index.php
```

预期的结果是首先会输出 true，毕竟已经加载了 pcntl 扩展，之后会输出各 5 行带有 `parent` 和 `child` 的字符串。形如：

```
true
parent:6 17
parent:6 18
parent:6 19
parent:6 20
parent:6 21
child:21
child:20
child:18
child:19
child:17
```

实际情况是：

```
true
parent:6 17
parent:6 18
parent:6 19
parent:6 20
parent:6 21
```

推测和现实产生了差异，乍一看似乎真的不能 fork 子进程，然而真实的执行情况是什么呢？

### echo 更换为写入文件

我们将 echo 操作换成写入文件:

```
<?php
	$logFn = dirname(__FILE__) . '/log';

	for ($i = 0; $i < 5; ++$i) {
		$pid = pcntl_fork();

		if (0 == $pid) {
			file_put_contents($logFn, "child:" . posix_getpid() . "\n", FILE_APPEND);
			exit();
		} else if ($pid > 0) {
			file_put_contents($logFn, "parent:" . posix_getpid() . " {$pid}\n", FILE_APPEND);
		} else {
			file_put_contents($logFn, posix_getpid() . " error\n", FILE_APPEND);
		}
	}
```

在结果文件中，出现了预期的结果。

那么，可以得出第一个结论：`pcntl` 在一定情况下是可以在 PHP-FPM 中使用的。

### 子进程 echo “未能执行”原因

PHP 使用的是缓冲 IO，存在输入输出缓冲区，很有可能在 PHP-FPM 被 WebServer 断开连接时，还没能刷新各自进程的输入输出缓冲区，导致结果无法输出。

我们在最初代码的基础之上再作一些改动，增加缓冲区刷新操作:

```
<?php
	for ($i = 0; $i < 5; ++$i) {
		$pid = pcntl_fork();

		if (0 == $pid) {
			echo "child:" . posix_getpid() . "\n";
			ob_flush();
			flush();
			exit();
		} else if ($pid > 0) {
			echo "parent:" . posix_getpid() . " {$pid}\n";
			ob_flush();
			flush();
		} else {
			echo posix_getpid() . " error\n";
			ob_flush();
			flush();
		}
	}
```

如此之后在操作系统上才有可能输出子进程的文本。

# 为什么在 PHP-FPM 中使用 pcntl 是不合理的

## 案例

同样是刚才的第一段代码，场景可能是有人期望在 HTTP 请求中使用多进程提升处理能力，编写出类似这样的代码。

我们看看执行前后容器的进程数有何变化:

```
➜ docker top pcntltest_pcntl_test_phpfpm_1 | grep php-fpm | wc -l
3
➜ curl -s http://127.0.0.1:9527/index.php > /dev/null
➜ docker top pcntltest_pcntl_test_phpfpm_1 | grep php-fpm | wc -l
8
```

可以看到进程数增加了5个。这里有人可能会问，不是都已经 exit() 了吗？为什么进程数还增加了呢？

## PHP 生命周期与 PHP-FPM

PHP 的[生命周期](http://www.php-internals.com/book/?p=chapt02/02-01-php-life-cycle-and-zend-engine) TIPI 上已经说明得很清楚了，可以看到调用 `exit()` 后会发生的情况：

> 请求处理完后就进入了结束阶段，一般脚本执行到末尾或者通过调用exit()或die()函数， PHP都将进入结束阶段。和开始阶段对应，结束阶段也分为两个环节，一个在请求结束后停用模块(RSHUTDOWN，对应RINIT)， 一个在SAPI生命周期结束（Web服务器退出或者命令行脚本执行完毕退出）时关闭模块(MSHUTDOWN，对应MINIT)。

也就是说，`exit()` 只是到达了执行的结束阶段，并不是退出进程，退出进程的时机由 Zend 引擎决定。

PHP-FPM 为了提高处理能力，fork 出进程执行 PHP 代码后并不会立刻退出，进程仍会运行一定时间才会退出。

综上，在 PHP-FPM 中使用 pcntl_fork() 期望完成并行操作不是一个合理的行为。

