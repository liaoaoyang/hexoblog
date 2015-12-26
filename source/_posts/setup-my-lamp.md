date: 2014-02-21 00:00:00
title: 尝试搭建LAMP环境
categories: LAMP/LNMP
tags: [LAMP]
---

之前自己一直用apt-get在Ubuntu上搭环境，实在是很方便，自动的解决了很多的依赖问题，可以说是真是很省心。用VPS的时候有时候也用比如[EZHTTP][1]这样的LAMP/LNMP一键安装包。不过老用这些方便的东西做自己就觉得有些问题，比如自己不能控制Apache、PHP、MySQL的安装位置，在需要进入这些软件的安装位置的时候，就得用dpkg -L来找到这些软件的安装位置。完全全自动的安装感觉自己也不能了解到更多的细节。

此外还有个很头疼的情况是Windows下对于一些软件的使用实在是很折腾，比如上个星期使用npm安装几个包，结果因为需要重新编译node，折腾了好长时间，各种花费时间成本，而在Linux下这些工作难度就会降低不少（相对的……）。在开发的时候经常需要部署代码到开发机上，然而在有跳板机的情况下，不能使用IDE直连开发环境，配Samba不知道是否可以，但是感觉上这样做不够安全。回家之后想要测点东西，开发机也难连上。最后还是决定在自己的PC上安装虚拟机的方式进行部署环境了。此外个人觉得在Linux下安装PHP扩展比在Win下安装方便好多，完全没有Win下那么的让人困惑。因为在自己工作中，目前只是需要一套LAMP的环境即可，发行版就用了自己用的比较多的Ubuntu。除了安装Apache、MySQL、PHP之外，还需要安装了memcached和Redis。

Apache、MySQL、PHP的安装过程基本上和自己接触过的一些Linux软件安装过程相似，都是
```
configure
make
sudo make install
```
这几个步骤，个人觉得关键的部分就在于configure这个地方。

安装顺序自己选择了Apache->MySQL->PHP->Redis->memcahced。这样选择原因是是PHP需要整合进Apache并且要对MySQL进行支持，所以首先安装好前面两者。整个安装过程还算便捷，其实安装过程中出现过各种依赖、错误，很多时候完全可以通过Google就能解决。

本次安装过程中安装的软件版本为:

+ Apache 2.2.26
+ MySQL 5.5.35
+ PHP 5.4.24
+ Redis 2.8.6
+ Memcached 1.4.17
+ libevent 2.0.21
+ libmemcached 1.0.16
+ PHP Memcached extension 2.1.0


----------

# Apache
安装Apache时，在configure这一步的时候，因为想要尽量和生产环境一致，使用了
```
cat $APACHE_PATH/build/config.nice
```
来查看安装时配置，$APACHE_PATH为生产环境上实际的Apache安装位置。例如：
```
➜  local  cat /usr/local/lamp/apache/build/config.nice
\#! /bin/sh
\#
\# Created by configure

"./configure" \
"--prefix=/usr/local/lamp/apache" \
"--enable-so" \
"--enable-rewrite=shared" \
"--enable-mime=shared" \
"$@"
```
上面的配置很少，实际配置项还是很多的，如果只用
```
./configure --prefix=/usr/local/lamp/apache
```
进行configure实际上也是能跑的，之后如果需要再安装其他模块，比如rewrite模块，可以通过[apxs][2]进行安装。

## apxs安装模块rewrite
在Apache的源代码目录中下的modules/mappers中存在rewrite模块的源码，通过
```
$APACHE_PATH/bin/apxs -c mod_rewrite.c
sudo $APACHE_PATH/bin/apxs -i -a -n rewrite mod_rewrite.la
```
其中-c选项表示编译，-i表示安装，-a表示在httpd中增加一句LoadModule语句，而增加的名称则通过-n的参数决定。
上述命令可以解释为编译mod_rewrite，之后安装，并在$APACHE_PATH/conf/httpd.conf中增加
```
LoadModule rewrite_module modules/mod_rewrite.so
```
其中LoadModule语句中第一个参数由安装命令中的-n选项指定。之后重启apache即可测试是否能够正常使用该模块。

----------

