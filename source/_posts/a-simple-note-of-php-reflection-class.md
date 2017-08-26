title: 用PHP反射类导出类的所有常量
date: 2016-05-07 16:56:41
categories: PHP
tags: [PHP]
---

# 概述

使用PHP作为API开发语言时，可以将状态码定义为类的常量。

如果同时存在 Android 与 iOS 客户端，相信状态码的维护、沟通多少会带来一些成本，同步状态码也比较费事，且不说存在概念不一致，拼写错误（譬如当年的 `Referer`）等问题。

考虑到这个问题，为什么不试着直接通过服务端的状态码生成各自语言所需的状态码定义文件呢？

在编写生成脚本过程中，PHP的状态码全部是类的常量，如何一次性导出这些常量成为了一个问题。不过PHP也提供了[`反射类`](http://php.net/manual/en/class.reflectionclass.php)，可以一次性获取所有的常量。

<!-- more -->

# 反射类

PHP 的[`反射类`](http://php.net/manual/en/class.reflectionclass.php) 接受字符串类名以及类作为构造方法的参数，通过生成的对象探明传入的类中包含的常量，方法等(private 也能探明)。

这里使用的正是它的 `getConstants` 方法。

其他方法参见[手册]((http://php.net/manual/en/class.reflectionclass.php))。

# 转换脚本

这里稍微提及一下转换脚本。

本身这个工作相当简单，仅仅是一个文本替换工作，其实完全通过 sed 等 Linux 工具即可完成。

但是，通过 PHP 完成的话，利用反射类这一机制，结合 PHP 的各类字符串操作方法，能让文件有更多的变化，甚至完成替换版本库文件等操作，当然作为 PHP 开发者使用熟悉的语言自然是极好的。

```php
<?php

define('OUTPUT_PATH', '/tmp');

require dirname(__DIR__) . '/path/to/your/class/defination/StatusCode.php';

$objcTpl =<<<TPL
//
//  StatusCode.h
//
//  Created by ng on %s.
//

#ifndef StatusCode_h
#define StatusCode_h

%s

#endif /* StatusCode_h */
TPL;

$androidTpl =<<<TPL
package your.package;

/**
 * Created by ng on %s.
 */
public class StatusCode {
%s
}

TPL;


$s = new ReflectionClass(\NameSpace\Of\Your\StatusCode::class);

$constants = $s->getConstants();
$longestWordLength = 0;

foreach ($constants as $k => $v) {
    if (strlen($k) > $longestWordLength) {
        $longestWordLength = strlen($k);
    }
}

/**
 * ObjC
 * Android
 */
$objCDefineContent = '';
$androidClassContent = '';

foreach ($constants as $k => $v) {
    $spaceLength = $longestWordLength - strlen($k);
    $spaceLength = $spaceLength <= 0 ? 1 : $spaceLength + 1;
    $objCOneLineTpl = "#define StatusCode_%s%{$spaceLength}s@\"%s\"\n";
    $objCDefineContent .= sprintf($objCOneLineTpl, $k, ' ', $v);
    $androidOneLineTpl = "\tpublic static final String %s%{$spaceLength}s= \"%s\";\n";
    $androidClassContent .= sprintf($androidOneLineTpl, $k, ' ', $v);
}

file_put_contents(OUTPUT_PATH . '/StatusCode.h', sprintf($objcTpl, date('Y/m/d'), trim($objCDefineContent, "\n")));
file_put_contents(OUTPUT_PATH . '/StatusCode.java', sprintf($androidTpl, date('Y/m/d'), trim($androidClassContent, "\n")));
```

转换后可以获得 `StatusCode.h` 与 `StatusCode.java` 两个文件。

# 通过反射调用构造方法

一般的，我们可以通过call_user_func之类的内置方法调用一个对象的方法，然而，如果有相同基类的一系列类，并且构造方法参数列表不相同，需要根据特定情况调用构造方法构造对象并执行方法可以怎么办？是可以写成多个if/else的。

也可以使用反射机制，通过反射类的newInstanceArgs方法，传入构造参数数组。

```
$subTasks = [
    'ClassA' => ['foo', 'bar'],
    'ClassB' => ['foo', 'bar', 1],
];

foreach ($subTasks as $className => $params) {
	$ref = new ReflectionClass($className);
	$model = $ref->newInstanceArgs($params);
	$model->run(); // run是一个公共方法

}
```

以上。


