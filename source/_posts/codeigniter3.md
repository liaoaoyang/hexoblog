title: CodeIgniter3
date: 2017-07-22 22:31:01
tags: [PHP, CodeIgniter]
categories: PHP
---

# TL;DR

CodeIgniter3是一个相当轻量、简便的并且上手难度低的PHP应用开发框架。目前最新版本是`3.1.5`。优点个人认为有：

+ 轻量
+ 对MySQL查询有较为友好的代码编写方式
+ 功能扩展较为简便
+ 可以支持较低版本的PHP（5.3.7+）
+ 可以不使用模板引擎

同时，个人也认为以下功能还可以有所变化：

+ 内置日志功能不够强大
+ 未使用命名空间
+ RESTful API开发支持

文章的剩余内容将会针对以上的各个方面详细说明。

<!-- more -->

# 优点

## 轻量

### 文件数量

首先通过cloc统计以下代码数量：

```
→ cloc .
     441 text files.
     395 unique files.
      57 files ignored.

github.com/AlDanial/cloc v 1.72 T=4.33 s (88.9 files/s, 45382.0 lines/s)
---------------------------------------------------------------------
Language           files          blank        comment           code
---------------------------------------------------------------------
HTML                 168          16847            167         107992
PHP                  198           8363          29905          29982
JavaScript             8            154            306           1420
XML                    4              0              0            552
CSS                    5            128             31            536
Markdown               1             38              0             57
JSON                   1              0              0             23
---------------------------------------------------------------------
SUM:                 385          25530          30409         140562
---------------------------------------------------------------------

```

仅仅198个php文件，对于一个拥有完整的数据库查询支持，表单验证，Session支持，XSS/CSRF防护的框架来说，并不算臃肿。

文件数量一定程度上反映了层次划分的粒度，如果文件数量过多，带来的首先是源码阅读的难度和改造的难度。

### 基本业务功能实现

对于一个号称轻量的框架来说，实现一个数据库的CRUD操作的Hello World的代码量应该最能反映问题。

CI3实现这样的一个Controller，需要多少代码量呢？答案因人而异。但是步骤上却是相当简单的。考虑如下数据表：

```
CREATE TABLE `ci3_test1` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```





