title: Yaf集成Eloquent——使用事务以及DB Facade
date: 2018-05-03 00:41:50
tags: [PHP, Yaf, Eloquent, Laravel]
categories: PHP
---

# TL;DR

集成方法参见 [Yaf集成Eloquent](/articles/2017/08/01/integrate-yaf-with-eloquent/) 。

集成基类的多个 Model 如果要正确的运行事务，需要保证各个 Model 的实例使用的是同一个数据库连接，在代码上可以通过共用同一个  `Illuminate\Database\Capsule\Manager` 对象实现。

使用 DB Facade 需要为 Facade 提供已经关联了 `db` 作为键，以 `Illuminate\Database\Capsule\Manager` 的实例为值的容器。

<!-- integrate-yaf-with-eloquent-ii -->
<!-- more -->

# 实验环境

+ MySQL 5.6
+ PHP 5.6.31
+ Yaf 2.3.3
+ Eloquent 4.2.17（5.0亦可）


# 事务

## Laravel 中的数据库事务

Laravel 中的数据库事务可以通过 [DB Facade](https://laravel.com/docs/5.0/database#database-transactions) 开启：

```
DB::transaction(function()
{
    DB::table('users')->update(['votes' => 1]);

    DB::table('posts')->delete();
});
```

即通过通过 DB Facade 访问了服务容器内的各个 Model 发起了事务。

## MySQL 事务

MySQL 中执行事务有两大前提条件：

+ 同一个数据库
+ 同一个数据库连接

如果使用 PDO 连接 MySQL 发起事务，那么 PDO 对象应该只有一个。

## 原有样例存在的问题

部分单独使用 Eloquent 的样例，包括自己早期的尝试在内，都存在一个类似的情况：

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

这里在构造函数中会构建一个 `Illuminate\Database\Capsule\Manager $capsue` 对象，这一对象是 Eloquent 的关键部分之一，实现了连接管理等功能：

```
<?php namespace Illuminate\Database\Capsule;

use PDO;
use Illuminate\Events\Dispatcher;
use Illuminate\Cache\CacheManager;
use Illuminate\Container\Container;
use Illuminate\Database\DatabaseManager;
use Illuminate\Database\Eloquent\Model as Eloquent;
use Illuminate\Database\Connectors\ConnectionFactory;
use Illuminate\Support\Traits\CapsuleManagerTrait;

class Manager {

	use CapsuleManagerTrait;

	/**
	 * The database manager instance.
	 *
	 * @var \Illuminate\Database\DatabaseManager
	 */
	protected $manager;
```

通过内置的 `protected \Illuminate\Database\DatabaseManager $manager` 对象，建立真正的连接。

Manager 会根据声明的连接配置名称创建对应的连接，通过连接配置名称可以直接获取对应的连接对象，注入到实际执行查询操作的 `Illuminate\Database\Query\Builder` 中完成查询。

连接是按序连接，在需要进行查询时，`\Illuminate\Database\DatabaseManager` 会调用 ` \Illuminate\Database\Connectors\ConnectionFactory` 成员的 `make()` 方法创建连接，在这一方法中完成连接对象的创建和连接操作。

```
<?php namespace Illuminate\Database\Connectors;

use PDO;
use Illuminate\Container\Container;
use Illuminate\Database\MySqlConnection;
use Illuminate\Database\SQLiteConnection;
use Illuminate\Database\PostgresConnection;
use Illuminate\Database\SqlServerConnection;

class ConnectionFactory {

	/**
	 * The IoC container instance.
	 *
	 * @var \Illuminate\Container\Container
	 */
	protected $container;

	/**
	 * Create a new connection factory instance.
	 *
	 * @param  \Illuminate\Container\Container  $container
	 * @return void
	 */
	public function __construct(Container $container)
	{
		$this->container = $container;
	}

	/**
	 * Establish a PDO connection based on the configuration.
	 *
	 * @param  array   $config
	 * @param  string  $name
	 * @return \Illuminate\Database\Connection
	 */
	public function make(array $config, $name = null)
	{
		$config = $this->parseConfig($config, $name);

		if (isset($config['read']))
		{
			return $this->createReadWriteConnection($config);
		}

		return $this->createSingleConnection($config);
	}

	/**
	 * Create a single database connection instance.
	 *
	 * @param  array  $config
	 * @return \Illuminate\Database\Connection
	 */
	protected function createSingleConnection(array $config)
	{
		$pdo = $this->createConnector($config)->connect($config);

		return $this->createConnection($config['driver'], $pdo, $config['database'], $config['prefix'], $config);
	}

```

那么问题来了，在多个 Model 实现类进行事务操作时，对于每一个子类来说，基类 `BaseModel` 中都会建立一个全新的 `DatabaseManager` 对象，实际上对于每个 Model 来说都建立了不同的数据库连接，不同连接下执行事务是不可能保证 ACID 的。

## 解决

### Laravel 中的处理

来看原生支持 Eloquent 的 Laravel 是如何处理的。

阅读 `Illuminate\Database\DatabaseServiceProvider` 的源码：

```
/**
 * Register the primary database bindings.
 *
 * @return void
 */
protected function registerConnectionServices()
{
    // The connection factory is used to create the actual connection instances on
    // the database. We will inject the factory into the manager so that it may
    // make the connections while they are actually needed and not of before.
    $this->app->singleton('db.factory', function ($app) {
        return new ConnectionFactory($app);
    });

    // The database manager is used to resolve various connections, since multiple
    // connections might be managed. It also implements the connection resolver
    // interface which may be used by other components requiring connections.
    $this->app->singleton('db', function ($app) {
        return new DatabaseManager($app, $app['db.factory']);
    });

    $this->app->bind('db.connection', function ($app) {
        return $app['db']->connection();
    });
}
```

Laravel的处理很简单，将 `\Illuminate\Database\DatabaseManager` 处理成单例。

### 本文解决方案

[样例](https://github.com/liaoaoyang/YafWithEloquentSample/blob/master/application/models/Base.php) 中使用了同样的方式，即将此对象实现为单例模式。

# DB Facade

通过 Facade 可以方便的访问数据库相关功能。

Eloquent 中 `Illuminate\Support\Facades\DB` 即是入口。

## Laravel 中的 DB Facade

Laravel 中的 Facade 实际上是通过访问已关联的应用程序对象中已关联的对象，通过对象实现的 __call 以及 __callStatic 魔术方法，实现功能。

摘录部分 DB Facade 代码：

```
protected static function getFacadeAccessor()
{
    return 'db';
}

/**
 * Resolve the facade root instance from the container.
 *
 * @param  string|object  $name
 * @return mixed
 */
protected static function resolveFacadeInstance($name)
{
    if (is_object($name)) {
        return $name;
    }

    if (isset(static::$resolvedInstance[$name])) {
        return static::$resolvedInstance[$name];
    }

    return static::$resolvedInstance[$name] = static::$app[$name];
}

/**
 * Get the root object behind the facade.
 *
 * @return mixed
 */
public static function getFacadeRoot()
{
    return static::resolveFacadeInstance(static::getFacadeAccessor());
}

/**
 * Handle dynamic, static calls to the object.
 *
 * @param  string  $method
 * @param  array   $args
 * @return mixed
 *
 * @throws \RuntimeException
 */
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    return $instance->$method(...$args);
}
```

可以看到，DB Facade 访问的是 `$app['db']` 对应的对象。

回到之前 `Illuminate\Database\DatabaseServiceProvider` 的 `registerConnectionServices` 方法：

```
$this->app->singleton('db', function ($app) {
    return new DatabaseManager($app, $app['db.factory']);
});
```

实际上 `$app['db']` 关联的是一个 `DatabaseManager` 对象。

## 文中的 DB Facade

按照这一思路，将 `Illuminate\Support\Facades\DB` 的 $app 对象增加一个 key db，并关联上当前存在的 `DatabaseManager` 对象即可：

```
self::$capsule = new IlluminateCapsule();
self::$capsule->bootEloquent();
Illuminate\Support\Facades\DB::setFacadeApplication([
    'db' => self::$capsule->getDatabaseManager(),
]);
```

