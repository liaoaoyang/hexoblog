title: 查看MySQL LOAD DATA进度
date: 2016-04-03 20:30:44
tags: MySQL
---

# 概述

开发过程中经常会使用MySQL的[LOAD DATA][3]功能，用于导入文件到MySQL的指定数据库表中。

若已经将文件切分为N个小文件再进行LOAD操作（例如使用Linux下的 `split` 工具），那么进度还是很容易把控的，可以通过直接查找当前正在进行导入的分片，进而判断当前的分片。

可是，如果某些情况下直接对一个大型的文件进行进行LOAD操作，整个过程并不能直观的获取当前的进度的，需要通过一些相对曲折的过程才能获取当前LOAD的进度。

<!-- more -->

# 分析

## /proc虚拟文件系统

Linux中的`/proc`虚拟文件系统是一个非常有趣的部分，这一个目录并不是包含了一些常规意义上的文件，而是表征了进程的部分运行时信息。部分Linux工具更是可以直接用读取目录中的部分信息来替代[\[1\]][1]。

在`/proc`下可以看到大量的名为数字的目录，这些数字正是进程的pid。而cd到其中任何一个目录下，可以看到类似的信息：

```
/proc/18458$ ll
总用量 0
dr-xr-xr-x 2 root root 0 3月  29 10:48 attr
-rw-r--r-- 1 root root 0 4月   3 21:34 autogroup
-r-------- 1 root root 0 4月   3 21:34 auxv
-r--r--r-- 1 root root 0 4月   3 21:34 cgroup
--w------- 1 root root 0 4月   3 21:34 clear_refs
-r--r--r-- 1 root root 0 1月  27 14:44 cmdline
-rw-r--r-- 1 root root 0 4月   3 21:34 comm
-rw-r--r-- 1 root root 0 4月   3 21:34 coredump_filter
-r--r--r-- 1 root root 0 4月   3 21:34 cpuset
lrwxrwxrwx 1 root root 0 4月   3 21:34 cwd -> /
-r-------- 1 root root 0 3月  18 10:31 environ
lrwxrwxrwx 1 root root 0 2月  23 13:24 exe -> /usr/local/nginx/sbin/nginx
dr-x------ 2 root root 0 3月  29 10:48 fd
dr-x------ 2 root root 0 4月   3 21:34 fdinfo
-r-------- 1 root root 0 4月   3 21:34 io
-rw------- 1 root root 0 4月   3 21:34 limits
-rw-r--r-- 1 root root 0 4月   3 21:34 loginuid
-r--r--r-- 1 root root 0 4月   3 21:34 maps
-rw------- 1 root root 0 4月   3 21:34 mem
-r--r--r-- 1 root root 0 4月   3 21:34 mountinfo
-r--r--r-- 1 root root 0 4月   3 21:34 mounts
-r-------- 1 root root 0 4月   3 21:34 mountstats
dr-xr-xr-x 5 root root 0 4月   3 21:34 net
dr-x--x--x 2 root root 0 4月   3 21:34 ns
-r--r--r-- 1 root root 0 4月   3 21:34 numa_maps
-rw-r--r-- 1 root root 0 4月   3 21:34 oom_adj
-r--r--r-- 1 root root 0 4月   3 21:34 oom_score
-rw-r--r-- 1 root root 0 4月   3 21:34 oom_score_adj
-r--r--r-- 1 root root 0 4月   3 21:34 pagemap
-r--r--r-- 1 root root 0 4月   3 21:34 personality
lrwxrwxrwx 1 root root 0 4月   3 21:34 root -> /
-rw-r--r-- 1 root root 0 4月   3 21:34 sched
-r--r--r-- 1 root root 0 4月   3 21:34 schedstat
-r--r--r-- 1 root root 0 4月   3 21:34 sessionid
-r--r--r-- 1 root root 0 4月   3 21:34 smaps
-r--r--r-- 1 root root 0 4月   3 21:34 stack
-r--r--r-- 1 root root 0 1月  27 14:44 stat
-r--r--r-- 1 root root 0 1月  27 15:55 statm
-r--r--r-- 1 root root 0 1月  27 14:44 status
-r--r--r-- 1 root root 0 4月   3 21:34 syscall
dr-xr-xr-x 3 root root 0 4月   3 21:34 task
-r--r--r-- 1 root root 0 4月   3 21:34 wchan
```

