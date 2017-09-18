title: Yaf源码概览(Part I)
date: 2017-09-17 21:27:59
tags: [Yaf, PHP]
categories: PHP
---

# TL;DR

`Yaf` 版本为 `2.3.0`。

本篇主要简单记录了：

+ yaf.c
+ yaf_application.c
+ yaf_bootstrap.c
+ yaf_controller.c
+ yaf_dispatcher.c
+ yaf_exception.c
+ yaf_loader.c
+ yaf_plugin.c
+ yaf_registry.c

源码阅读过程中的一些问题和理解。

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

定义 ini 文件中的可配置项目代码段自 `PHP_INI_BEGIN()` 开始，到 `PHP_INI_END()` 结束，通过名如 `STD_PHP_INI_*` 的宏进行设定。

```
PHP_INI_BEGIN()
	STD_PHP_INI_ENTRY("yaf.library",         	"",  PHP_INI_ALL, OnUpdateString, global_library, zend_yaf_globals, yaf_globals)
	...
	STD_PHP_INI_BOOLEAN("yaf.use_namespace",   	"0", PHP_INI_SYSTEM, OnUpdateBool, use_namespace, zend_yaf_globals, yaf_globals)
#endif
PHP_INI_END();
```

这里需要注意的是，宏的第三个参数有 `PHP_INI_ALL` 和 `PHP_INI_SYSTEM` 两种值，这两个值决定了是否可以在运行时修改这类参数。设定为 `PHP_INI_SYSTEM` 的配置项是不允许在运行时改变的。

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
+ 读取解析配置文件

从手册上可以得知构造方法的原型为：

```
public void Yaf_Application::__construct(mixed  $config,
                                         string $section = ap.environ);
```

根据传递的字符串作为本应用 ini 配置文件的文件名，进行解析。

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

根据方法原型中传入的配置文件路径，会解析出如 bootstrap 类所在文件路径之类的配置，并赋值到全局变量中，供其他功能使用。

在读取配置文件的过程中，还会通过 `yaf_loader_register()` 函数注册默认的自动加载方法: `Yaf_Loader::autoload()`。

## run()

run() 方法完成的工作比较简单，判断当前 app 对象是否已经在运行中，如果在运行中则产生错误，否则执行 dispatch 过程，获得 response 对象。

## bootstrap()

bootstrap() 方法是一个比较重要的方法，这个方法可以对 yaf 进行一些全局的初始化操作。

首先会在类名表（`EG(class_table)`）中查找名为 `YAF_DEFAULT_BOOTSTRAP_LOWER` 即名为 `bootstrap` 的类。

如果不存在这样的一个 class，则通过读取名为 bootstrap 的全局变量（`YAF_G(bootstrap)`），来确定具体的需要执行的类所在的文件。如果全局变量也没有配置，则会在当前目录中查找是否存在 `Bootstrap.php` 的文件。bootstrap 这一个全局变量，对应的是传入的 ini 文件中的 `application.bootstrap` 配置项的值。

在确定需要执行的 bootstrap 的文件路径之后，通过 `yaf_loader_import()` 函数加载文件。并会再次尝试在类名表（`EG(class_table)`）中查找名为 `YAF_DEFAULT_BOOTSTRAP_LOWER` 的类，最后判断这个类是否继承了 `Yaf_Bootstrap_Abstract`，任何一项不满足，都会触发错误。

摘录一下上述这段源码：

```
if (!yaf_loader_import(bootstrap_path, len + 1, 0 TSRMLS_CC)) {
	php_error_docref(NULL TSRMLS_CC, E_WARNING, "Couldn't find bootstrap file %s", bootstrap_path);
	retval = 0;
} else if (zend_hash_find(EG(class_table), YAF_DEFAULT_BOOTSTRAP_LOWER, YAF_DEFAULT_BOOTSTRAP_LEN, (void **) &ce) != SUCCESS)  {
	php_error_docref(NULL TSRMLS_CC, E_WARNING, "Couldn't find class %s in %s", YAF_DEFAULT_BOOTSTRAP, bootstrap_path);
	retval = 0;
} else if (!instanceof_function(*ce, yaf_bootstrap_ce TSRMLS_CC)) {
	php_error_docref(NULL TSRMLS_CC, E_WARNING, "Expect a %s instance, %s give", yaf_bootstrap_ce->name, (*ce)->name);
	retval = 0;
}
```

