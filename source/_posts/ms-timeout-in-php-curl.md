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
[root@vm curl_test]# php curl_test.php 200 1
SLOW RESPONSE
sleep:   1s
timeout: 200ms
start:   1497626295.4526
end:     1497626296.456
used:    1003.4101009369ms
```

客户端运行了1000ms+的时间。


