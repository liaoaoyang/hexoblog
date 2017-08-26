date: 2015-11-27 00:00:00
title: 使用Nagios监控Redis
categories: Learning
tags: [Nagios,Redis]
---

# 前言

近期新上项目之后出现了后台服务的一些诡异的问题，追查之后发现居然有个Redis的实例把分配的几十G内存给用满了……

终于意识到之前“土法炼钢”的不科学的地方，惭愧之余迅速的想要给Redis加上基本的监控。

最初的想法是直接自己写脚本轮询各个实例的info信息，然后自行parse，当前运行Redis的实例不多，感觉工作量并不大，然而这时候想起组内之前的监控是在用`Nagios`，觉得为什么不让更专业的软件来完成这项工作呢？于是决定通过Nagios来完成监控。

<!-- more -->

以下会使用`/nip`指代Nagios的安装路径，以实际安装路径为准。

# Nagios配置

Nagios分为`Server`和`Client`，通过各个插件完成对Client的监控，并且在Server中收集展现。

既然运用了Nagios，自然就要按照它的设计思路来进行使用，其实也就是使用合适的插件，同时也不想再造轮子，Google之后确定使用这一[项目][1]中的[`check_redis.pl`][2]插件（然而这一文件自2013年7月之后没有过更新了……），这一插件从功能上基本上满足了我对使用内存、连接数、key个数、特定队列长度的监控需求。

这一插件基本上是帮助完成了从`redis-cli -h host -p port info`到Nagios的收集展现过程，所以本身只要在Server上安装即可。

## 安装插件

安装插件本身并不复杂，只需要download & copy即可，即copy到`/nip/libexec`目录，并`chmod +x`赋予可执行权限即可。

然而，真正头疼的是这个插件的依赖安装，由于在服务器上初始化cpan的工作始终无法完成，无奈之下只好通过手工安装依赖。

这一插件的依赖分别是：

+ ExtUtils-MakeMaker
+ IO-Socket-Timeout
+ Try-Tiny
+ Redis

可以在cpan的网站上搜索下载tar.gz文件，解压后基本都通过：

```
perl Makefile.PL
make
sudo make install
```

这一过程完成依赖的安装过程。

## 配置Nagios

想要让监控正常的run起来，配置也是很关键的一个因素，个人认为，配置主要针对监控对象以及监控动作。

### 监控对象

对于监控对象来说，其实就是标明哪台Server上有着对应的服务，同时还可以对他们进行分组。

监控对象的声明，可以在`/nip/etc/conf/hosts.cfg`中的对应Server配置中增加一个别名，如：

```
define host{
    use linux-server
    host_name 192.168.1.101
    address 192.168.1.101
    contact_groups admins,redis_dba
    alias redis-linux
}
```

声明的别名会在报警邮件中得以展现。

同时，需要在`/nip/etc/conf/hostgroups.cfg`中，为这一批机器分组，以便对一组实例完成监控。

```
define hostgroup {
    hostgroup_name  Redis_Servers_9101
    alias           Redis Servers 9101
    members         192.168.1.101,192.168.1.102,192.168.1.103
}
```

### 监控动作

对Redis监控，首先要保证Redis实例可访问，不过这一点不用特别配置Nagios,我们需要做的只是针对我们关心数值，进行声明以及配置报警阈值即可。

首先需要在`/nip/objects/commands.cfg`中配置一个检查指令：

```
define command {
    command_name    check_redis
    command_line    $USER1$/check_redis.pl -H $HOSTADDRESS$ -p $ARG1$ -a $ARG2$ -w $ARG3$ -c $ARG4$ -f
}
```

以上参数的含义可以通过`/nip/libexec/check_redis.pl --help`查看详情：

```
...
-H, --hostname=ADDRESS
   Hostname or IP Address to check
 -p, --port=INTEGER
   port number (default: 6379)
...
-a, --variables=STRING[,STRING[,STRING...]]
   List of variables from info data to do threshold checks on.
   ...
 -w, --warn=STR[,STR[,STR[..]]]
   ...
 -c, --crit=STR[,STR[,STR[..]]]
   ...

Performance Data Processing Options:
 -f, --perfparse
   This should only be used with '-a' and causes variable data not only as part of
   main status line but also as perfparse compatible output (for graphing, etc).
```

上述command的含义为针对主机$HOSTADDRESS$的指定端口$ARG1$，检查参数为$ARG2$（可以对照redis-cli info），在$ARG3$设定WARNING级别告警的数值，在$ARG4$设定CRITICAL级别告警的数值，同时生成数据（-f）。

在配置完监控指令之后，还需要针对之前的已经声明的主机组配置使用监控指令进行监控，在`/nip/etc/conf/services.cfg`增加一项配置：

```
define service {
    use                     generic-service
    hostgroup_name          Redis_Servers_9101
    service_description     Redis Pool
    # WARN: 40G*0.7 CRIT: 40G*0.9
    check_command           check_redis!5104!'used_memory_human,used_memory_peak_human,used_memory_rss,total_keys'!~,~,30064771072,300000!~,~,38654705664,400000
}
```
此处根据个人的实际业务情况，当使用的内存超过28G（30064771072 = 40 * 0.7 * 1024 * 1024 * 1024，以下类比）或者key个数超过30w个时会发出WARNING信息，而在当使用36G内存或者key个数超过40w个时，发出CRITICAL警报。

Nagios的check_command在参数前使用`!`，之后的数值针对每一个监控属性，`~`表示不关注，而对应位置的数值则标称各自的报警阈值。

# 最后

修改了配置之后，Nagios需要重启才能开始执行监控，那么为了防止因为修改配置而出错，需要通过Nagios先行检测配置文件的正确性：

```
sudo /nip/bin/nagios -v /nip/etc/nagios.cfg
```

Nagios的报错信息非常详细，基本可以直接定位到出错的行数。

检查正确之后自然就是重启，等待数据的到来。

[1]: https://github.com/willixix/WL-NagiosPlugins
[2]: https://github.com/willixix/WL-NagiosPlugins/blob/master/check_redis.pl