`yaf_loader_import()` 成功加载之后返回值是`1`，`zend_hash_find()` 执行成功之后返回值是 `SUCCESS` 即`0`，`instanceof_function()` 判断为真时返回值是`1`，所以，上述一段代码，在一切正常的逻辑下，三个 if/else 判断中的语句都会执行。

所以，上述步骤说明了两点：

+ bootstrap 文件的绝对路径是可以配置的
+ bootstrap 类名必须是 Bootstrap（因为只会在类名表中查找这一个值）

在类加载完成后，会逐个调用 `_init` 开头的方法，完成初始化操作，所有方法会接收 dispatcher 作为参数。

# yaf_bootstrap.c

这一文件主要是声明了 `Yaf_Bootstrap_Abstract` 这一个抽象类。

# yaf_controller.c

这一文件声明了 `Yaf_Controller_Abstract` 这一个抽象类。同时也定义了视图层的 render 与 display 操作，二者区别在于 render 返回渲染好的视图字符串（准确来说是一个 zval）。

# yaf_dispatcher.c

这是 yaf 执行过程中的关键部分之一。

一个请求到来，Yaf_Application 对象在执行 run() 方法时，最后一步就是通过已经设置好的 dispatcher 开始对请求进行处理，即调用 `yaf_dispatcher_dispatch()` 函数。

## yaf_dispatcher_dispatch()

### 路由

`yaf_dispatcher_dispatch()` 函数在处理过程中，会先判断当前请求是否已经被路由过，如果没有被路由过，则通过 `yaf_dispatcher_route()` 函数对当前请求执行路由操作。

在执行路由之前，yaf 的钩子机制会通过 `YAF_PLUGIN_HANDLE` 这个宏逐个调用已注册插件中 `routerstartup` 回调方法，调用顺序为注册插件的顺序。

Yaf_Dispatcher 中包含一个成员变量 `_router` (即 `YAF_DISPATCHER_PROPERTY_NAME_ROUTER`)，此处记录了当前 app 已注册的各个路由规则。在初始化阶段（调用 `yaf_router_instance()` 函数），会注册默认路由，如果不设定，则使用的是 `static` 路由。

同样的，一个 app 的 Dispatcher 对象的默认 Controller/Module/Action 等等参数，以及需要执行的插件，也都在这一个阶段完成了默认值的设定。

如果有多个路由规则，这里要注意，后加入的路由规则会先被执行，此处在手册中也可以获知，从代码上来看，原因是 `yaf_router_route()` 函数在遍历 HashTable 时从 HashTable 的末端开始进行遍历：

```
...
ht = Z_ARRVAL_P(routers);
for(zend_hash_internal_pointer_end(ht);
		zend_hash_has_more_elements(ht) == SUCCESS;
		zend_hash_move_backwards(ht)) {
...
```

一旦路由规则命中，则结束路由过程。

在执行路由之后，yaf 的钩子机制会通过 `YAF_PLUGIN_HANDLE` 这个宏逐个调用已注册插件中 `routershutdown` 回调方法，调用顺序为注册插件的顺序。

### 分发

分发开始之前，yaf 的钩子机制会通过 `YAF_PLUGIN_HANDLE` 这个宏逐个调用已注册插件中 `dispatchloopstartup` 回调方法，调用顺序为注册插件的顺序。

之后会执行视图的初始化操作。

请求在分发过程中有最大 forward 次数（将请求交给指定的 module / controller / action 处理），即配置文件中 `yaf.forward_limit` 这一个配置项，这个配置可以避免让用户请求陷入无限循环处理的问题之中（如用户权限系统出现bug，无限转入登录逻辑）。

