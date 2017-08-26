title: 使用Nagios监控Redis-按内存使用率监控
date: 2016-01-05 19:09:55
categories: Learning
tags: [Nagios,Redis]
---

# 概述

本文是 [使用Nagios监控Redis][1] 的补充记录。

根据前期选用的插件，通过直接针对 `used_memory_rss` 设定监控阈值完成监控，而这一阈值的问题在于是使用字节数的表示的，如果要按照`36GB`的`70%`设定阈值监控，就需要将监控的值设定为(结果已四舍五入)：

```
36 * 1024 * 1024 * 1024 * 0.7 = 27058293965
```

这一数字对于config文件来说，以及监控用户来说并不友好。

<!-- more -->

# 解决

插件本身也提供了直接通过使用率进行监控的使用方式，我们所需要做的就是告知插件对应实例的最大内存容量，即：

```
-M, --total_memory=NUM[B|K|M|G]
   Amount of memory on a system for memory utilization calculations above.
   If it does not end with K,M,G then its assumed to be B (bytes)
```

这一参数可以使用人类友好的单位表示方式表示数值。

同时设定阈值需要使用 `-m` 参数，即：

```
-m, --memory_utilization=[WARN,CRIT]
   This calculates percent of total memory on system used by redis, which is
      utilization=redis_memory_rss/total_memory*100.
   Total_memory on server must be specified with -M since Redis does not report
   it and can use maximum memory unless you enabled virtual memory and set a limit
   (I plan to test this case and see if it gets reported then).
   If you specify -m by itself, the plugin will just output this info,
   with '-f' it will also include this in performance data. You can also specify
   parameter values which are interpreted as WARNING and CRITICAL thresholds.
```

即通过逗号分隔 `warn` 与 `critical` 两个级别的报警阈值。如同手册所说的，必须与 `-M` 同时使用。

我们所需要做的就是在 `/nip/etc/objects/commands.cfg` （`nip`即`nagios安装目录`的缩写，根据实际情况决定）中新增或者修改一个命令：

```
define command {
    command_name    check_redis_with_max_mem
    command_line    $USER1$/check_redis.pl -H $HOSTADDRESS$ -p $ARG1$ -M $ARG2$ -m $ARG3$ -a $ARG4$ -w $ARG5$ -c $ARG6$ -f
}
```

同时在 `/nip/etc/conf/services.cfg` 中使用这一命令：

```
check_command           check_redis_with_max_mem!6379!36G!70,90!'used_memory_human,used_memory_peak_human,used_memory_rss'!~,~,~!~,~,~
```

当然，也可以修改插件完成。

以上。

[1]: http://www.liaoaoyang.com/articles/2015/11/27/monitor-redis-by-nagios/


