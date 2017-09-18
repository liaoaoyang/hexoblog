title: Yaf——源码概览
date: 2017-09-17 21:27:59
tags: [Yaf, PHP]
categories: PHP
---

# TL;DR

`Yaf` 版本为 `2.1.8`。

<!-- more -->

# config.m4

扩展源码阅读，从 `config.m4` 文件开始。

对于这个文件，最值得注意的应该是 `PHP_NEW_EXTENSION` 这一个函数声明了。声明了这一个扩展的名称是 `yaf`，需要编译 `yaf.c` 等多个文件，以及是否 build 到 PHP 的二进制文件中。

# yaf.c

`yaf.c` 这一个文件中，主要做了如下的工作：

+ 定义各个生命周期的回调函数（MINIT/RINIT/RSHUTDOWN/MSHUTDOWN）
+ 定义 ini 中可配置的项目
+ 声明依赖
+ 加载所需模块

## 定义各个生命周期的回调方法

PHP 扩展的生命周期可以简单的概括为如下几个步骤：

+ MINIT 
+ RINIT
+ RSHUTDOWN
+ MSHUTDOWN

其中 RINIT/RSHUTDOWN 在每次请求 PHP 代码执行过程中都会执行一次。

资源和全局的一些初始化工作可以在 MINIT 回调函数中进行。如 yaf 的 MINIT 方法中，就完成了声明 ini 可配置项目（`REGISTER_INI_ENTRIES()`），常量定义，以及模记载。

与 MINIT 相对的，MSHUTDOWN 阶段的回调函数则做了资源释放额操作，如释放了读取配置所需要使用的内存空间。
 

## 定义 ini 中可配置的项目

定义 ini 文件中的可配置项目代码段自 `PHP_INI_BEGIN()` 开始，到 `PHP_INI_END();` 结束，通过名如 `STD_PHP_INI_*` 的宏进行设定。

```
PHP_INI_BEGIN()
	STD_PHP_INI_ENTRY("yaf.library",         	"",  PHP_INI_ALL, OnUpdateString, global_library, zend_yaf_globals, yaf_globals)
	...
	STD_PHP_INI_BOOLEAN("yaf.use_namespace",   	"0", PHP_INI_SYSTEM, OnUpdateBool, use_namespace, zend_yaf_globals, yaf_globals)
#endif
PHP_INI_END();
```

这里需要注意的是，宏的第三个参数有 `PHP_INI_ALL` 和 `PHP_INI_SYSTEM` 两种值，这两个值决定了是否可以在运行时修改这类参数。设定为 `PHP_INI_SYSTEM` 的是不允许在运行时改变的。

所以，想要启用 yaf 的命名空间模式，就必须在 ini 中进行开启。

# yaf_application.c

这一文件主要定义了 `Yaf_Application` 这一个 class。

定义的内容包括：

+ 类名与命名空间名称
+ 类属性与访问权限控制（Yaf_Application 这一个 class 被定义为 final class，即不能再被继承）
+ 类的方法与访问权限控制

除了上述内容，还实现了配置项解析和初始化功能。下面会对一些重要的方法进行简单的描述。

## __construct()

构造方法中主要完成了如下的一些工作：

+ 解析构造方法中的参数
+ 初始化 request/dispatcher/loader 等对象

从手册上可以得知构造方法的原型为：

```
public void Yaf_Application::__construct(mixed  $config,
                                         string $section = ap.environ);
```

根据传递的字符串作为框架 ini 文件的文件名，进行解析。

第二个参数 section 存在的情况下，会只读取 `section:` 开头的配置项目。

如，设定 section 为 product 之后，读取到的数据库端口配置项应为 3306。

```
[common]
application.directory = APPLICATION_PATH  "/application"
application.dispatcher.catchException = TRUE

[product:common]
database.driver    = "mysql"
database.host      = "127.0.0.1"
database.port      = "3306"
database.database  = "learn"
database.username  = "learn"
database.password  = "123456"
database.charset   = "utf8"
database.collation = "utf8_general_ci"
database.prefix    = ""

[dev:common]
database.driver    = "mysql"
database.host      = "127.0.0.1"
database.port      = "13306"
database.database  = "learn"
database.username  = "learn"
database.password  = "123456"
database.charset   = "utf8"
database.collation = "utf8_general_ci"
database.prefix    = ""
```

# 参考资料

+ [phpbook](https://github.com/walu/phpbook)

