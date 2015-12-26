date: 2014-12-07 00:00:00
title: YUM安装的Nginx如何添加模块
categories: LAMP/LNMP
tags: [Nginx,yum]
---

# 为yum安装的Nginx添加模块

`Nginx`一般都是编译安装，不过它本身也提供了通过[`yum`][1]安装的方式，比如在CentOS 6.5中需要添加的`/etc/yum.repos.d/nginx.repo `文件内容为：

```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/6/$basearch/
gpgcheck=0
enabled=1
```

之后就是常规的`yum -y install nginx`。

通过这一方式安装的Nginx已经可以通过系统服务的方式进行启动，相当便捷，但是很多有趣的第三方插件并没有能够加入，比如[`header-more-nginx-module`][2]，Nginx目前看来不像`Tengine`是可以后期动态添加模块的，所以解决的方案出了编译安装似乎没有其他的方式了。

不过Nginx编译只生成一个二进制文件，那么，如果获取yum安装的Nginx编译参数，之后使用同一版本的源代码进行编译，之后替换生成文件就可以了。

通过`nginx -V`就可以看到编译参数(1.6.2):

```
./configure --prefix=/etc/nginx \
            --sbin-path=/usr/sbin/nginx \
            --conf-path=/etc/nginx/nginx.conf \
            ...
```

在获取同版本的nginx源码，同时也获得所需模块的源码后，在获得编译参数中加入`--add-module=/path/to/module/source`：

```
./configure --prefix=/etc/nginx \
            --sbin-path=/usr/sbin/nginx \
            --conf-path=/etc/nginx/nginx.conf \
			...
            --add-module=../headers-more-nginx-module
```

之后进行`make`即可。

编译完成之后先备份位于/usr/sbin/中的nginx文件，之后关闭nginx服务，替换文件，重启服务，一个添加了所需模块的nginx应该就能正常工作了。

[1]: http://nginx.org/en/linux_packages.html
[2]: http://github.com/agentzh/headers-more-nginx-module

