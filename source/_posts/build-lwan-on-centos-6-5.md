date: 2014-12-26 00:00:00
title: CentOS 6.5使用lwan
categories: Try
tags: [lwan,CentOS]
---

这些天看到了[`lwan`][1]，[`官网`][1]上号称作者历经三年打造的高性能轻量级可扩展Web Server，号称具备了低内存占用、最小化的系统调用次数、对静态文件根据大小智能的处理，预缓存目录信息以及7200行主体代码等特点，到今天（2014年12月26日）为止，[`Github`][2]已经收集到了2000+个star。

其中最吸引我的就是静态文件能达到18w qps，加上它较小的代码体积，于是想要试用一下。

阅读[`Github`][2]上的说明可知，这一软件需要用cmake，无碍，自己手上这台CentOS 6.5的机器上有，不过看到编译参数里面的-flto之后心里凉了半截，这东西CentOS 6.5上带的GCC 4.4.7不能编译啊……

好吧，简直就是噩耗，更换GCC实在不是什么省心的事儿，主要是手上测试机性能实在太差（Pentium双核+2G RAM+160GB HDD，10年的台式机……），估计只编译C/C++支持的话就得一个多小时了，无碍，那么就编译吧。

本次使用的是[`GCC 4.8.4`][3]，使用了日本镜像，速度较快。GCC还依赖三个库：

+ [`gmp`][4] (6.0.0)
+ [`mpfr`][5] (3.1.2)
+ [`mpc`][6] (1.0.2)

这三个库安装顺序可以为gmp -> mpfr -> mpc，mpc依赖前两者。

同时，需要通过yum安装glibc-devel以及glibc-devel.i686两个包。

在开始config之前，需要添加环境变量：

```
export LD_LIBRARY_PATH=/usr/local/lib
```

否则之前安装的gmp等三个库是没法用上的。

之后便是配置安装：

```
./configure --enable-checking=release \
            --enable-languages=c,c++ \
            --disable-multilib
make
make install
```

漫长的编译之后，会在先前的gcc目录（/user/local/gcc）生成，备份原有gcc（需要mv操作）之后通过可以考虑通过

```
mv /usr/local/bin/gcc /usr/local/bin/gcc-4.4.7
update-alternatives --install \
                    /usr/bin/gcc \
                    gcc \
                    /usr/local/bin/x86_64-unknown-linux-gnu-gcc-4.8.4 50
```

切换版本。

GCC准备好了之后自然就要开始编译，编译时原本出现了一些对于比如luajit以及sqlite3的版本要求，但是作者今天在一个[`issue`][7]中提到似乎已经解决这一问题，这里就先略过，实际出现的话，根据出现问题的cmake提示，将软件依赖改为全路径即可。

编译时需要编译安装luajit以及通过yum安装mysql-devel。

在顶级目录编译完成之后，可以看到在顶级目录下的lwan目录中会生成一个名为lwan的二进制文件，将其拷贝到顶级目录，启动之后就可以体验lwan了。

![lwan][8]

关于端口的问题，可以通过修改顶级目录下的lwan.conf修改监听端口。

不过，最后要说的是其实自己的测试结果并不尽如人意，ab并发过万会引起lwan不响应其他请求，而Nginx则没有这类问题，还需要再次试验，确定问题的原因。

[1]: http://lwan.ws/
[2]: https://github.com/lpereira/lwan
[3]: http://ftp.tsukuba.wide.ad.jp/software/gcc/releases/gcc-4.8.4/
[4]: https://gmplib.org/#DOWNLOAD
[5]: http://www.mpfr.org/mpfr-current/#download
[6]: http://www.multiprecision.org/index.php?prog=mpc&page=download
[7]: https://github.com/lpereira/lwan/issues/75
[8]: https://blog.wislay.com/wp-content/uploads/2014/12/lwan_succ.jpg
