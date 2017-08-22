title: PHP实现daemon
date: 2017-08-22 21:39:12
tags: [PHP, daemon]
categories: PHP
---

# TL;DR

PHP实现守护进程可以通过 `pcntl` 与 `posix` 扩展实现。

编程中需要注意的地方有：

+ 通过二次 `pcntl_fork()` 以及 `posix_setsid` 让主进程脱离终端
+ 通过 `pcntl_signal()` 忽略或者处理终端关闭时的 `SIGHUP` 信号
+ 多进程程序需要通过二次 `pcntl_fork()` 或者 `pcntl_signal()` 忽略 `SIGCHLD` 信号防止子进程变成 Zombie 进程
+ 通过 `umask()` 设定文件权限掩码，防止继承文件权限而来的权限影响功能
+ 将运行进程的 `STDIN/STDOUT/STDERR` 重定向到 `/dev/null` 或者其他流上

如果要做的更好，还需要注意：

+ 如果通过 root 启动，运行时更换到低权限用户身份
+ 及时 `chdir()` 防止操作错误路径
+ 多进程程序考虑定时重启，防止内存泄露

<!-- more -->

# 什么是daemon

文章的主角[守护进程（daemon）](https://zh.wikipedia.org/wiki/%E5%AE%88%E6%8A%A4%E8%BF%9B%E7%A8%8B)，Wikipedia 上的定义是：

> 在一个多任务的电脑操作系统中，守护进程（英语：daemon，/ˈdiːmən/或/ˈdeɪmən/）是一种在后台执行的电脑程序。此类程序会被以进程的形式初始化。守护进程程序的名称通常以字母“d”结尾：例如，syslogd就是指管理系统日志的守护进程。
通常，守护进程没有任何存在的父进程（即PPID=1），且在UNIX系统进程层级中直接位于init之下。守护进程程序通常通过如下方法使自己成为守护进程：对一个子进程运行fork，然后使其父进程立即终止，使得这个子进程能在init下运行。这种方法通常被称为“脱壳”。

[UNIX环境高级编程（第二版）](https://book.douban.com/subject/1788421/)（以下使用简称 APUE 指代） 13章有云：

> 守护进程也成精灵进程（ daemon ）是生存周期较长的一种进程。它们常常在系统自举时启动，仅在系统关闭时才终止。因为他们没有控制终端，所以说他们是在后台运行的。

这里注意到，daemon有如下特征：

+ 没有终端
+ 后台运行
+ 父进程 pid 为1

想要查看运行中的守护进程可以通过 `ps -ax` 或者 `ps -ef` 查看，其中 `-x` 表示会列出没有控制终端的进程。

# 实现关注点

## 重定向0/1/2

这里的0/1/2分别指的是 `STDIN/STDOUT/STDERR`，即标准输入/输出/错误三个流。

### 样例

首先来看一个样例：

```
<?php

// not_redirect_std_stream_daemon.php

$pid1 = pcntl_fork();

if ($pid1 > 0) {
    exit(0);
} else if ($pid1 < 0) {
    exit("Failed to fork 1\n");
}

if (-1 == posix_setsid()) {
    exit("Failed to setsid\n");
}

$pid2 = pcntl_fork();

if ($pid2 > 0) {
    exit(0);
} else if ($pid2 < 0) {
    exit("Failed to fork 2\n");
}

umask(0);
declare(ticks = 1);
pcntl_signal(SIGHUP, SIG_IGN);

echo getmypid() . "\n";

while(true) {
    echo time() . "\n";
    sleep(10);
}
```

上述代码几乎完成了文章最开始部分提及的各个方面，唯一不同的是没有对标准流做处理。通过 `php not_redirect_std_stream_daemon.php` 指令也能让程序在后台进行。

在 `sleep` 的间隙，关闭终端，会发现进程退出。

通过 `strace` 观察系统调用的情况：

```
➜  ~ strace -p 6723
Process 6723 attached - interrupt to quit
restart_syscall(<... resuming interrupted call ...>) = 0
write(1, "1503417004\n", 11)            = 11
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigaction(SIGCHLD, NULL, {SIG_DFL, [], 0}, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
nanosleep({10, 0}, 0x7fff71a30ec0)      = 0
write(1, "1503417014\n", 11)            = -1 EIO (Input/output error)
close(2)                                = 0
close(1)                                = 0
munmap(0x7f35abf59000, 4096)            = 0
close(0)                                = 0
```

发现发生了 EIO 错误，导致进程退出。

原因很简单，即我们编写的 daemon 程序使用了当时启动时终端提供的标准流，当终端关闭时，标准流变得不可读不可写，一旦尝试读写，会导致进程退出。

在[信海龙](http://www.bo56.com/)的博文[《一个echo引起的进程崩溃》](http://www.bo56.com/%E4%B8%80%E4%B8%AAecho%E5%BC%95%E8%B5%B7%E7%9A%84%E8%BF%9B%E7%A8%8B%E5%B4%A9%E6%BA%83/)中也提到过类似的问题。

### 解决方案

#### APUE 样例

APUE 13.3中提到过一条编程规则（第6条）：

> 某些守护进程打开 `/dev/null` 时期具有文件描述符0、1和2，这样，任何一个视图读标准输入、写标准输出或者标准错误的库例程都不会产生任何效果。因为守护进程并不与终端设备相关联，所以不能在终端设备上显示器输出，也无从从交互式用户那里接受输入。及时守护进程是从交互式会话启动的，但因为守护进程是在后台运行的，所以登录会话的终止并不影响守护进程。如果其他用户在同一终端设备上登录，我们也不会在该终端上见到守护进程的输出，用户也不可期望他们在终端上的输入会由守护进程读取。

简单来说：

+ daemon 不应使用标准流
+ 0/1/2 要设定成 /dev/null

例程中使用：

```
for (i = 0; i < rl.rlim_max; i++)
	close(i);

fd0 = open("/dev/null", O_RDWR);
fd1 = dup(0);
fd2 = dup(0);
```

实现了这一个功能。`dup()` （参考[手册](http://man7.org/linux/man-pages/man2/dup.2.html)）系统调用会复制输入参数中的文件描述符，并复制到最小的未分配文件描述符上。所以上述例程可以理解为：

```
关闭所有可以打开的文件描述符，包括标准输入输出错误；
打开/dev/null并赋值给变量fd0，因为标准输入已经关闭了，所以/dev/null会绑定到0，即标准输入；
因为最小未分配文件描述符为1，复制文件描述符0到文件描述符1，即标准输出也绑定到/dev/null；
因为最小未分配文件描述符为2，复制文件描述符0到文件描述符2，即标准错误也绑定到/dev/null；
```

#### 开源项目实现：Workerman

[Workerman](https://github.com/walkor/Workerman) 中的 [Worker.php](https://github.com/walkor/Workerman/blob/801d914/Worker.php) 中的 `resetStd()` 方法实现了类似的操作。

```
/**
* Redirect standard input and output.
*
* @throws Exception
*/
public static function resetStd()
{
   if (!self::$daemonize) {
       return;
   }
   global $STDOUT, $STDERR;
   $handle = fopen(self::$stdoutFile, "a");
   if ($handle) {
       unset($handle);
       @fclose(STDOUT);
       @fclose(STDERR);
       $STDOUT = fopen(self::$stdoutFile, "a");
       $STDERR = fopen(self::$stdoutFile, "a");
   } else {
       throw new Exception('can not open stdoutFile ' . self::$stdoutFile);
   }
}
```

