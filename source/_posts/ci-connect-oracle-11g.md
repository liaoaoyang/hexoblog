date: 2014-09-20 00:00:00
title: CI连接Oracle 11G数据库
categories: LAMP/LNMP
tags: [PHP,Oracle]
---

[CI][1]框架算是个人最喜欢的PHP框架之一，易用性上没的说，还有完备的[中文文档][2]，不过大多数时候是搭配MySQL一起使用。

不过最近接触的一个项目使用的是Oracle 11G数据库，开发前给大家搭环境的时候发现连接有一些问题，主要来说是安装配置上的一些问题。

<!-- more -->

## 环境

+ CodeIgniter 2.2.0
+ Oracle 11G R2
+ CentOS 6.4
+ PHP 5.2

## 扩展安装

首先CI本身是能支持Oracle数据库的，在DB Driver的代码中可以明确地看到，下面需要的就是安装oci8扩展了。

oci8扩展在安装上和其他的PHP扩展没有太多的区别，稍微有点区别的是需要下载安装一个[Instant Client][3]，Windows下的下载安装倒也还算顺利，然后Linux下的下载真是让人哭笑不得了，因为页面上的js错误，点击我同意按钮之后是不会出现熟悉的下载功能的，即各个链接仍然连接到本页，不过没有关系，看了下页面源码，还是找出了[rpm][4]包的实际下载链接（当然这个也是要注册Oracle的账户才能下载的）。

> http://download.oracle.com/otn/linux/instantclient/112010/oracle-instantclient11.2-basic-11.2.0.1.0-1.x86_64.rpm

同时还需要安装[devel][5]包否则在编译扩展时会出现找不到头文件的情况。

> http://download.oracle.com/otn/linux/instantclient/112010/oracle-instantclient11.2-devel-11.2.0.1.0-1.x86_64.rpm

之后就是常规的`phpize && configure && make && make install`了。

## CI配置

下面来看在autoload配置文件中已经配置了autoload database配置的情况下CI的配置。

网上对于CI的配置主要区别在hostname这一个项目，有写成tnsnames.ora样式的，这个自己没实验成功，最后读了一下CI连接部分的代码，确定了连接中hostname配置应该是：

> //数据库IP:数据库端口/数据库名称

最终连接成功的配置如下：

```
$db['default']['hostname'] = "//192.168.1.200:1521/db200";
$db['default']['username'] = 'learn';
$db['default']['password'] = '123456';
$db['default']['database'] = '';
$db['default']['dbdriver'] = 'oci8';
```

db200是dbca安装数据库时指定的名称。

环境搭好之后开发自然是要开始了。

以上。

[1]: https://github.com/EllisLab/CodeIgniter/
[2]: http://codeigniter.org.cn/user_guide/toc.html
[3]: http://www.oracle.com/technetwork/cn/topics/linuxx86-64soft-095635-zhs.html
[4]: http://download.oracle.com/otn/linux/instantclient/112010/oracle-instantclient11.2-basic-11.2.0.1.0-1.x86_64.rpm
[5]: http://download.oracle.com/otn/linux/instantclient/112010/oracle-instantclient11.2-devel-11.2.0.1.0-1.x86_64.rpm


