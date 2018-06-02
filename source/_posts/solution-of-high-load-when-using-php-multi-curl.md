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


