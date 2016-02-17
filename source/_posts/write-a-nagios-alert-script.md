title: 自定义Nagios报警脚本
date: 2016-02-17 21:53:48
tags: [Nagios]
---

# 概述

Nagios可以使用邮件报警，但是如果使用IM软件提供的API进行报警的话，时效性上来说必然是会更好的。

另外如果使用自定义的报警脚本，可以针对报警做更多的事情，譬如限频，异步发送，记录入库等操作。

总而言之就是可以拥有更为灵活的工具。

# 关键点

## 开发语言

众多的Nagios插件均使用Perl编写，如 [监控Redis][1] 中使用到的工具。

选用Perl语言对于我来说并不是一个好的选择，如果选用了Perl，那么在主要使用PHP的环境下，不能方便的重复利用已有框架的各种工具，同时从语言的熟悉程度上来说，自然是PHP胜过Perl。

综上，选用PHP作为插件开发语言。

选用PHP作为Nagios的开发语言，需要在脚本的首行指定 [Shebang][2] ，即指定PHP可执行程序的绝对路径，形如：

```
#!/usr/local/server/bin/php
```

Shebang之下仍然需要使用 `<?php` 标签。

同时，为了能让Nagios能直接执行报警插件，需要赋予可执行权限：

```
chmod a+x /path/to/your/nagios/plugin
```

## 脚本输入输出

### 输入

脚本的输入方式与普通的命令行工具并无太多区别，可以使用 `getopt` 来获取输入的参数。

由于需要实现限频，以及针对主机和服务做一些处理，定义了如下的参数：

|参数|说明|
|---|---|
|`u`|通知用户|
|`m`|报警消息体|
|`r`|频率限制值，秒为单位|
|`h`|主机|
|`p`|端口|
|`l`|通知等级，即OK/CRITICAL/WARNING/UNKNOW等|
|`n`|通知类型，即PROBLEM/CUSTOM等|


```php
$options = getopt('u:m:r:h:s:l:n:');

$warning_users = isset($options['u']) ? $options['u'] : '';
$message = isset($options['m']) ? $options['m'] : '';
$rate = isset($options['r']) ? $options['r'] : 0;
$host = isset($options['h']) ? $options['h'] : '';
$service = isset($options['s']) ? $options['s'] : '';
$level = isset($options['l']) ? $options['l'] : NOTIFICATION_LEVEL_NOTICE;
$notification_type = isset($options['n']) ? $options['n'] : NOTIFICATION_TYPE_PROBLEM;
```

限频的目的在于防止接收过多信息，使得必须处理的信息无法及时被发现。

而针对服务恢复正常的信息，不需要限频。

### 输出

Nagios的插件通过返回值确定这次检验的状态，即：

|Exit code|状态|
|---|---|
|0|OK|
|1|WARNING|
|2|CRITICAL|
|3|UNKNOWN|

不过由于是报警脚本，不妨直接让脚本返回0吧。

## 自定义变量

Nagios在调用编写的报警脚本时，通过定义好的command格式，完成传参。

对应到每一个联系人，需要有变量告知联系人的联系方式，针对IM，比如QQ，自然是QQ号。查阅[Nagios预定义宏][3]，会发现并没有QQ号这样的预定义宏。

当然可以选用预定义的 [CONTACTPAGER][4] 。考虑到寻呼机已经几乎没人使用了，可以借用一下变量。

针对各种工具（IM/内部通信接口/短信平台接口等），需要自定义。

关于自定义，Nagios的[文档][5]已有说明，这里简单提及一下：

+ 自定义变量必须以`_`开头以防止与预定义变量冲突
+ 自定义变量使用时需要转为全大写
+ 自定义变量会在前方加上所属的对象类型

针对第三点，简单说来，针对Contact，即联系人这一对象，如果定义了名为`uid`的自定义变量，那么，在contact的配置中，需要写成`_uid`，而实际在配置command时，会变为`$_CONTACTUID$`。

## 配置commands

新增一个发送报警命令`send_pm`，定义这一自定义报警的调用方式。

```
# 'send_pm' is an alert command
define command {
    command_name    send_pm
    command_line    /path/to/your/own/plugins/send_pm.php -h $HOSTADDRESS$ -u $_CONTACTUID$ -l $SERVICESTATE$ -s $SERVICEDISPLAYNAME$  -m "$HOSTADDRESS$|$
SERVICEDISPLAYNAME$|$SERVICEOUTPUT$" -n $NOTIFICATIONTYPE$ -r 300
}
```

参数列表正如上文提及的。

报警内容为了简短，只会发送主机名、服务名称以及检查结果的首行。

## 配置contacts

在需要报警的联系人配置中，加上自定义的`uid`变量，以及指定报警方法：

```
service_notification_commands   send_pm,notify-service-by-email
host_notification_commands      send_pm,notify-service-by-email
_uid 9527
```

以上。




[1]: http://www.liaoaoyang.com/articles/2016/01/05/monitor-redis-by-nagios-part-2/
[2]: https://zh.wikipedia.org/zh/Shebang
[3]: https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/macrolist.html
[4]: https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/objectdefinitions.html#contact
[5]: https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/customobjectvars.html