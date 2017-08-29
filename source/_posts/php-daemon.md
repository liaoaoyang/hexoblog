title: PHP实现daemon
date: 2017-08-22 21:39:12
tags: [PHP, daemon]
categories: PHP
---

# TL;DR

PHP实现守护进程可以通过 `pcntl` 与 `posix` 扩展实现。

编程中需要注意的地方有：

+ 通过二次 `pcntl_fork()` 以及 `posix_setsid` 让主进程脱离终端
+ 通过 `pcntl_signal()` 忽略或者处理 `SIGHUP` 信号
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

## 二次 fork 与 setsid

### fork 系统调用

[fork](http://man7.org/linux/man-pages/man2/fork.2.html) 系统调用用于复制一个与父进程几乎完全相同的进程，新生成的子进程不同的地方在于与父进程有着不同的 pid 以及有不同的内存空间，根据代码逻辑实现，父子进程可以完成一样的工作，也可以不同。子进程会从父进程中继承比如文件描述符一类的资源。

PHP 中的 `pcntl` 扩展中实现了 `pcntl_fork()` 函数，用于在 PHP 中 fork 新的进程。

### setsid 系统调用

[setsid](http://man7.org/linux/man-pages/man2/setsid.2.html) 系统调用则用于创建一个新的会话并设定进程组 id。

这里有几个概念：`会话`，`进程组`。

在 Linux 中，用户登录产生一个会话（Session），一个会话中包含一个或者多个进程组，一个进程组又包含多个进程。每个进程组有一个组长（Session Leader），它的 pid 就是进程组的组 id。进程组长一旦打开一个终端，这一个终端就被称为控制终端。一旦控制终端发生异常（断开、硬件错误等），会发出信号到进程组组长。

后台运行程序（如 shell 中以`&`结尾执行指令）在终端关闭之后也会被杀死，就是没有处理好控制终端断开时发出的`SIGHUP`信号，而`SIGHUP`信号对于进程的默认行为则是退出进程。

调用 `setsid` 系统调用之后，会让当前的进程新建一个进程组，如果在当前进程中不打开终端的话，那么这一个进程组就不会存在控制终端，也就不会出现因为关闭终端而杀死进程的问题。

PHP 中的 `posix` 扩展中实现了 `posix_setsid()` 函数，用于在 PHP 中设定新的进程组。

### 孤儿进程

父进程比子进程先退出，子进程就会变成孤儿进程。

init 进程会收养孤儿进程，即孤儿进程的 ppid 变为 1。

### 二次 fork 的作用

首先，`setsid` 系统调用不能由进程组组长调用，会返回-1。

二次 fork 操作的样例代码如下：

```
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
```

假定我们在终端中执行应用程序，进程为 a，第一次 fork 会生成子进程 b，如果 fork 成功，父进程 a 退出。b 作为孤儿进程，被 init 进程托管。

此时，进程 b 处于进程组 a 中，进程 b 调用 `posix_setsid` 要求生成新的进程组，调用成功后当前进程组变为 b。

此时进程 b 事实上已经脱离任何的控制终端，例程：

```
<?php

cli_set_process_title('process_a');

$pidA = pcntl_fork();

if ($pidA > 0) {
    exit(0);
} else if ($pidA < 0) {
    exit(1);
}

cli_set_process_title('process_b');

if (-1 === posix_setsid()) {
    exit(2);
}

while(true) {
    sleep(1);
}
```

执行程序之后：

```
➜  ~ php56 2fork1.php
➜  ~ ps ax | grep -v grep | grep -E 'process_|PID'
  PID TTY      STAT   TIME COMMAND
28203 ?        Ss     0:00 process_b
```

从 ps 的结果来看，process_b 的 TTY 已经变成了 `？`，即没有对应的控制终端。

代码走到这里，似乎已经完成了功能，关闭终端之后 process_b 也没有被杀死，但是为什么还要进行第二次 fork 操作呢？

StackOverflow 上的一个[回答](https://stackoverflow.com/questions/31485204/why-fork-twice-while-daemonizing)写的很好：

> The second fork(2) is there to ensure that the new process is not a session leader, so it won't be able to (accidentally) allocate a controlling terminal, since daemons are not supposed to ever have a controlling terminal.

这是为了防止实际的工作的进程主动关联或者意外关联控制终端，再次 fork 之后生成的新进程由于不是进程组组长，是不能申请关联控制终端的。

综上，二次 fork 与 setsid 的作用是生成新的进程组，防止工作进程关联控制终端。

## SIGHUP 信号处理

一个进程收到 `SIGHUP` 信号的默认动作是结束进程。

而 `SIGHUP` 会在如下情况下发出：

+ 控制终端断开，SIGHUP 发送到进程组组长
+ 进程组组长退出，SIGHUP 会发送到进程组中的前台进程
+ SIGHUP 常被用于通知进程重载配置文件（APUE 中提及，daemon 由于没有控制终端，被认为不可能会收到这一个信号，所以选择复用）

由于实际的工作进程不在前台进程组中，而且进程组的组长已经退出并且没有控制终端，不处理正常情况下当然也没有问题，然而为了防止偶然的收到 `SIGHUP` 导致进程退出，也为了遵循守护进程程序设计的惯例，还是应当处理这一信号。

## Zombie 进程处理

### 何为 Zombie 进程

简单来说，子进程先于父进程退出，父进程没有调用 `wait` 系统调用处理，进程变为 Zombie 进程。

子进程先于父进程退出时，会向父进程发送 `SIGCHLD` 信号，如果父进程没有处理，子进程也会变为 Zombie 进程。

Zombie 进程会占用可 fork 的进程数，Zombie 进程过多会导致无法 fork 新的进程。

此外，Linux 系统中 ppid 为 init 进程的进程，变为 Zombie 后会由 init 进程回收管理。

### Zombie 进程的处理

从 Zombie 进程的特点，对于多进程的daemon，可以通过两个途径解决这一问题：

+ 父进程处理 `SIGCHLD` 信号
+ 让子进程被 init 接管

父进程处理信号无需多说，注册信号处理回调函数，调用回收方法即可。

对于让子进程被 init 接管，则可以通过2次 fork 的方法，让第一次 fork 出的子进程 a 再 fork 出实际的工作进程 b，让 a 先行退出，使得 b 成为孤儿进程，这样就能被 init 进程托管了。

## umask

umask 会从父进程中继承，影响创建文件的权限。

PHP [手册](http://php.net/manual/zh/function.umask.php)上提到：

> umask() 将 PHP 的 umask 设定为 mask & 0777 并返回原来的 umask。当 PHP 被作为服务器模块使用时，在每个请求结束后 umask 会被恢复。

如果父进程的 umask 没有设定好，那么在执行一些文件操作时，会出现意想不到的效果：

```
➜  ~ cat test_umask.php
<?php
        chdir('/tmp');
        umask(0066);
        mkdir('test_umask', 0777);
➜  ~ php test_umask.php
➜  ~ ll /tmp | grep umask
drwx--x--x 2 root root 4.0K 8月  22 17:35 test_umask
```

所以，为了保证每一次都能按照预期的权限操作文件，需要置0 umask 值。


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

Workerman 中如此实现，结合[博文](https://www.adayinthelifeof.nl/2013/07/10/phps-resources-and-garbage-collection/)，可能与 PHP 的 GC 机制有关，对于 fd 0 1 2来说，PHP 会维持对这三个资源的引用计数，在直接 fclose 之后，会使得这几个 fd 对应的资源类型的变量引用计数为0，导致触发回收。所需要做的就是将这些变量变为全局变量，保证引用的存在。




