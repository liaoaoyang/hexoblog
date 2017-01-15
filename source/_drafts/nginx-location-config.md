title: Nginx location
date: 2017-01-15 00:13:16
tags: [Nginx]
---

# 概述

本文代码版本为`1.11.8`，手册原文来自于[官网](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)。

# location

> Syntax:	location [ = | ~ | ~* | ^~ ] uri { ... }
location @name { ... }
Default:	—
Context:	server, location

> Sets configuration depending on a request URI.

设定针对请求URI的各个配置项。

> The matching is performed against a normalized URI, after decoding the text encoded in the “%XX” form, resolving references to relative path components “.” and “..”, and possible compression of two or more adjacent slashes into a single slash.

匹配操作会在规范化的URI上进行，归一化操作包含对“%XX”形式的内容进行解码，找到相对路径实际表示的内容，或者是将多个相邻的`/`变化为一个`/`。

**注：**

源代码中通过`ngx_http_parse_complex_uri`函数完成上述这些功能的。

这一函数主要是针对请求uri做分析，逻辑上是如下枚举的状态数值的迁移：

```
enum {
   sw_usual = 0,
   sw_slash,
   sw_dot,
   sw_dot_dot,
   sw_quoted,
   sw_quoted_second
} state, quoted_state;
```

这一个方法之中有一个很有趣的实现，即：

```
if (usual[ch >> 5] & (1U << (ch & 0x1f))) {
	state = sw_usual;
	*u++ = ch;
	ch = *p++;
	break;
}
```

当中的`usual[ch >> 5] & (1U << (ch & 0x1f))`可以视为`判断常规字符操作`，`usual`是一个静态的数组，有8个`uint32`数据的成员，通过位上的`0/1`表示是否是常规字符，这个可以看做一个8*32=256的矩阵，正好覆盖ASCII字符范围，`ch >> 5`获得行号，`1U << (ch & 0x1f)`获得列号。

简而言之：

+ 读取uri字符的指针记为`p`，写入uri数据字符的指针记为`u`（事实上也是代码中的变量名）；
+ 逐个读取uri字符过程中遇到`?`，就会处理HTTP请求参数；遇到`#`就会终止解析（`#`即锚点）；
+ `sw_usual`是常规状态，操作是将uri中的字符填写到uri数据的对应位置（即函数的第一个参数`ngx_http_request_t *r`中`r->uri.data`之中）。先进行`判断常规字符操作`，如果是常规字符，则维持`sw_usual`状态；如果遇到`/%`这几个字符，则会有特殊处理，即做状态迁移；
+ `sw_slash`处理`/`的情况，先进行`判断常规字符操作`，如果是常规字符，则转移`sw_usual`状态；如果设定了需要合并`/`，则推动`p`继续前行，读取下一个字符，否则会记录当前的字符，推动`u`前行，准备写入下一个字符；类似的，如果遇到`.%`这几个字符，则会有特殊处理，即做状态迁移；
+ `sw_dot`处理`.`的情况，先进行`判断常规字符操作`，如果是常规字符，则转移`sw_usual`状态；类似的，如果遇到`/.%`这几个字符，则会有特殊处理，即做状态迁移；再次遇到`.`的处理比较特殊，会迁移到`sw_dot_dot`状态；
+ `sw_dot_dot`处理连续的`.`的情况，如果遇到`/`，则认为是上层目录，会进行目录的回退工作，例如uri为`http://a.net/b/c/../`，在`sw_dot_dot`状态的处理过程中，会将u回退5个位置，即从最后一个`/`回退到`c`字符处，此时uri可以看做是`http://a.net/b/c`，之后继续回退，直到遇到上一个`/`字符为止，经过上述这些处理，最后uri变为`http://a.net/b/`；
+ `sw_quoted`处理形如`%XX`表示的数据；
+ `sw_quoted_second`则处理形如`%XX`表示的数据中的第二个X部分的数据；