每一次分发过程中，yaf 的钩子机制会先通过 `YAF_PLUGIN_HANDLE` 这个宏逐个调用已注册插件中 `predispatch` 回调方法，之后通过 `yaf_dispatcher_handle()` 函数实际处理请求，请求完成之后在通过 `YAF_PLUGIN_HANDLE` 这个宏逐个调用已注册插件中 `postdispatch` 回调方法。

在请求执行完成之后，向用户发送对应的请求结果。

#### yaf_dispatcher_handle()

`yaf_dispatcher_handle()` 这一方法完成的工作是从 request 对象中得知当前的 `module` 与 `controller`，之后找到对应的文件，实例化对应的 Controller，为对应的 Controller 设定好模板目录等这类基础属性。

之后会从 request 对象中获取 `action`，即实际需要执行的方法。

执行完对应的action之后，如果执行结果的返回值不为真值（非0），则不会执行渲染页面以及输出等工作。

如果 Dispatcher 中设置了名为 `$_auto_render` 且值为真的成员变量（`yaf_dispatcher.h` 中的 `#define	YAF_DISPATCHER_PROPERTY_NAME_RENDER		"_auto_render"`），当前 Controller 可能会触发自动输出。

这里说可能，是因为 Controller 中的成员变量也会影响到这一行为。

如果 Controller 中设置了名为 `$yafAutoRender` 且值为真的成员变量（`yaf_controller.h` 中的 `#define YAF_CONTROLLER_PROPERTY_NAME_RENDER     "yafAutoRender"`），当前 Controller 会触发自动输出。只有 Controller 中没有设定这一个成员变量，Dispatcher 中的配置才会产生影响。

实际上，`Yaf_Dispatcher` 中的 `enableView()`/`disableView()` 方法所做的就是修改这一成员变量的值。

# yaf_exception.c

主要定义了各种类型的异常类。

# yaf_loader.c

Yaf 的又一核心组成部分，代码与类自动加载器。

## import()

import() 方法主要完成的工作是加载对应的 PHP 文件到当前执行环境。实际上仍然调用的是 `yaf_loader_import()` 函数进行加载工作。

## autoload()

当代码遇到当前文件中未定义的类时，需要自动加载器完成对应代码的加载工作。

从这个方法的逻辑可以看到 yaf 自动加载的规律。

在未做特殊配置的情况下，在默认模块下进行开发，简单来说可以概括成如下几个：

+ 代码的起始查找目录都在于在ini中定义的`application.directory`(此处值可以使用PHP代码中的预定义常量)
+ `_`表示目录分隔符，`class Foo_BarBar_Var`等同于目录`Foo/BarBar/Var.php`
+ `Controller`为结尾的类会在`controllers`目录下进行查找
+ `Model`为结尾的类会在`models`目录下进行查找
+ `Plugin`为结尾的类会在`plugins`目录下进行查找
+ 其他类会在`library`目录下进行查找

### use_spl_autoload 配置项作用

手册里面提及：

> 在use_spl_autoload关闭的情况下, Yaf Autoloader在一次找不到的情况下, 会立即返回, 而剥夺其后的自动加载器的执行机会.

