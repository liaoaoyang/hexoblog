title: Nginx location 从手册到源码（Part I）
date: 2017-10-30 00:13:16
tags: [Nginx]
categories: LAMP/LNMP
---

# 概述

本文代码版本为`1.11.8`，手册原文来自于[官网](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)。

<!-- more -->

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

源代码中通过`ngx_http_parse_complex_uri()`函数完成上述这些功能的。

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

> A location can either be defined by a prefix string, or by a regular expression. Regular expressions are specified with the preceding “~*” modifier (for case-insensitive matching), or the “~” modifier (for case-sensitive matching). To find location matching a given request, nginx first checks locations defined using the prefix strings (prefix locations). Among them, the location with the longest matching prefix is selected and remembered. Then regular expressions are checked, in the order of their appearance in the configuration file. The search of regular expressions terminates on the first match, and the corresponding configuration is used. If no match with a regular expression is found then the configuration of the prefix location remembered earlier is used.

一个location配置要么被定义为uri的前缀字符串，要么通过正则表达式表示。正则表达式之前的符号可以是`~*`（大小写无关的情况），或者`~`（大小写敏感的情况）。为了找到请求匹配的location设置，Nginx首先通过前缀匹配的方式，查找location设置。对于各个location配置，拥有最长匹配长度的location配置将会胜出，并临时记录下来。（如果条件允许的话）之后会接着检查配置文件中正则表达式表示的location配置。正则表达式比较一旦匹配成功，将会结束正则表达式阶段的匹配查找，匹配的正则表达式表示的location配置将会胜出。如果没有正则表达式配置完成了匹配，那么之前被记录的前缀字符串的匹配结果的配置项将会被采用。

**注：**

众所周知，Nginx配置文件中`location`指令的顺序不影响处理顺序，Nginx在处理`location`命令时的顺序，决定于Nginx在通过配置文件生成对应的数据结构时的处理逻辑。

Nginx存储配置文件中的`location`指令时，使用的是的双向链表`ngx_queue_s`：

```
struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

链表中的数据在`ngx_http_init_locations()`函数中会先进行排序，即通过`ngx_queue_sort()`函数完成排序，而排序的比较函数是`ngx_http_cmp_locations()`，这个函数的实现，影响了Nginx各类`location`指令的遍历顺序。

从`location`配置对应的数据结构说起，数据结构中的部分结构需要注意：

```
struct ngx_http_core_loc_conf_s {
    ngx_str_t     name;          /* location name */

#if (NGX_PCRE)
    ngx_http_regex_t  *regex;
#endif

    unsigned      noname:1;   /* "if () {}" block or limit_except */
    unsigned      lmt_excpt:1;
    unsigned      named:1;

    unsigned      exact_match:1;
    unsigned      noregex:1;
```

如`struct ngx_http_core_loc_conf_s`的代码所示：

+ `name`表示的是location配置中的uri规则部分
+ `noname`表示的是常规的location配置，Nginx将`if () {}`以及`limit_except`也视作一个location配置
+ `named`表示是有名location配置，即常见的`@`符号开头的location配置
+ `exact_match`表示这一个配置是否需要精准匹配，即`=`符号开头的location配置
+ `noregex`表示是否是正则匹配
+ `*regex`则是指向Nginx正则表达式的指针

再说排序实现：

```
void
ngx_queue_sort(ngx_queue_t *queue,
    ngx_int_t (*cmp)(const ngx_queue_t *, const ngx_queue_t *))
{
    ngx_queue_t  *q, *prev, *next;

    q = ngx_queue_head(queue);

    if (q == ngx_queue_last(queue)) {
        return;
    }

    for (q = ngx_queue_next(q); q != ngx_queue_sentinel(queue); q = next) {

        prev = ngx_queue_prev(q);
        next = ngx_queue_next(q);

        ngx_queue_remove(q);

        do {
            if (cmp(prev, q) <= 0) {
                break;
            }

            prev = ngx_queue_prev(prev);

        } while (prev != ngx_queue_sentinel(queue));

        ngx_queue_insert_after(prev, q);
    }
}
```

实际上是一个对链表使用稳定[插入排序](https://zh.wikipedia.org/wiki/%E6%8F%92%E5%85%A5%E6%8E%92%E5%BA%8F)算法进行处理。`prev`与`q`两个location配置进行排序操作，当`prev`与`q`在`cmp`函数中返回值是`<=0`时，`q`的顺序会被认为比较靠后，会在`prev`之后插入`q`。否则会会认为`prev`变量指向的节点比`q`指向的节点位置更靠后，继续向前寻找`q`的插入位置。

设`A`和`B`是两个`location`配置，`ngx_http_cmp_locations()`这一函数的排序算法可以简单概括如下：

+ 先判断是否是有名location配置，有名location配置在链表中排序靠后
+ 对于都是有名location配置的情况，名称长度大的位置在链表中排序靠后
+ 对于无名location配置，带有正则判断的location配置在链表中排序靠后
+ 对于无名且非正则location配置，通过文件名比较顺序（匹配内容为uri，如果是Windows这类大小写不敏感的系统则会同一变为小写字母处理）
+ 当文件名比较结果也相同时，精准匹配优先

# 参考

+ [Nginx 源代码笔记 - URI 匹配](http://ialloc.org/posts/2013/08/21/ngx-notes-http-location/)