# MySQL
MySQL的安装比Apache的安装过程稍微麻烦了一些，在配置的时候需要使用cmake而不是用一般软件都自带的configure。
```
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/lamp/mysql \
      -DDEFAULT_CHARSET=utf8 \
      -DDEFAULT_COLLATION=utf8_general_ci \
      -DWITH_MYISAM_STORAGE_ENGINE=1 \
      -DWITH_INNOBASE_STORAGE_ENGINE=1 \
      -DMYSQL_DATADIR=/usr/local/lamp/mysql/data .
```
上述命令主要指定了使用的目录以及默认的字符集以及校对（DEFAULT_COLLATION，关于校对问题，参见[MySQL文档-字符集支持][3]），同时开启了MyISAM和InnoDB存储引擎。之后就是熟悉的make && sudo make install过程。Ubuntu上需要先安装好libncurses5-dev。

好在MySQL的[安装文档][4]写得很详细，基本上按照文档中的操作方式就不会出岔子。

之后可以给mysql单独建立用户和用户组，例如均为mysql。之后将整个mysql的安装目录都更改所属人为建立的用户和用户组，同时还有复制标准配置文件并修改成自己的配置文件。这之后有一步非常关键的建立基本数据库的的操作：
```
sudo sh $MYSQL_PATH/scripts/mysql_install_db.sh \
        --basedir=$MYSQL_PATH --datadir=$MYSQL_PATH/data \
        --user=mysql
```
这一个操作使得数据库创建了基本的授权表，使得数据库可以开始使用。

由于使用mysql连接数据库需要使用套接字文件，文件的位置又是通过配置文件获取的，而且mysql读取配置文件的默认顺序为/etc/my.cnf>/etc/mysql/my.cnf>$MYSQL_PATH/etc/my.cnf>~/.my.cnf，所以需要在这些文件中指定套接字文件位置。例如配置成：
```
[client]
port            = 3306
socket          = /usr/local/lamp/mysql/data/mysql.sock

\# Here is entries for some specific programs
\# The following values assume you have at least 32M ram

\# This was formally known as [safe_mysqld]. Both versions are currently parsed.
[mysqld_safe]
socket          = /usr/local/lamp/mysql/data/mysql.sock
nice            = 0

[mysqld]
\#
\# * Basic Settings
\#
user            = mysql
socket          = /usr/local/lamp/mysql/data/mysql.sock
```

配置结束之后，可以通过mysql_safe脚本启动mysql:
```
sudo mysqld_safe --defaults-file=$MYSQL_PATH/conf/mysql.conf \
                 --basedir=$MYSQL_PATH/ \
                 --datadir=$MYSQL_PATH/data/ \
                 --plugin-dir=$MYSQL_PATH/lib/plugin \
                 --user=mysql \
                 --log-error=/var/lamp/log/mysql/mysql_error_log \
                 --pid-file=$MYSQL_PATH/data/mysql.pid \
                 --socket=$MYSQL_PATH/data/mysql.sock --port=3306 &
```
在命令中指定了pid文件位置和socket文件的位置。socket文件的路径可以用于关闭mysql。

关闭mysql的操作可以使用mysqladmin进行：
```
sudo mysqladmin --socket=$MYSQL_PATH/data/mysql.sock shutdown
```
通过socket参数指定关闭哪一个mysql实例。
同样的，mysql工具也可以指定使用哪一个sock文件进行连接。

----------

# PHP
PHP安装过程中最让人头疼的就是那些依赖，好在还有apt-get，安装时一般将对应的依赖包安装即可（例如需要libpng，那么一般安装上libpng和libpng-dev即可，dev一般包含了头文件等），这里不再赘述。

PHP的configure参数也比较多，但是主要是指指定了需要开启的功能支持：
```
./configure --prefix=/usr/local/lamp/php \
            --with-apxs2=$APACHE_PATH/bin/apxs \
            --with-mysql=$MYSQL_PATH \
            --with-mysqli=$MYSQL_PATH/bin/mysql_config \
            --with-zlib-dir \
            --enable-mbstring=all \
            --with-iconv-dir \
            --enable-sockets \
            --with-pear \
            --enable-ftp \
            --with-jpeg-dir \
            --with-png-dir \
            --with-freetype-dir \
            --with-libxml-dir \
            --with-xsl \
            --with-mcrypt \
            --with-pdo-mysql=$MYSQL_PATH \
            --enable-pcntl \
            --with-curl \
            --enable-soap \
            --enable-shmop \
            --with-openssl \
            --with-ldap
```
参数本身也很清晰，在只阅读参数的情况下，大致可以确定需要安装的依赖库的名称。

但是依赖库中也需要注意，在Ubuntu下，实验得知对于curl的支持还需要安装libcurl4-gnutls-dev。参数中的--with-apxs2指定了Apache中的apxs的路径，这里用于生成mod_php5（即对应文件libphp5.so）模块（参见[TIPI 第二章 用户代码的执行 »	第二节 SAPI概述 »	 Apache模块][5]）。

