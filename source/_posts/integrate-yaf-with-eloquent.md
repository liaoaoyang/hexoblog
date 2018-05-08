title: Yaf集成Eloquent
date: 2017-08-1 11:46:00
tags: [PHP, Yaf, Eloquent, Laravel]
categories: PHP
---

# TL;DR

在Yaf中集成Eloquent ORM需要通过Composer管理相关依赖，并完成Eloquent的初始化。

<!-- more -->

# Yaf

[Yaf](https://github.com/laruence/yaf) 是以性能优秀著称的通过PHP扩展实现的框架。

与常规的框架使用上不同，这一框架需要安装PHP扩展才可以使用。安装方法不再赘述。

Yaf 的优点自然是性能，抽象出了常规框架必备的加载、路由等功能，提供了插件机制，对一个请求的一些阶段可以进行hook，实现对输入输出的控制，可谓是极简框架。

但是极简框架带来的问题在于，一些便于代码编写的工具并不会被包含到框架之中，虽然与框架的设计目的相违背，但是对于使用者来说，也增加了一些成本。

# Eloquent

[Eloquent](http://laravel.com/docs/eloquent) 是Laravel框架操作数据库的利器，已被抽出成为独立组件[illuminate/database](https://github.com/illuminate/database)，对于基本的查询工作，省去了编写SQL的时间，同时提供自动修改时间戳，软删除等功能也让开发成本大大降低。`Seeder`与`Migration`工具更是开发过程的有力工具。`Seeder`结合`Faker`方便制造假数据供接口测试，`Migration`则帮助建立库表结构。

# Yaf集成Eloquent

使用Yaf作为PHP开发框架，又想要享受Eloquent便捷的数据库操作方式，唯一的办法自然是将二者集成。

示例代码位于[GitHub](https://github.com/liaoaoyang/YafWithEloquentSample)上。

## 开启Yaf的命名空间模式（非必须）

Yaf框架可以在`php.ini`中通过 `yaf.use_namespace = 1` 这一配置项开启命名空间模式。开启之后Yaf的一些内置类名称从下划线分隔变为命名空间的形式，如`Yaf_Controller_Abstract` 变为 `Yaf\Controller_Abstract`。

这一步骤是非必须的，因为不开启命名空间也可以实现后续的功能。

如果开发阶段不想在php.ini里修改yaf的配置，也可以通过php -d传入参数启动内置http服务器，如：`php -d 'yaf.use_namespace=1' -S 127.0.0.1:8080`。

## 通过yaf_cg工具生成初始化项目

Yaf文档中有[经典的项目结构](http://yaf.laruence.com/manual/tutorial.firstpage.html#tutorial.directory)：

```
+ public
  |- index.php //入口文件
  |- .htaccess //重写规则    
  |+ css
  |+ img
  |+ js
+ conf
  |- application.ini //配置文件   
+ application
  |+ controllers
     |- Index.php //默认控制器
  |+ views    
     |+ index   //控制器
        |- index.phtml //默认视图
  |+ modules //其他模块
  |+ library //本地类库
  |+ models  //model目录
  |+ plugins //插件目录
```

如果还想更简单，[Yaf Codes Generator](https://github.com/laruence/php-yaf/tree/master/tools/cg) 可以帮助生成一个类似这一结构的项目，也就是示例代码的目录结构。

如果开启了命名空间模式，需要对当中的一些代码进行改写。

## Composer添加依赖

Composer用作PHP项目依赖管理工具是最好的选择之一，通过`composer require illuminate/database v4.2.17`可以在`composer.json`中添加依赖，这里使用的是较老版本的illuminate/database。

执行`composer install`进行安装依赖。

## 自动加载配置

### Yaf

Yaf有自己的一套加载规则，在未做特殊配置的情况下，在默认模块下进行开发，简单来说可以概括成如下几个：

+ 代码的起始查找目录都在于在ini中定义的`application.directory`(此处值可以使用PHP代码中的预定义常量)
+ `_`表示目录分隔符，`class Foo_BarBar_Var`等同于目录`Foo/BarBar/Var.php`
+ `Controller`为结尾的类会在`controllers`目录下进行查找
+ `Model`为结尾的类会在`models`目录下进行查找
+ `Plugin`为结尾的类会在`plugins`目录下进行查找
+ 其他类会在`library`目录下进行查找

### Eloquent

Eloquent的加载规则就是标准的PSR-4加载规则。

每次更新依赖之后或者在`composer.json`中新增了映射规则之后，通过`composer dump-autoload`更新`vendor/autoload.php`文件即可。

### Yaf+Eloquent

Yaf的加载规则和PSR-4并不能完全匹配，所幸Yaf允许增加自动加载规则，通过`Yaf\Loader::import()`方法可以将`vendor/autoload.php`引入，针对不满足Yaf框架的加载规则的依赖包完成依赖加载工作。

## 初始化Eloquent

在上述步骤完整之后，可以定义一个基类，继承了这一个基类的model可以不用再重复处理连接初始化等问题。

我们使用的数据库是MySQL，对于MySQL需要配置用户名、密码等连接参数，为了方便起见，在ini文件中按照Eloquent需要的形式配置字段：

```
database.driver    = "mysql"
database.host      = "127.0.0.1"
database.port      = "3306"
database.database  = "learn"
database.username  = "learn"
database.password  = "123456"
database.charset   = "utf8"
database.collation = "utf8_general_ci"
database.prefix    = ""
```

Yaf读取这一配置之后，会转化为一个数组，key为`database.`之后的字符串，值即等号之后的值。

按照Eloquent的手册，赋值后完成初始化操作：

```
<?php

use Illuminate\Database\Capsule\Manager as IlluminateCapsule;
use Illuminate\Database\Eloquent\Model as IlluminateModel;
use Yaf\Registry as YRegistry;


class BaseModel extends IlluminateModel
{
    protected $config = null;
    protected $capsule = null;

    public function __construct(array $attributes = array())
    {
        parent::__construct($attributes);
        $dbConfigKey = DATABASE_CONFIG_KEY;
        $this->config = YRegistry::get('config');

        if (!$this->config->$dbConfigKey) {
            throw new Exception("Must configure database in .ini first");
        }

        $this->config = $this->config->$dbConfigKey->toArray();
        $this->capsule = new IlluminateCapsule();
        $this->capsule->addConnection($this->config);
        $this->capsule->bootEloquent();
    }
}
```

通过`Yaf\Registry::get()`方法是因为在`Bootstrap.php`的`_initConfig()`方法中已经对配置做了存储：

```
public function _initConfig()
{
   //把配置保存起来
   $arrConfig = Yaf\Application::app()->getConfig();
   Yaf\Registry::set('config', $arrConfig);
}
```

`2018-05-09更新`：针对于需要使用事务以及门面的情况，此处进行了[描述](/articles/2018/05/03/integrate-yaf-with-eloquent-ii/)。

## 实现Model

将要操作的表结构为：

```
CREATE TABLE `db_sample` (
    `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
    `name` varchar(100) NOT NULL DEFAULT '',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

继承BaseModel，设置表、是否使用时间戳等属性，实现Model：

```
class DBSampleModel extends \BaseModel
{
    protected $table = 'db_sample';
    public $timestamps = false;
}
```

至此，Yaf之中已经可以通过Eloquent操作MySQL数据库。


