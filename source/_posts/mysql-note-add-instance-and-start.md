date: 2014-07-28 00:00:00
title: MySQL的简单使用笔记：增加实例以及启动
categories: LAMP/LNMP
tags: [MySQL]
---

## 增加实例

增加实例这里指的的在源码编译安装完MySQL之后建立一个初始的数据库实例，占用某一端口，或者是使用新端口启动新的mysqld进程。

MySQL需要一些基础的数据库以及表来完成基本的设定，比如控制连接的**`mysql`.`user`**表：

```
+------------------+----------------+------+-----+---------+-------+
| Field            | Type           | Null | Key | Default | Extra |
+------------------+----------------+------+-----+---------+-------+
| Host             | char(60)       | NO   | PRI |         |       |
| User             | char(16)       | NO   | PRI |         |       |
| Password         | char(41)       | NO   |     |         |       |
| Select_priv      | enum('N','Y')  | NO   |     | N       |       |
| Insert_priv      | enum('N','Y')  | NO   |     | N       |       |
| Update_priv      | enum('N','Y')  | NO   |     | N       |       |
| Delete_priv      | enum('N','Y')  | NO   |     | N       |       |
| ...              | ...            | ...  | ... | ...     | ...   |
+------------------+----------------+------+-----+---------+-------+

```

在编译完成MySQL之后这些表是不存在的，需要通过安装目录下的**`script/mysql_install_db`**完成基础表的安装工作。

在这个脚本中完成安装工作所需要的参数至少需要如下几个：

+ **basedir** MySQL的安装目录
+ **datadir** MySQL实例的数据文件目录，比如数据库文件、socket文件等
+ **user** 安装MySQL时的设定的用户名

通过执行这一脚本，例如：

```
./mysql_install_db --basedir=/usr/local/mysql
                   --datadir=/usr/local/mysql/mysql_data/
                   --user=mysql
```

即完成了对MySQL的初始化操作，即完成了一个数据文件位于**`/usr/local/mysql/mysql_data/
`**的数据库实例，通过使用不同的端口和目录，完成新增一个数据实例的工作。

## 启动

启动MySQL可以通过MySQL安装完成的**[mysqld_safe][1]**工具完成。

mysqld_safe脚本是推荐的启动MySQL的方式，其中特点是增加了一些安全保证的机制比如遇到错误重启并且写入日志（参数中的log-error指定位置）。

比较重要的个人认为是：

+ **default-file** 默认配置文件位置，如果使用这个参数，会通过默认文件获取配置，需要注意的是，如果需要使用，这个参数必须要放在第一个参数才能生效
+ **basedir** MySQL安装目录
+ **datadir** MySQL数据库的数据目录
+ **user** MySQL安装时配置的用户
+ **pid-file** pid文件位置
+ **port** 监听端口
+ **socket** 响应本地MySQL连接请求的socket文件位置

所有的参数事实上都会传送给**mysqld**，如果在配置文件(假设是/usr/local/mysql/my.cnf)中指明了MySQL的一切，则只需要使用简单的一句：

```
/usr/local/mysql/bin/mysqld_safe -default-file=/usr/local/mysql/my.cnf --user=mysql &
```
即可启动，同时也可以在参数中补齐缺失的项目。

后期计划整出一份MySQL启动所需最小配置参数列表。

## 其他

新增的数据库实例以及忘记密码的情况下需要通过**mysqladmin**工具完成密码的设定，需要指定数据库的数据socket连接文件位置以及用户名以及新的密码，例如：

```
/usr/local/mysql/bin/mysqladmin -u root --socket=/usr/local/mysql/mysql_data/mysql.sock password 123456
```

以上。



[1]: http://dev.mysql.com/doc/refman/5.0/en/mysqld-safe.html
