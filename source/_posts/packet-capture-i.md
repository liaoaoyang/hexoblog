title: 抓包那点事儿（常用工具简介）
date: 2018-11-01 20:00:00
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

# 参考

+ [whistle](https://github.com/avwo/whistle)
+ [tcpdump](https://www.tcpdump.org/manpages/tcpdump.1.html)
+ [libpcap实现机制及接口函数](https://www.jianshu.com/p/ed6db49a3428)
+ [Charles 从入门到精通| 唐巧的博客](https://blog.devtang.com/2015/11/14/charles-introduction/)





