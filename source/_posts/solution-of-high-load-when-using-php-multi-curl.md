title: 降低使用PHP curl_multi* 时的 load
date: 2018-06-01 09:51:09
tags: [cURL,PHP, multicurl]
categories: PHP
---

#TL;DR

如果选择通过 `curl_multi*` 函数并行发起请求，需要在使用 `curl_multi_select` 返回 -1 时增加休眠时间以降低 load。形如（代码来自 Guzzle）：

```
if ($this->active &&
    curl_multi_select($this->_mh, $this->selectTimeout) === -1
) {
    usleep(250);
}
```

<!-- solution-of-high-load-when-using-php-multi-curl -->
<!-- more -->


