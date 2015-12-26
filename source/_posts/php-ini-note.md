date: 2014-05-23 00:00:00
title: PHP配置问题汇总
categories: LAMP/LNMP
tags: [PHP]
---

## 1.短记号问题

一般来说PHP代码需要以`<?php`开始，以`?>`，但是PHP也支持短记号`<?`以及`?>`。

在使用Yaf的过程中，在mac本机调试的时候发现模板不能正确显示，发现模板中都是短记号的形式，形如：

```
&lt;div id="taskIdHolder" data-id="&lt;?php echo $task_id; ?&gt;"&gt;
&lt;/div&gt;
```

这类模板会不能正确地显示，解决的办法自然就是在php.ini中开启对短记号的支持：

```
short_open_tag = On
```
