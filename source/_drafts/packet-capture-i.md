title: 抓包那点事儿（常用工具简介）
date: 2018-07-01 11:14:53
tags: [抓包,开发工具]
categories: Try
---

# TL;DR

抓包是常用的问题排查手段，通过获取网络通讯内容的方式分析应用工作情况，辅助定位问题。

抓包工具相当多，PC/服务端常见的抓包工具 `tcpdump`/`Wireshark`/`Fiddler`/`Charles`/`whistle`/`mitmproxy` 。

本篇只做简要介绍，具体工具使用再起篇幅。

<!-- packet-capture-i -->
<!-- more -->

# 工具

## 对比

个人对之前提及的工具做一个简单的对比：

| 特性/软件 | tcpdump | Wireshark | Fiddler | Charles | whistle | mitmproxy |
| --- | --- | --- | --- | --- | --- | --- |
| **HTTP(S)协议展现效果** | 无 | 一般 | 好 | 好 | 好 | 好 |
| **TCP/UDP协议展现效果** | 一般 | 好 | 不支持 | 不支持 | 不支持 | 不支持 |
| **协议支持** | 多 | 多 | 主要支持HTTP(S) | 主要支持HTTP(S) | 主要支持HTTP(S) | 主要支持HTTP(S) |
| **安装配置** | 简单 | 简单 | 简单 | 简单 | 一般 | 一般 |
| **支持平台** | 主要是*nix | Windows/Mac/Linux | Windows（其他平台需要Mono） | Windows/Mac/Linux | 基于node.js | 基于Python |
| **费用** | 免费 | 免费 | 免费 | 收费 | 免费 | 免费 |
| **GUI** | 无 | 有 | 有 | 有 | 有（基于浏览器） | 有（基于浏览器） |
| **源代码** | 公开 | 公开 | 不公开 | 不公开 | 公开 | 公开 |

## tcpdump

经典的服务端抓包工具，在各大 Linux 发行版软件仓库中均有提供，主要用于终端下的抓包排查问题。

[tcpdump](https://www.tcpdump.org/manpages/tcpdump.1.html) 顾名思义，直接能够抓取 TCP 协议的数据包，对于 Web 应用开发者来说已经足够底层，可以用于排查各种 DNS 以及更上层网络协议中出现的问题。

tcpdump 基于 libpcap 实现，是全能型的网络抓包选手。

# 参考

+ https://github.com/avwo/whistle
+ [libpcap实现机制及接口函数](https://www.jianshu.com/p/ed6db49a3428)





