date: 2014-02-24 00:00:00
title: BackWPup不能备份的一种可能性
categories: Try
tags: [backwpup]
---

[BackWPup][1]是一款非常赞的WP备份插件，可以自动增加定时任务实现对Wordpress的备份，并且可以指定目录文件以及数据库的具体备份规则，并且提供了备份到Dropbox的功能。不想折腾的话用这款插件是非常适合的。

由于[VPS][2]性能比较一般，为了能够提高响应速度，使用了Wordpress的[Memcached][3]插件做缓存操作，这样可以减少数据库的查询操作，经过测试提升速度还是比较明显的，用法参见[我爱水煮鱼][4]。

在没有一次的自动备份完成后，由于新增了一些配置，决定自己手动备份一下，点击 *立即执行* 手动备份计划时，却被通知“每周备份”正在进行中（ ***每周备份*** 是我设定的一个备份计划的名称），无法进行备份操作。

简单看了一下BackWPup的代码，发现这里读取的是缓存中的 ***site-options*** 类型的数据，而在Memcached插件的readme中找到可能缓存的数据组中包含了这一数据，并且在Memcached插件的object-cache.php文件中，发现默认超时为0，即缓存的数据不失效，那么应该是上一次写入缓存标记BackWPup正在执行的数据尚未过期，所以引起的这个问题。

解决方法目前就是直接用 ***kill -9*** 杀死memcached的进程，之后手动执行备份操作。

这样做坏处就在于缓存都没了，后面打算想想办法如何能够针对这一插件做一些特殊处理。

[1]: http://wordpress.org/plugins/backwpup
[2]: https://www.kvmla.com
[3]: http://wordpress.org/plugins/memcached
[4]: http://blog.wpjam.com/m/wordpress-memcached