各个目录的说明可以参考[此处][2]。

## /proc下的fdinfo

这里我们关注的地方是如何通过这些丰富的信息获取导入数据库的进度。

考虑到这一导入操作，实际上是利用了MySQL进行读取文件的操作，那么，只需要知道MySQL当前读取的文件位置，就可以了解到当前的进度了。

`/proc/[PID]/fdinfo/`这一目录正是解决这一问题的关键，这一目录包含了当前进程已打开的文件的信息，其中文件名正是文件描述符的名称，而相关信息则存储在这个只读文件之中。包含的信息形如：

```
/proc/18458/fd$ cat ../fdinfo/7
pos:	2293603
flags:	0102001
```

### pos

`pos`即文件读取游标的偏移值，也就是我们关注的已读取到的位置。

### flags

`flags`则是一个八进制数，表征当前文件的打开状态。

以上述打开的文件为例，这是一个Nginx打开的日志文件，通过`lsof +fg -p [PID]`可以看到这一文件打开使用的flag：

```
COMMAND  PID USER   FD   TYPE  FILE-FLAG             DEVICE   SIZE/OFF     NODE NAME
...
nginx   18458 root    7w   REG    W,AP,LG                8,7    2293603  4194313 /home/www/logs/access_log
```

可以看到使用了W、AP、LG三个flag，而`W`对应的是`O_WRONLY`，`AP`对应的是`O_APPEND`，`LG`对应的的`O_LARGEFILE`，这三个常量的值一般可以在`/usr/include/bits/fcntl.h`中找到：

```
bits/fcntl.h:36:#define O_WRONLY	     01
bits/fcntl.h:42:#define O_APPEND	  02000
bits/fcntl.h:71:#  define O_LARGEFILE	0100000
```
所以flags的值为何是`0102001`也可以解释了。

# 获取进度

根据上述分析，首先我们直接找到正在进行LOAD操作的MySQL进程的PID，获取之后查看当前打开的文件（假设文件名为foo）在进程中的fd：

```
ls -l /proc/[PID]/fd | grep foo | grep -v grep | awk '{print $9}'
```

获取fd之后，直接读取对应的fdinfo：

```
cat /proc/[PID]/fdinfo/[fd]
```

根据`pos`可以知道当前已读取了的文件位置，进而获知LOAD进度。

# 工具化

参见[GitHub](https://github.com/liaoaoyang/toolbox/blob/master/scripts/mysql/mysql_load_progress.sh);

用法：

```
./mysql_load_progress.sh /full/path/to/loading/file
```

输入文件名需要是完整路径。

代码：

```
#!/bin/sh

LOAD_FILE_NAME=$1

if [ ! -f $LOAD_FILE_NAME ];then
	echo "No such file"
	exit
fi

MYSQL_PIDS=`ps -ef | grep mysql | awk '{print $2}'`
fsize=`ls -l $LOAD_FILE_NAME | awk '{print $5}'`

for pid in $MYSQL_PIDS
do
    fd=`lsof -p $pid | grep $LOAD_FILE_NAME | grep -vE "grep|sh -c" | awk '{print $4}' | grep -oP '\d+(?=r)'`

    if [ -f  /proc/$pid/fdinfo/$fd ];then
        read_pos=`cat /proc/$pid/fdinfo/$fd  | grep pos | awk "{print \\$2/$fsize*100\"%\"}"`
        echo $LOAD_FILE_NAME" "$read_pos
    fi
done
```

以上。

[1]: http://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html
[2]: http://man7.org/linux/man-pages/man5/proc.5.html
[3]: http://dev.mysql.com/doc/refman/5.7/en/load-data.html


