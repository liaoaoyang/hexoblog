title: PHP中curl设置毫秒级超时的问题
date: 2017-06-16 22:16:20
category: PHP
tags: [PHP]
---

# TL;DR

PHP5/7加上7.19的libcurl，设置低于1s的超时时间时，`curl_exec`仍会执行超过1s以上。原因在于此版本的libcurl实现逻辑上以1000ms作为`curl_exec`中`poll`系统调用的超时值。

<!-- more -->

# 问题

某些HTTP接口响应时间可能因为种种原因会引起响应时间延长的问题，在这种情况下，从性能、体验角度考虑，设置调用方的超时时间成为必要。

从[手册](http://php.net/manual/zh/function.curl-setopt.php)上得知：

| 选项	 | 设置value为 | 备注 |
| --- | --- | --- |
| CURLOPT_TIMEOUT_MS | 设置cURL允许执行的最长毫秒数。 如果 libcurl 编译时使用系统标准的名称解析器（ standard system name resolver），那部分的连接仍旧使用以秒计的超时解决方案，最小超时时间还是一秒钟。 | 在 cURL 7.16.2 中被加入。从 PHP 5.2.3 起可使用。 |

在符合版本条件的服务器上用PHP内置的HTTP服务器模拟一个时长超过1s的接口：

```
[root@vm curl_test]# cat slow.php
<?php
        $sleep = isset($_GET['sleep']) ? $_GET['sleep'] : 1;
        sleep($sleep);
        echo "SLOW RESPONSE\n";
[root@vm curl_test]# php -S 127.0.0.1:8000
```

通过如下代码访问这一接口：

```
<?php
   $timeout_ms = isset($argv[1]) ? $argv[1] : 200;
   $sleep = isset($argv[2]) ? $argv[2] : 1;
   $url = "http://127.0.0.1:8000/slow.php?sleep={$sleep}";
   $ch = curl_init();
   curl_setopt($ch, CURLOPT_URL, $url);
   curl_setopt($ch, CURLOPT_NOSIGNAL, 1);
   curl_setopt($ch, CURLOPT_TIMEOUT_MS, $timeout_ms);
   $start = microtime(true);
   curl_exec($ch);
   $end = microtime(true);
   $time = ($end - $start) * 1000;
   curl_close($ch);

   echo "sleep:   {$sleep}s\n";
   echo "timeout: {$timeout_ms}ms\n";
   echo "start:   {$start}\n";
   echo "end:     {$end}\n";
   echo "used:    " . ($end - $start) * 1000  . "ms\n";
```
通过如下命令，设定了200ms的超时时间，同时还有1s的接口延迟响应时间。

```
[root@vm curl_test]# php curl_test.php 200 1
```

预料中，客户端应该会在200ms左右的时间结束程序，然而实际执行输出让人感到困惑：

```
[root@vm curl_test]# php curl_test.php 200 2
sleep:   2s
timeout: 200ms
start:   1497665724.8918
end:     1497665725.8937
used:    1001.8589496613ms
```

客户端运行了1000ms+的时间。


# 分析

## 查询手册

从PHP手册中查询到的结果，字面意思并没有太多的歧义，使用方法上也并没有明显错误，下面可以从运行时的状态开始分析。

## Strace跟踪运行过程

通过`strace`查看运行过程中的系统调用：

```
strace -o curl_strace.log -ttt php curl_test.php 200 1
```

为了方便分析，将输出记录到`curl_strace.log`文件之中，并记录各个系统调用产生时的时间戳。

关键的网络请求部分系统调用记录如下：

```
     1  1497665724.892491 connect(3, {sa_family=AF_INET, sin_port=htons(8000), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
     2  1497665724.892632 clock_gettime(CLOCK_MONOTONIC, {17533335, 287262686}) = 0
     3  1497665724.892660 poll([{fd=3, events=POLLOUT|POLLWRNORM}], 1, 200) = 1 ([{fd=3, revents=POLLOUT|POLLWRNORM}])
     4  1497665724.892689 getsockopt(3, SOL_SOCKET, SO_ERROR, [0], [4]) = 0
     5  1497665724.892714 clock_gettime(CLOCK_MONOTONIC, {17533335, 287343820}) = 0
     6  1497665724.892737 clock_gettime(CLOCK_MONOTONIC, {17533335, 287371368}) = 0
     7  1497665724.892767 clock_gettime(CLOCK_MONOTONIC, {17533335, 287395792}) = 0
     8  1497665724.892789 clock_gettime(CLOCK_MONOTONIC, {17533335, 287418504}) = 0
     9  1497665724.892811 clock_gettime(CLOCK_MONOTONIC, {17533335, 287439788}) = 0
    10  1497665724.892849 sendto(3, "GET /slow.php?sleep=2 HTTP/1.1\r\n"..., 69, MSG_NOSIGNAL, NULL, 0) = 69
    11  1497665724.893017 poll([{fd=3, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 0) = 0 (Timeout)
    12  1497665724.893046 poll([{fd=3, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 0) = 0 (Timeout)
    13  1497665724.893070 clock_gettime(CLOCK_MONOTONIC, {17533335, 287699255}) = 0
    14  1497665724.893093 clock_gettime(CLOCK_MONOTONIC, {17533335, 287722637}) = 0
    15  1497665724.893117 clock_gettime(CLOCK_MONOTONIC, {17533335, 287746751}) = 0
    16  1497665724.893139 poll([{fd=3, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 1000) = 0 (Timeout)
    17  1497665725.893450 poll([{fd=3, events=POLLIN|POLLPRI|POLLRDNORM|POLLRDBAND}], 1, 0) = 0 (Timeout)
    18  1497665725.893480 clock_gettime(CLOCK_MONOTONIC, {17533336, 288109801}) = 0
    19  1497665725.893505 clock_gettime(CLOCK_MONOTONIC, {17533336, 288133751}) = 0
    20  1497665725.893545 clock_gettime(CLOCK_MONOTONIC, {17533336, 288174671}) = 0
    21  1497665725.893576 gettimeofday({1497665725, 893582}, NULL) = 0
    22  1497665725.893602 close(3)              = 0
```

`fd=3`的文件描述符即用于处理网络请求。在上述步骤的第10行发出请求之后，调用了`poll`系统调用来检验fd是否可以读写。查阅[手册](http://man7.org/linux/man-pages/man2/poll.2.html)得知，poll系统调用的声明为：

```
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

第三个参数即以毫秒为单位的等待超时时间。

请注意16行的系统`poll`调用，超时时间足足设定了1000ms，之后程序执行结束退出，那么这个1000ms的超时设定，应该就是程序没有在200ms左右的时间退出的主要原因。

## PHP curl扩展

PHP中的curl函数在curl的扩展中定义，如果怀疑对运行的方式产生了怀疑，可以查看扩展源码。

在PHP源码的`ext/curl`目录下的`interface.c`文件中，可以看到`curl_exec`方法的定义：

```
/* {{{ proto bool curl_exec(resource ch)
     Perform a cURL session */
  PHP_FUNCTION(curl_exec)
  {
      CURLcode    error;
      zval        *zid;
      php_curl    *ch;

      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "r", &zid) == FAILURE) {
          return;
      }

      ZEND_FETCH_RESOURCE(ch, php_curl *, &zid, -1, le_curl_name, le_curl);

      _php_curl_verify_handlers(ch, 1 TSRMLS_CC);

      _php_curl_cleanup_handle(ch);

      error = curl_easy_perform(ch->cp);
```

可以看到，实际上PHP中的`curl_exec`调用的是`libcurl`中的`curl_easy_perform`这一函数执行的网络请求。

对于超时的问题，需要深入到libcurl之中了。

## libcurl 7.19

通过`curl --version`命令可以获取cUrl的版本，之后在GitHub是哪个clone[源码repo](https://github.com/curl/curl)，checkout对应版本（`7.19.x`）即可。

可以看到`curl_easy_perform`在libcurl源码目录中`lib/easy.c`中被定义了。主要执行步骤如下：

1.执行`Curl_perform`函数
2.在`Curl_perform`中完成连接（`connect_host`函数）与传输数据操作（`Transfer`函数）
3.在`Transfer`中对第一次poll操作设定0ms的超时，之后均设定为1000ms的超时，参见代码：

```
    if(first &&
       ((fd_read != CURL_SOCKET_BAD) || (fd_write != CURL_SOCKET_BAD)))
      /* if this is the first lap and one of the file descriptors is fine
         to work with, skip the timeout */
      timeout_ms = 0;
    else
      timeout_ms = 1000;

    switch (Curl_socket_ready(fd_read, fd_write, timeout_ms)) {
    case -1: /* select() error, stop reading */
#ifdef EINTR
      /* The EINTR is not serious, and it seems you might get this more
         often when using the lib in a multi-threaded environment! */
      if(SOCKERRNO == EINTR)
        continue;
#endif
      return CURLE_RECV_ERROR;  /* indicate a network problem */
    case 0:  /* timeout */
    default: /* readable descriptors */

      result = Curl_readwrite(conn, &done);
      /* "done" signals to us if the transfer(s) are ready */
      break;
    }

    ...
```

4.`Curl_socket_ready`方法中会根据传入的超时时间即变量`timeout_ms`的值设定`poll`系统调用的超时值，结合strace日志，即`11`与`16`行的`poll`操作。
5.第一次超时为0ms时，`poll`会立刻返回，显然没有socket可以读写，会走入`switch`语句的`case 0`的分支，调用`Curl_readwrite`函数；在第二次以及之后的执行中，超时会恒定在1000ms，在可读写之后，会走入`switch`语句的`default`的分支，也会调用`Curl_readwrite`函数。
6.在`Curl_readwrite`函数中会再次调用`Curl_socket_ready`方法，但是这次传入的超时时间则固定为0ms了：

```
/*
 * Curl_readwrite() is the low-level function to be called when data is to
 * be read and written to/from the connection.
 */
CURLcode Curl_readwrite(struct connectdata *conn,
                        bool *done)
{
  struct SessionHandle *data = conn->data;
  struct SingleRequest *k = &data->req;
  CURLcode result;
  int didwhat=0;

  curl_socket_t fd_read;
  curl_socket_t fd_write;
  int select_res = conn->cselect_bits;

  conn->cselect_bits = 0;

  /* only use the proper socket if the *_HOLD bit is not set simultaneously as
     then we are in rate limiting state in that transfer direction */

  if((k->keepon & KEEP_RECVBITS) == KEEP_RECV) {
    fd_read = conn->sockfd;
  } else
    fd_read = CURL_SOCKET_BAD;

  if((k->keepon & KEEP_SENDBITS) == KEEP_SEND)
    fd_write = conn->writesockfd;
  else
    fd_write = CURL_SOCKET_BAD;

   if(!select_res) { /* Call for select()/poll() only, if read/write/error
                         status is not known. */
       select_res = Curl_socket_ready(fd_read, fd_write, 0);
   }

	...
```

这里的`Curl_socket_ready`就是strace日志中`12`与`17`行两次`poll`操作记录的来源。
7.在`Curl_readwrite`中会判断当前已执行时间是否已经超过了设定的超时时间：

```
 if(data->set.timeout &&
     (Curl_tvdiff(k->now, k->start) >= data->set.timeout)) {
    if(k->size != -1) {
      failf(data, "Operation timed out after %ld milliseconds with %"
            FORMAT_OFF_T " out of %" FORMAT_OFF_T " bytes received",
            data->set.timeout, k->bytecount, k->size);
    } else {
      failf(data, "Operation timed out after %ld milliseconds with %"
            FORMAT_OFF_T " bytes received",
            data->set.timeout, k->bytecount);
    }
    return CURLE_OPERATION_TIMEDOUT;
  }
```

如果已超过则，返回已超时。

## 小结

综上，老版本curl设定了ms级别超时但是仍然会执行s级别的问题在于curl的`poll`超时设定被固定在1000ms之上，造成了这一问题。

## 解决

可以考虑升级libcurl，或者在编译的PHP中使用高版本的libcurl。

```
[root@vm curl_test]# curl --version
curl 7.55.0-DEV (x86_64-unknown-linux-gnu) libcurl/7.55.0-DEV OpenSSL/1.0.1e zlib/1.2.3
Release-Date: [unreleased]
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp smb smbs smtp smtps telnet tftp
[root@vm curl_test]# /opt/softwares/php/7.1.6_newlibcurl/bin/php curl_test.php 200 1
sleep:   1s
timeout: 200ms
start:   1497667324.1965
end:     1497667324.3967
used:    200.21796226501ms
```

以上。

