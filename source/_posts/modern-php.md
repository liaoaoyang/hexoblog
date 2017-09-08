title: 《Modern PHP》
date: 2017-07-25 21:36:32
tags: [PHP]
categories: PHP
---

PHP也是一门在不断变化的语言。

本书介绍了PHP 5.3之后出现的一些新特性，对PSR标准做了介绍，但是因为年代（2015年出版）的问题，PSR标准建议到官网进行阅读，本书最后还对配置、部署方面提出了一些建议。

<!-- more -->

# 新特性

5.3之后的PHP引入了很多有趣的新特性。

1.命名空间在PHP5.3被引入，使用反斜杠`\`作为分隔符，通过类似文件目录层级的形式组织PHP代码，主要实现了两大特性：类名字的唯一性，更接近与文件系统的自动加载规则（相比起之前通过`_`作为目录分隔符的编码方式）。

2.作者推崇PHP编程也要面向接口编程。面向接口让代码更为灵活，一次约定，任意遵循约定的实现皆可使用。

3.`Traits`从单词词义上为特征，实际效果即可复用的代码块（类似`mixins`），在PHP5.4被引入。Traits让没有关系的两个类实现了代码复用。Traits和类很相似，但是并不能实例化。

4.`Generators`在PHP5.5中被引入，为低成本的操作大数据集提供了可能性，一个常用的应用场景，如逐行读取大文件，在每次读取一个文件时，唤醒上下文，读取部分文件记录，可以降低处理过程中的内存占用。`yield`只能在函数中使用，返回一个Generators对象。生成器是不可回溯的（只能不断地向前推进到下一个返回值，虽然提供了[rewind()](http://cn2.php.net/manual/en/generator.rewind.php)方法）。关于Generator有一篇有趣的[文章](http://gywbd.github.io/posts/2014/12/understand-generator-in-php.html)。

5.闭包`Closures`在PHP5.3被引入，即匿名函数。匿名函数两个有意思的特性，一个是增加属性值，即通过`use`关键字提供附加参数，一个是通过`bindTo()`方法传入作用域，其中[bindTo()](http://cn2.php.net/manual/en/closure.bindto.php)方法的第二个参数需要指定操作的类名称，否则无法让闭包置身于设定了的上下文之中。

6.内置的HTTP Server，在PHP5.4之后，还有什么比`php -S ip:port`更直接迅速的启动一个HTTP Server来测试你的API呢？

# 其他

1.[PSR（PHP Standards Recommendation）](http://www.php-fig.org/)为了解决框架的互操作性，指定了一些关于接口、自动加载还有代码风格的推荐标准，除了书中提到的PSR0-4标准，目前已经存在的[标准](http://www.php-fig.org/psr/)有：


| 编号 | 标准 |
| --- | --- |
| 1 | [Basic Coding Standard](http://www.php-fig.org/psr/psr-1/) |
| 2 | [Coding Style Guide](http://www.php-fig.org/psr/psr-2/) |
| 3 | [Logger Interface](http://www.php-fig.org/psr/psr-3/) |
| 4 | [Autoloading Standard](http://www.php-fig.org/psr/psr-4/) |
| 6 | [Caching Interface](http://www.php-fig.org/psr/psr-6/) |
| 7 | [HTTP Message Interface](http://www.php-fig.org/psr/psr-7/) |
| 11 | [Container Interface](http://www.php-fig.org/psr/psr-11/) |
| 13 | [Hypermedia Links](http://www.php-fig.org/psr/psr-13/) |
| 16 | [Simple Cache](http://www.php-fig.org/psr/psr-16/) |


按照PSR标准编写的类库，可以较大程度上的与其他项目协同工作。当中最重要的，个人觉得应为PSR4与PSR7标准，一个统一了加载方式，一个确定了操作PHP API的方法。

2.对于PSR4，只需要记住一行代码就可以了：

```
require $base_dir . str_replace('\\', '/', str_replace($class_prefix, '', $class_name)) . php
```

即类文件目录为将命名空间前缀替换为实际代码的起始目录，并将命名空间的反斜杠替换成目录分隔符即可。

3.想要创建自己的组件，可以参考[https://github.com/thephpleague/skeleton](https://github.com/thephpleague/skeleton)。简单来说，在`src`目录中放置自身的代码，在`tests`目录中存放测试用例，通过`composer.json`文件声明名称、依赖、自动加载器等组件信息。

当然了，作为PHPer，是时候读读[PHP The Right Way](http://www.phptherightway.com/)了。


