title: 降低使用PHP curl_multi_* 时的 load
date: 2018-06-01 09:51:09
tags: [cURL,PHP, curlmulti]
categories: PHP
---

# TL;DR

如果选择通过 `curl_multi_*` 函数并行发起请求，需要在使用 `curl_multi_select` 返回 -1 时增加休眠时间以降低 load。形如（代码来自 Guzzle）：

```
if ($this->active &&
    curl_multi_select($this->_mh, $this->selectTimeout) === -1
) {
    usleep(250);
}
```

使用的软件版本为：

+ PHP 5.4.41
+ libcurl 7.19
+ Guzzle 5.3

<!-- solution-of-high-load-when-using-php-multi-curl -->
<!-- more -->

# curl_multi_*

如果要并发的发起请求，PHP 可以通过多进程发起请求的方式实现，每个请求由一个进程执行，然而这一方式开发以及运行成本较高，带来了繁重的进程管理以及 IPC 的编码工作。

进程是重量级的，而连接较之进程是轻量级的，结合IO多路复用可以让并发发起网络请求的成本降低，`curl_multi*` 这一组函数就是 PHP 基于 `libcurl` 封装的并发发起请求的工具。

网络上一些博文声称这一族函数是多线程发起请求，然而[libcurl官网](https://curl.haxx.se/libcurl/c/libcurl-tutorial.html)的文档中的 **The multi Interface** 部分说得很清楚，与这些博文相反：

> The multi interface, on the other hand, allows your program to transfer multiple files in both directions at the same time, without forcing you to use multiple threads. The name might make it seem that the multi interface is for multi-threaded programs, but the truth is almost the reverse. The multi interface allows a single-threaded application to perform the same kinds of multiple, simultaneous transfers that multi-threaded programs can perform.

编码上，通过 [`curl_multi_init`](http://php.net/manual/en/function.curl-multi-init.php) 创建并行请求处理对象，加入多个普通的 curl 对象；通过 [`curl_multi_exec`](http://php.net/manual/en/function.curl-multi-exec.php) 按照内置的状态机建立连接，传输数据；使用 [`curl_multi_select`](http://php.net/manual/en/function.curl-multi-select.php) 获取活跃的连接；之后尝试读取当中活跃连接的信息，记录对端返回的数据。

还是以 Guzzle 作为样板，选出最重要的执行过程，参见文件 `guzzlehttp/ringphp/src/Client/CurlMultiHandler.php`：

```
/**
 * Runs until all outstanding connections have completed.
 */
public function execute()
{
    do {

        if ($this->active &&
            curl_multi_select($this->_mh, $this->selectTimeout) === -1
        ) {
            // Perform a usleep if a select returns -1.
            // See: https://bugs.php.net/bug.php?id=61141
            usleep(250);
        }

        // Add any delayed futures if needed.
        if ($this->delays) {
            $this->addDelays();
        }

        do {
            $mrc = curl_multi_exec($this->_mh, $this->active);
        } while ($mrc === CURLM_CALL_MULTI_PERFORM);

        $this->processMessages();

        // If there are delays but no transfers, then sleep for a bit.
        if (!$this->active && $this->delays) {
            usleep(500);
        }

    } while ($this->active || $this->handles);
}

private function processMessages()
{
    while ($done = curl_multi_info_read($this->_mh)) {
        $id = (int) $done['handle'];

        if (!isset($this->handles[$id])) {
            // Probably was cancelled.
            continue;
        }

        $entry = $this->handles[$id];
        $entry['response']['transfer_stats'] = curl_getinfo($done['handle']);

        if ($done['result'] !== CURLM_OK) {
            $entry['response']['curl']['errno'] = $done['result'];
            if (function_exists('curl_strerror')) {
                $entry['response']['curl']['error'] = curl_strerror($done['result']);
            }
        }

        $result = CurlFactory::createResponse(
            $this,
            $entry['request'],
            $entry['response'],
            $entry['headers'],
            $entry['body']
        );

        $this->removeProcessed($id);
        $entry['deferred']->resolve($result);
    }
}
```

# curl_multi_select 之后的 usleep()

Guzzle 作为优秀的 PHP HTTP 客户端，实现上在 `curl_multi_select` 返回 **-1** 时会休眠，这是降低 load 的一个看起来比较奇怪的关键操作。

## 不带 sleep/usleep 的实现导致的问题

有部分的代码实现上并没有在这一情况下进行休眠，导致的问题是 CPU 使用率过高，导致服务器 load 上升。

load 与 CPU 使用时间有关系，当 CPU 繁忙时，服务器 load 会升高。常见的让服务器 CPU 飙升的操作就是死循环或者执行时间极短的不带休眠的循环。

例如以下的实现：

```
$running = true;
while ($running)
{
    if (CURLM_CALL_MULTI_PERFORM == curl_multi_exec($mh, $running)) {
        continue;
    }

    curl_multi_select($mh, $selectTimeout);
    
    while ($ret = curl_multi_info_read($mh, $id)) {
        // ETC     
    }
}
```

就会在服务器上就会引起服务器 load 上升。

## PHP curl_multi_select 实现

不带休眠的实现引起服务器 load 上述，要从 PHP `curl_multi_select` 的实现说起。

PHP 内核源码中自带了 curl 扩展的源码，在 `ext/curl/multi.c` 文件中可以看到这一函数的实现：

```
/* {{{ proto int curl_multi_select(resource mh[, double timeout])
   Get all the sockets associated with the cURL extension, which can then be "selected" */
PHP_FUNCTION(curl_multi_select)
{
	zval           *z_mh;
	php_curlm      *mh;
	fd_set          readfds;
	fd_set          writefds;
	fd_set          exceptfds;
	int             maxfd;
	double          timeout = 1.0;
	struct timeval  to;

	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "r|d", &z_mh, &timeout) == FAILURE) {
		return;
	}

	ZEND_FETCH_RESOURCE(mh, php_curlm *, &z_mh, -1, le_curl_multi_handle_name, le_curl_multi_handle);

	_make_timeval_struct(&to, timeout);
	
	FD_ZERO(&readfds);
	FD_ZERO(&writefds);
	FD_ZERO(&exceptfds);

	curl_multi_fdset(mh->multi, &readfds, &writefds, &exceptfds, &maxfd);
	if (maxfd == -1) {
		RETURN_LONG(-1);
	}
	RETURN_LONG(select(maxfd + 1, &readfds, &writefds, &exceptfds, &to));
}
/* }}} */
```

文档上提及：

> On success, returns the number of descriptors contained in the descriptor sets. This may be 0 if there was no activity on any of the descriptors. On failure, this function will return -1 on a select failure (from the underlying select system call).

即 -1 只在 `select` 系统调用失败时返回。 

由于请求是用户主动发起的，select 系统调用的监听集合需要通过某种方式获取，在这个编程模型下，`curl_multi_fdset` 就是产生监听集合的方法。

然而在实现中除去 PHP 函数的一些基本操作，可以看到返回 -1 的情况还会在 libcurl 的库函数 `curl_multi_fdset` 的 `maxfd` 变量被修改为 -1 后出现。

[libcurl 的 curl_multi_fdset](https://curl.haxx.se/libcurl/c/curl_multi_fdset.html) 文档里提到的一段话值得注意：

> If no file descriptors are set by libcurl, max_fd will contain -1 when this function returns. Otherwise it will contain the highest descriptor number libcurl set. When libcurl returns -1 in max_fd, it is because libcurl currently does something that isn't possible for your application to monitor with a socket and unfortunately you can then not know exactly when the current action is completed using select(). You then need to wait a while before you proceed and call curl_multi_perform anyway. How long to wait? Unless curl_multi_timeout gives you a lower number, we suggest 100 milliseconds or so, but you may want to test it out in your own particular conditions to find a suitable value.

即 max_fd 返回 -1 时，需要主动休眠 100ms 或者根据实际情况决定。

在 PHP 执行过程中，我们无法判断是 select 系统调用返回的 -1 还是 `curl_multi_fdset` 的 `max_fd` 返回的 -1。

而当 `curl_multi_fdset` 的 `max_fd` 返回 -1 时，说明 fd 集合中没有可以读写的 fd，应当避免频繁轮询这个函数，导致占满 CPU。

# 结论

推广到 PHP 扩展方法 `curl_multi_select` 的调用，可以得知当返回 -1 时，不应该继续轮询请求 `curl_mutli_exec`，应当主动休眠，降低CPU占用。