从代码执行逻辑上来看，确实如此，[spl_autoload_register()](http://php.net/manual/zh/function.spl-autoload-register.php) 这一函数在注册的回调方法返回 `TRUE` 时，不会调用已注册函数列表中的下一个加载函数。

Yaf 在设置这一个值为空或者关闭时（0）,做法是无论何种情况都返回真值：

```
if (!YAF_G(use_spl_autoload)) {
		/** directory might be NULL since we passed a NULL */
		if (yaf_internal_autoload(file_name, file_name_len, &directory TSRMLS_CC)) {
			char *lc_classname = zend_str_tolower_dup(origin_classname, class_name_len);
			if (zend_hash_exists(EG(class_table), lc_classname, class_name_len + 1)) {
...
				RETURN_TRUE; // 注意
			} else {
				efree(lc_classname);
				php_error_docref(NULL TSRMLS_CC, E_STRICT, "Could not find class %s in %s", class_name, directory);
			}
		}  else {
			php_error_docref(NULL TSRMLS_CC, E_WARNING, "Failed opening script %s: %s", directory, strerror(errno));
		}

...
		RETURN_TRUE; // 注意
	}
```

但是在[集成 Eloquent 的尝试](/articles/2017/08/01/integrate-yaf-with-eloquent/)里，在配置文件中并没有配置这一个项目为1（即开启），按照扩展源码的逻辑，Composer 生成的加载器应该不起作用，这个和实际情况不符，因为 Eloquent 在[样例程序](https://github.com/liaoaoyang/YafWithEloquentSample)中表现正常。

究其原因，其实很简单，即 Composer 生成的自动加载器在注册时要求注册到了加载函数队列的首位。

先来看 `spl_autoload_register()` 方法的原型：

```
bool spl_autoload_register ([ callable $autoload_function [, bool $throw = true [, bool $prepend = false ]]] )
```

让人值得注意的是 `$prepend` 参数：

```
prepend
	如果是 true，spl_autoload_register() 会添加函数到队列之首，而不是队列尾部。
```

进入到 `vendor` 目录，翻看 `autoload_real.php` 源码，可以看到：

```
class ComposerAutoloaderInit4604f3b23b635a9f5adc52f8616258a1
{
    private static $loader;

    public static function loadClassLoader($class)
    {
        if ('Composer\Autoload\ClassLoader' === $class) {
            require __DIR__ . '/ClassLoader.php';
        }
    }

    public static function getLoader()
    {
        if (null !== self::$loader) {
            return self::$loader;
        }

        ...

        $loader->register(true); // 注意，这里为 true
        
        ...

        return $loader;
    }
}
```

这个 register 方法的定义为：

```
/**
 * Registers this instance as an autoloader.
 *
 * @param bool $prepend Whether to prepend the autoloader or not
 */
public function register($prepend = false)
{
    spl_autoload_register(array($this, 'loadClass'), true, $prepend);
}
```

至此，可以看到，我们通过 Composer 生成的自动加载方法，实际上会优先于 yaf 自身的自动加载方法，由于 yaf 的自动加载方法也是通过 `spl_autoload_register()` 方法注册的，处于同一个加载函数队列，在 Composer 声明优先的情况下，加载函数执行顺序就会发生变化。

# yaf_plugin.c

插件抽象类 `Yaf_Plugin_Abstract` 的定义。在这里，可以看到一个插件可以实现的 hook 方法：

```
zend_function_entry yaf_plugin_methods[] = {
	PHP_ME(yaf_plugin, routerStartup,  		 plugin_arg, ZEND_ACC_PUBLIC)
	PHP_ME(yaf_plugin, routerShutdown,       plugin_arg, ZEND_ACC_PUBLIC)
	PHP_ME(yaf_plugin, dispatchLoopStartup,  plugin_arg, ZEND_ACC_PUBLIC)
	PHP_ME(yaf_plugin, dispatchLoopShutdown, plugin_arg, ZEND_ACC_PUBLIC)
	PHP_ME(yaf_plugin, preDispatch,  		 plugin_arg, ZEND_ACC_PUBLIC)
	PHP_ME(yaf_plugin, postDispatch, 		 plugin_arg, ZEND_ACC_PUBLIC)
	PHP_ME(yaf_plugin, preResponse, 		 plugin_arg, ZEND_ACC_PUBLIC)
	{NULL, NULL, NULL}
};
```

他们都只接受两个参数：`request` 与 `response`。

```
ZEND_BEGIN_ARG_INFO_EX(plugin_arg, 0, 0, 2)
	ZEND_ARG_OBJ_INFO(0, request, Yaf_Request_Abstract, 0)
	ZEND_ARG_OBJ_INFO(0, response, Yaf_Response_Abstract, 0)
ZEND_END_ARG_INFO()
```

# yaf_registry.c

Yaf 全局存储类 `Yaf_Registry` 定义。用于在整个 app 声明周期内存储共有数据。所有数据都会存储到一个 zval 之中。

# 参考资料

+ [phpbook](https://github.com/walu/phpbook)

