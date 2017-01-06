date: 2015-08-21 00:00:00
title: emoji，json_encode与MySQL
categories: PHP
tags: [PHP,json_encode,emoji]
---

## 前言

之前有看到过对于emoji表情这类字符，想要存入MySQL之中，需要建立或者修改表的字符集为`utf8mb4`，而且对MySQL版本还有一定的要求（[`>=5.5.3`][3]），否则MySQL会提示类似：

```
ERROR 1366: Incorrect string value: '\xF0\x9F\x8D\xBA' for column
```

的错误。

然而在PHP中其实有一种情况是可以“存储”emoji表情字符的，那就是将包含emoji的字符串通过PHP的`json_encode`方法处理后再进行存储操作。

## emoji不能直接存入UTF-8字符集的表中的原因

根据维基百科上对于[`字符编码`][1]的定义，我们可以将`字符集`和`字符编码`看做同义词，后面二者将会不加区分的使用。

emoji不能直接存入UTF-8字符集的表中的原因，就得从emoji是什么说。

[`emoji`][2]简而言之就是若干组`Unicode`字符，随着iPhone、微博、微信等硬件、软件的支持、语普及，emoji也算是越来越常见。

这类字符因为其码位值的原因（`U+1F300..1F545`，`U+1F600..1F641`，`U+1F300-1F5FF`，`U+1F600-1F64F`，`U+1F300-1F5FF`，`U+1F600-1F64F`），是无法通过3字节长度的UTF-8编码表示的。

以啤酒（`BEER MUG`）符号举例，这个符号的Unicode的码位值为`U+1F37A`，那么转换成UTF-8编码则是：

`0xF0` `0x9F` `0x8D` `0xBA`

明显可以看出需要通过4个字符进行表示，而在MySQL中，一般DBA给定的默认字符集都是UTF-8，而MySQL的文档中写道：


>The utf8 character set is the same in MySQL 5.6 as before 5.6 and has exactly the same characteristics:

>No support for supplementary characters (BMP characters only).

>A maximum of three bytes per multibyte character.

也就是说直到5.6及以前的MySQL的UTF-8字符集最大只支持3个字符，那么emoji字符无法直接存入也是可以理解的了。

## emoji转化后进行存储

从上一主题可以看到，不能直接存储emoji字符的话，那么是不是可以通过其他间接手段对emoji字符进行存储呢？

答案是肯定的，如果能够对emoji字符转化成为其编码的字符串之后进行存储，在展示时从字符串转化为原始字符即可，或者是将其转化成为一些特别的字符串进行[处理][5]，总而言之就是

```
原始字符->字符串->MySQL->字符串->原始字符
```

的过程。

## json_encode与emoji

在实际项目中，JSON是常见一种数据格式，很多结构化的数据可以通过JSON进行存储，而PHP的JSON处理极其简便，配合array简直不能更轻松，很多情况下，数组数据可以通过json_encode之后当做字符串存入MySQL中。

然而这与emoji有什么关系呢？

答案是json_encode会对作为数组中值的emoji字符进行转化，之后转化结果字符串可以方便的存入数据库中。

翻阅在使用的`PHP 5.4.24`的源码，阅读json扩展的源码`ext\json\json.c`，可以看到`php_json_encode`函数中的位于文件的`629`~`631`行：

```
case IS_STRING:
	json_escape_string(buf, Z_STRVAL_P(val), Z_STRLEN_P(val), options TSRMLS_CC);
	break;
```

跟进到`json_escape_string`函数，可以看到文件的`422`行：

```
ulen = json_utf8_to_utf16(utf16, s, len);
```

即会将UTF-8编码的字符转化为UTF-16的编码。

之后`423`到`538`行将转化结果添加到返回的字符串`buf`中（详情参见PHP源代码文件）。

实际操作一下：

![emoji_json_encode_test][6]


这样就解释了为什么经过json_encode处理之后的字符可以存入MySQL中，当然，这样处理之后的成本之一就是存储的字节数的增加了。


[1]: https://zh.wikipedia.org/wiki/%E5%AD%97%E7%AC%A6%E7%BC%96%E7%A0%81
[2]: https://zh.wikipedia.org/wiki/%E7%B9%AA%E6%96%87%E5%AD%97
[3]: https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html
[4]: http://dev.mysql.com/doc/refman/5.6/en/charset-unicode-utf8.html
[5]: http://stackoverflow.com/questions/3220031/how-to-filter-or-replace-unicode-characters-that-would-take-more-than-3-bytes
[6]: https://blog.wislay.com/wp-content/uploads/2015/08/emoji_json_encode_test.png
