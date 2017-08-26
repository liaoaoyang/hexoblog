date: 2014-10-18 00:00:00
title: CentOS装机
categories: LAMP/LNMP
tags: [CentOS,VNC]
---

最近想要做一些实验，无奈自己手头上没有富余的机器，看到好几个同事都自带电脑，没有用公司发的台式机，于是打算用这些台式机来做做实验。

<!-- more -->

公司本身是有完整的LAMP/LNMP环境安装包的，但是基于一些原因是不能使用的，整套环境只能自行安装，综合了一下决定安装如下软件：

+ CentOS 6.5 64位（可以考虑使用 [北理工][1] 的源，速度很快）
+ Nginx 1.2.7
+ PHP 5.4

装机其实是一个很无聊的过程，然而每次都能出现新的状况，真是让人抓狂，写过好几次这样的笔记，但是这次却又遇到了各种情况。

# CentOS

## 启动项问题

安装CentOS已经变得极为简便，由于安装的是公司的台式机，直接用UltraISO或者直接用dd将镜像写入U盘，再通过GUI完成安装即可（[参考此处][2]），不过这次用U盘安装倒是出了个不大不小的问题，即安装之后只要拔掉U盘就无法启动。

这个问题首先感觉应该是引导项写入的问题，前期安装的时候一阵无脑的`下一步`点选，没有注意到写入的设备居然是U盘，而且这台机器U盘被识别成了/dev/sda，并不是意想中的/dev/sdb，那么单纯把启动项写入/dev/sdb，也就是实际的机器磁盘之后仍然是不能启动的，因为启动之后机器硬盘就被识别为/dev/sda了……

关键地方在于设定写入的磁盘的时候，点击`更改设备`按钮（如果你用的是中文安装环境的话），同时还要设定`BIOS驱动器顺序`这一项目（这里是一个折叠的选项），将驱动器设定为机器磁盘，比如安装的时候U盘是/dev/sda，那么这个项目就要设定为/dev/sdb，这样之后的安装才能正确的完成，机器才会能够启动。

## 远程桌面

作为Server来说是不需要GUI界面的，但是首先来说，为了方便实验，同时后期可能会安装Oracle数据库，那么一个图形界面还是很需要的，安装的时候可以选择Basic Server，同时自行设定软件界面，在`桌面`选项卡中选择上需要的桌面软件，我自己是勾选了KDE以外的所有项目。

GUI界面虽好，但是好几台机器放在脚下，每台机器再接一个显示器，还得有键鼠等设备，我的桌面肯定是放不下了。这时候就需要用上远程桌面了。

因为是做实验，选择了一个自己觉得简单易用的远程桌面软件`TigerVNC`。

安装上来说非常容易，使用`yum`就可以：

```
yum install tigervnc tigervnc-server
```
这个软件的配置文件也是很简单，编辑`/etc/sysconfig/vncservers`，设定一下即可

```
VNCSERVERS="1:root"
VNCSERVERARGS[1]="-geometry 1024x768"
```

简单来说就是设定第1个可登陆用户为root，同时设定分辨率为1024X768。

之后su到root下，通过`vncpasswd`设定登录所需的密码，之后通过客户端连接的时候需要通过这个密码登录，之后通过`vncserver`启动服务，这个软件也把自己注册成了服务，可以考虑使用`service`启动。

连接的时候需要使用端口，这里比较有意思的是这个软件监听的端口与你设定的用户编号有关，也就是用户1是5901，用户2是5902.

连接的客户端可以考虑直接使用[Chrome的VNC Viewer扩展][3]。

![VNC Viewer][4]

再也不用在桌面上摆上一堆显示器键鼠什么的了。

![VNC Viewer connected][5]

# PHP

PHP真是没太多麻烦的地方，然后在装`libmemcached`的时候发现1.0.18版本的libmemcached依然没法在我的CentOS 6.5的机器上成功的安装，没想到上次在Ubuntu上配环境遇到的问题在又遇到了一次，老办法，换[1.0.16][6]版本解决问题。

# 其他

测试的机器使用路由器串联起来的，只需要在路由器上使用一个简单的端口转发配置（家用路由器应该都有），将制定的端口请求绑定到各台路由器已连接的机器上即可。

![端口转发][7]

SSH的话需要多监听一个端口，CeotOS的话修改`/etc/ssh/sshd_config`，解除对`Port 22`这一行的注释，同时加上想要监听的端口即可，即增加一行`Port your-port`，之后重启sshd服务。

同时配置上一条端口转发规则即可，之后用

```
ssh -p port username@router_ip
```
即可。即所有的请求都发向路由器IP就好。


[1]: http://mirror.bit.edu.cn/centos/6.5/isos/x86_64/
[2]: http://wiki.centos.org/zh/HowTos/InstallFromUSBkey
[3]: https://chrome.google.com/webstore/detail/vnc%C2%AE-viewer-for-google-ch/iabmpiboiopbgfabjmgeedhcmjenhbla
[4]: https://blog.wislay.com/wp-content/uploads/2014/10/vncviewer-1024x796.png
[5]: https://blog.wislay.com/wp-content/uploads/2014/10/vncviewer_connected-1024x796.png
[6]: https://launchpad.net/libmemcached/1.0/1.0.16/+download/libmemcached-1.0.16.tar.gz
[7]: https://blog.wislay.com/wp-content/uploads/2014/10/set_port_dispatch-1024x584.png


