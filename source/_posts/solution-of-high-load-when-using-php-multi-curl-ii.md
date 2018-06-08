title: 降低使用PHP curl_multi_* 时的 load（DNS查询）
date: 2018-06-06 02:00:42
tags: [cURL,PHP, curlmulti, DNS]
categories: PHP
---

# TL;DR

`curl_multi_perform` 驱动每个连接的内部状态机，完成创建连接，解析 DNS，连接服务器，发送数据请求，接收响应等多个阶段任务。

在解析 DNS 时，如果采用了异步 DNS（c-ares），由于传递到内部用于轮询 DNS 相关的 fd 的 `select` 系统调用超时为0，变成了一个非阻塞的调用，如果没有 fd 可以读写则立即返回。会导致curl状态机一直维持在等待域名解析完成的状态，如果在外部逻辑中不主动休眠，会继续轮询请求 `curl_mutli_exec` / `curl_multi_select`，CPU占用会提高。

<!-- solution-of-high-load-when-using-php-multi-curl -->
<!-- more -->

# 现象

首先安装启用了 c-ares 和不启用 c-ares 的两个 libcurl，并编译 curl 扩展

## 启用 c-ares

```
➜  curl git:(master) ✗ /usr/local/cmdphp7/bin/php --ri curl

curl

cURL support => enabled
cURL Information => 7.29.0
Age => 3
Features
AsynchDNS => Yes
CharConv => No
Debug => No
GSS-Negotiate => Yes
IDN => Yes
IPv6 => Yes
krb4 => No
Largefile => Yes
libz => Yes
NTLM => Yes
NTLMWB => Yes
SPNEGO => No
SSL => Yes
SSPI => No
TLS-SRP => No
Protocols => dict, file, ftp, ftps, gopher, http, https, imap, imaps, ldap, ldaps, pop3, pop3s, rtsp, scp, sftp, smtp, smtps, telnet, tftp
Host => x86_64-redhat-linux-gnu
SSL Version => NSS/3.28.4
ZLib Version => 1.2.7
libSSH Version => libssh2/1.4.3
➜  curl git:(master) ✗ strace -c /usr/local/cmdphp7/bin/php ~/testmc1.php https://test.com 0 1
curl_multi_select return -1 1554 times
curl_multi_select runs 1556 times
curl_multi_exec runs 1556 times
Lenght 21296 occurred 1 times
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 86.61    0.011480           2      5527      5520 access
  2.46    0.000326           2       162           mmap
  2.41    0.000320           3       113           mprotect
  2.34    0.000310           2       141        56 open
  1.63    0.000216           1       355         3 read
  1.08    0.000143           3        51           munmap
  0.72    0.000095           2        53           brk
  0.60    0.000080           1       118           fstat
  0.56    0.000074           1        81           rt_sigaction
  0.41    0.000054           1        91           close
  0.38    0.000050           1        68           fcntl
  0.29    0.000038           1        29        11 stat
  0.17    0.000023           1        32         3 lseek
  0.12    0.000016           8         2         2 statfs
  0.10    0.000013           2         7           lstat
  0.02    0.000003           3         1           getcwd
  0.02    0.000003           1         3           getrlimit
  0.02    0.000002           1         2         2 ioctl
  0.02    0.000002           2         1           execve
  0.02    0.000002           2         1           arch_prctl
  0.02    0.000002           2         1           set_tid_address
  0.01    0.000001           1         2           rt_sigprocmask
  0.01    0.000001           1         1         1 madvise
  0.01    0.000001           1         1           set_robust_list
  0.00    0.000000           0         5           write
  0.00    0.000000           0         3           poll
  0.00    0.000000           0         1           select
  0.00    0.000000           0         3           socket
  0.00    0.000000           0         1         1 connect
  0.00    0.000000           0         4           sendto
  0.00    0.000000           0        16           recvfrom
  0.00    0.000000           0         1           getsockname
  0.00    0.000000           0         4         1 getpeername
  0.00    0.000000           0         1           setsockopt
  0.00    0.000000           0         1           getsockopt
  0.00    0.000000           0         1           clone
  0.00    0.000000           0         2           uname
  0.00    0.000000           0         2           sysinfo
  0.00    0.000000           0         1           getuid
  0.00    0.000000           0         1           gettid
  0.00    0.000000           0         2           futex
------ ----------- ----------- --------- --------- ----------------
100.00    0.013255                  6892      5600 total
```

## 不启用 c-ares

```
➜  curl git:(master) ✗ /usr/local/cmdphp7/bin/php --ri curl

curl

cURL support => enabled
cURL Information => 7.28.2-DEV
Age => 3
Features
AsynchDNS => No
CharConv => No
Debug => No
GSS-Negotiate => No
IDN => No
IPv6 => Yes
krb4 => No
Largefile => Yes
libz => Yes
NTLM => Yes
NTLMWB => Yes
SPNEGO => No
SSL => Yes
SSPI => No
TLS-SRP => No
Protocols => dict, file, ftp, ftps, gopher, http, https, imap, imaps, pop3, pop3s, rtsp, smtp, smtps, telnet, tftp
Host => x86_64-unknown-linux-gnu
SSL Version => OpenSSL/1.0.2k
ZLib Version => 1.2.7
➜  curl git:(master) ✗ strace -c /usr/local/cmdphp7/bin/php ~/testmc1.php https://test.com 0 1
curl_multi_select return -1 0 times
curl_multi_select runs 2 times
curl_multi_exec runs 2 times
Lenght 21296 occurred 1 times
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 33.76    0.000186           5        37           munmap
 19.42    0.000107           2        47           brk
 12.16    0.000067           1       128           mmap
 11.80    0.000065           1        80           mprotect
  8.89    0.000049           1        98        40 open
  4.54    0.000025           0       141           read
  2.72    0.000015           4         4         3 connect
  2.00    0.000011           0        66           close
  1.63    0.000009           0        63           fstat
  1.27    0.000007           1        13        11 stat
  0.54    0.000003           1         3           futex
  0.36    0.000002           1         3           getrlimit
  0.18    0.000001           0         7           poll
  0.18    0.000001           1         1           getsockname
  0.18    0.000001           1         1           getpeername
  0.18    0.000001           1         1           getsockopt
  0.18    0.000001           1         1           getuid
  0.00    0.000000           0         9           write
  0.00    0.000000           0         7           lstat
  0.00    0.000000           0         6         3 lseek
  0.00    0.000000           0        83           rt_sigaction
  0.00    0.000000           0         3           rt_sigprocmask
  0.00    0.000000           0         4         2 ioctl
  0.00    0.000000           0         4         2 access
  0.00    0.000000           0         1           select
  0.00    0.000000           0         1         1 madvise
  0.00    0.000000           0         2           alarm
  0.00    0.000000           0         5           socket
  0.00    0.000000           0         2           recvfrom
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           uname
  0.00    0.000000           0         2           fcntl
  0.00    0.000000           0         1           getcwd
  0.00    0.000000           0         2         2 statfs
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           set_robust_list
  0.00    0.000000           0         1           sendmmsg
------ ----------- ----------- --------- --------- ----------------
100.00    0.000551                   832        64 total
```