由于使用的PC是联想的K29，使用的是Pentium B960 CPU，不能虚拟出64位主机，所以使用的是32位系统，这里就有一个安装了ldap之后对应的so文件找不到的问题，通过*dpkg -L*可以看到这些文件在/usr/lib/i386-linux-gnu/目录下，这里解决的方法是在/usr/lib中创建对应文件的软链。
```
sudo ln -s /usr/lib/i386-linux-gnu/libldap_r.so /usr/lib/libldap.so
sudo ln -sf /usr/lib/i386-linux-gnu/libldap_r.so /usr/lib/libldap.so
sudo ln -sf /usr/lib/i386-linux-gnu/libldap_r-2.4.so.2.8.1 /usr/lib/libldap_r.so
sudo ln -sf /usr/lib/i386-linux-gnu/libldap_r-2.4.so.2.8.1 /usr/lib/libldap_r-2.4.so.2
sudo ln -sf /usr/lib/i386-linux-gnu/libldap_r.a /usr/lib/libldap.a
sudo ln -sf /usr/lib/i386-linux-gnu/libldap_r.a /usr/lib/libldap-2.4.so.2
sudo ln -sf /usr/lib/i386-linux-gnu/libldap_r-2.4.so.2 /usr/lib/libldap-2.4.so.2
```


make过程中，编译ldap相关的代码时出现了一个liblber.so could not read symbols的问题，通过修改configure完成之后Makefile中的*EXTRA_LIBS*参数，增加-llber字段，这个问题得以解决。

安装完成之后需要设定php.ini文件配置PHP的运行属性，其中一些在编译过程中没有添加的支持，例如gd库，可以通过扩展的方式进行安装。

最后还要设定php文件需要通过执行才能显示，否则请求php就会显示具体代码了……
```
&lt;IfModule mime_module&gt;
    TypesConfig conf/mime.types
    AddType application/x-compress .Z
    AddType application/x-gzip .gz .tgz
    AddType application/x-httpd-php .php
&lt;/IfModule&gt;
```

-------

# Redis
Redis的安装非常方便，只需要进行configure && make即可，完成安装之后需要的是将src目录下编译完成的

+ redis-benchmark
+ redis-check-aof
+ redis-check-dump
+ redis-cli
+ redis-server

复制到需要安装的目录，同时设定配置文件，位置随意，之后用redis-server启动服务：
```
sudo $REDIS_PATH/bin/redis-server $REDIS_PATH/conf/redis_7890.conf
```
配置项中需要将redis作为daemon方式执行，即在配置项目中将*daemonize*配置设定为*yes*。

--------

# Memcached
Memcached的安装需要已安装libevent，之后在configure时加入libevent目录路径：
```
./configure --prefix=/usr/local/srv/memcached --with-libevent=/usr/lib/libevent
```

Memcached的启动参数中的-u选项值得注意，如果使用PHP操作Memcached，当用户为www时，需要将-u也设定为www，才可以进行读写。
```
sudo /usr/local/srv/memcached/bin/memcached -d -u www -m 256 -P /var/srv/memcached.pid
```
其中-d以daemon方式运行，-m表示可以使用内存的MB数目，-P即pid文件的位置。

--------

# PHP的Memcached扩展
这一个扩展的安装比较麻烦，首先需要需要安装libmemcached，而libmemcached的安装需要注意版本，实验中必须安装1.0以上版本才能成功的编译PHP的Memcached扩展，并且需要通过apt-get安装libcloog-ppl0。在：
```
./configure  --prefix=/usr/local/libmemcached --with-memcached
```
之后编译安装libmemcached。

在编译安装完libmemcached之后，在PHP的Memcached扩展的代码顶级目录下，执行：
```
phpize
./configure --enable-memcached \
            --with-php-config=$PHP_PATH/bin/php-config \
            --with-libmemcached-dir=/usr/local/libmemcached
make
sudo make install
```
完成PHP的Memcached扩展的安装，同时在php.ini中启用扩展。

[1]: http://www.centos.bz/ezhttp/
[2]: http://apache.jz123.cn/programs/apxs.html
[3]: https://dev.mysql.com/doc/refman/5.1/zh/charset.html
[4]: https://dev.mysql.com/doc/refman/5.1/zh/installing.html
[5]: http://www.php-internals.com/book/?p=chapt02/02-02-01-apache-php-module
