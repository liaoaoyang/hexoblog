title: 从Pelican到Hexo
date: 2015-12-06 20:44:27
categories: Try
tags: Hexo,Pelican
---

# 起因

用Pelican在GitHub上搭blog有段时间了，一直想要更清爽简单的blog解决方案，之前使用的Pelican算是满足了我的需求，但是还想尝试一下其他的系统，同时从视觉效果上来说Hexo+[`Next`][1]主题目前更让我觉得满意，于是决定从Pelican迁移到Hexo。

# 问题

搭建、使用Hexo的教程google一下就能找到，这里主要说一下自己迁移过程中遇到的一些小问题。

## 文档迁移

Pelican本身也是支持Markdown的的文章写作方式，其实需要修正的地方主要是头部的部分属性。

首先可以将Pelican的`content`目录下的所有Markdown文件复制到Hexo目录下的`source/_posts/`目录下。

Pelican通过`Date`，`Title`，`Category`，`Tags`，`Slug`来表征写作时间、标题、分类、标签、Url等信息，例如：

```
Date: 2014-09-20
Title: CI连接Oracle 11G数据库
Category: LAMP/LNMP
Tags: PHP,Oracle
Slug: ci-connect-oracle-11g
```

而Hexo也是类似：

```
date: 2014-09-20 00:00:00
title: CI连接Oracle 11G数据库
categories: LAMP/LNMP
tags: [PHP,Oracle]
---
```

可以简单使用`sed`完成转换：

```
sed -i '' -e 's/Date: \{1,\}\(.\{1,\}$\)/date: \1 00:00:00/' *
sed -i '' -e 's/Title: \{1,\}\(.\{1,\}$\)/title: \1/' *
sed -i '' -e 's/Category: \{1,\}\(.\{1,\}$\)/categories: \1/' *
sed -i '' -e 's/Tags: \{1,\}\(.\{1,\}$\)/tag: [\1]/' *
sed -i '' -e 's/Slug: \{1,\}\(.\{1,\}$\)/---/' *
```

关于sed -i后的参数问题，参见`stackoverflow`上关于在Mac OS X下的sed使用[`问题`][2]。

## Git设置

### 允许Git添加非ASCII文件

在`hexo deploy`过程中会在Git中在添加非ascii文件，所以需要在选项中开启这一设置。

```
git config hooks.allownonascii true
```

### Git中文文件名

某些情况下文章使用了中文tag，而生成Tag时会生成中文文件名的目录文件，Git需要关闭对中文文件名的转码。

```
git config core.quotepath false
```

### GitHub CNAME问题

使用GitHub部署的情况下，绑定自定义域名的方法自然是添加一个CNAME文件，最简单的方法可以使用[`插件`][3]来完成。

# 最后

以上


[1]: https://github.com/iissnan/hexo-theme-next
[2]: http://stackoverflow.com/questions/19456518/invalid-command-code-despite-escaping-periods-using-sed
[3]: https://github.com/leecrossley/hexo-generator-cname
