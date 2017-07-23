title: CodeIgniter3
date: 2017-07-22 22:31:01
tags: [PHP, CodeIgniter]
categories: PHP
---

# TL;DR

CodeIgniter3是一个相当轻量、简便的并且上手难度低的PHP应用开发框架。在CodeIgnitor2时代曾经接触并开发了一些项目。目前最新版本是`3.1.5`。优点个人认为有：

+ 轻量
+ 对MySQL查询有较为友好的代码编写方式
+ 功能扩展较为简便
+ 可以支持较低版本的PHP（5.3.7+）
+ 可以不使用模板引擎

同时，个人也认为以下功能还可以有所变化：

+ 内置日志功能不够强大
+ 内置读写分离
+ RESTful API开发支持

文章的剩余内容将会针对以上的各个方面详细说明。

<!-- more -->

# 优点

## 轻量

### 文件数量

首先通过cloc统计以下代码数量：

```
→ cloc .
     441 text files.
     395 unique files.
      57 files ignored.

github.com/AlDanial/cloc v 1.72 T=4.33 s (88.9 files/s, 45382.0 lines/s)
---------------------------------------------------------------------
Language           files          blank        comment           code
---------------------------------------------------------------------
HTML                 168          16847            167         107992
PHP                  198           8363          29905          29982
JavaScript             8            154            306           1420
XML                    4              0              0            552
CSS                    5            128             31            536
Markdown               1             38              0             57
JSON                   1              0              0             23
---------------------------------------------------------------------
SUM:                 385          25530          30409         140562
---------------------------------------------------------------------

```

仅仅198个php文件，对于一个拥有完整的数据库查询支持，表单验证，Session支持，XSS/CSRF防护的框架来说，并不算臃肿。

文件数量一定程度上反映了层次划分的粒度，如果文件数量过多，带来的首先是源码阅读的难度和改造的难度。

### 基本业务功能实现

对于一个号称轻量的框架来说，实现一个数据库的CRUD操作的Hello World的代码量应该最能反映问题。

CI3实现这样的一个Controller，需要多少代码量呢？答案因人而异。但是步骤上却是相当简单的。考虑如下数据表：

```
CREATE TABLE `ci3_test1` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

解压缩CI3之后，在`application`中的`config`目录下的`database.php`中配置好数据库配置：

```
$db['default'] = array(
	'dsn'	=> '',
	'hostname' => '127.0.0.1',
	'username' => 'test',
	'password' => 'passwd',
	'database' => 'ci3',
	'dbdriver' => 'mysqli',
	'dbprefix' => '',
	'pconnect' => FALSE,
	'db_debug' => (ENVIRONMENT !== 'production'),
	'cache_on' => FALSE,
	'cachedir' => '',
	'char_set' => 'utf8',
	'dbcollat' => 'utf8_general_ci',
	'swap_pre' => '',
	'encrypt' => FALSE,
	'compress' => FALSE,
	'stricton' => FALSE,
	'failover' => array(),
	'save_queries' => TRUE
);
```

在`application`中的`config`目录下的`routes.php`中配置好路由规则：

```
$route['test1/(\S+)']['POST']      = '/test1/create/$1';
$route['test1/(\d+)']['GET']       = '/test1/retrieve/$1';
$route['test1/(\d+)/(\S+)']['PUT'] = '/test1/update/$1/$2';
$route['test1/(\d+)']['DELETE']    = '/test1/delete/$1';
```

只需要编写少许几行代码即可完成上述操作。

```
class Test1 extends CI_Controller {
    const TABLE = 'ci3_test1';
    /**
     * PHPStorm IDE hint
     * @var CI_DB_mysqli_driver $db
     */
    public $db = null;

    public function __construct()
    {
        parent::__construct();
        $this->load->database();
    }

    public function create($name)
    {
        $this->db->insert(self::TABLE, ['name' => $name]);
        $insertedId = $this->db->insert_id();

        if ($insertedId) {
            echo "Inserted {$insertedId} with {$name}"; exit();
        }

        echo "Failed to insert with {$name}";
    }

    public function retrieve($id)
    {
        $result = $this->db->get_where(self::TABLE, ['id' => $id])->result_array();

        if (count($result) == 0) {
            echo "No such record"; exit();
        }

        echo "Retrieved id={$result[0]['id']} name={$result[0]['name']}";
    }

    public function update($id, $name)
    {
        $this->db->update(self::TABLE, ['name' => $name], ['id' => $id]);

        if ($this->db->affected_rows() > 0) {
            echo "Updated record[{$id}] with name={$name}"; exit();
        }

        echo "Failed to update record[{$id}]";
    }

    public function delete($id)
    {
        $this->db->delete(self::TABLE, ['id' => $id]);

        if ($this->db->affected_rows() == 0) {
            echo "Failed to delete record[$id]"; exit();
        }

        echo "Deleted record[$id]";
    }
}
```

如果在Model层做一些封装，相信对于这类基本的CRUD操作的编写会更加的便捷。

## 友好的MySQL查询代码编写方式

日常开发中，个人更倾向于使用纯SQL进行数据库的查询工作，但是纯SQL的可读性不一定比精心设计的查询工具类更好，同时实用查询记录构造工具，可以规避一些错误的SQL编写方式（如不转义用户输入，直接拼接参数到SQL语句之中）。

如手册中的例子：

```
$this->db->select('*')->from('my_table')
    ->group_start()
        ->where('a', 'a')
        ->or_group_start()
            ->where('b', 'b')
            ->where('c', 'c')
        ->group_end()
    ->group_end()
    ->where('d', 'd')
->get();

// Generates:
// SELECT * FROM (`my_table`) WHERE ( `a` = 'a' OR ( `b` = 'b' AND `c` = 'c' ) ) AND `d` = 'd'
```


CI3的查询构造器在调用`get()`方法前可以声明各个查询条件的组合。通过链式方法，让代码拥有的了一定的可读性。

### CI3数据库查询过程简述（PDO）

虽然在上述例子中使用的是`mysqli`作为示例，但是PDO在工作中更为常用，简要的描述一下CI3使用PDO作为driver进行一次查询过程的流程。

1. CI3中，在`database.php`中配置了数据库之后，通过`load`对象中的`database()`方法加载数据库对象。
2. `database()`方法实际上会将数据库配置传入`DB()`这一方法，如果传入参数为空，则会查找应用目录下的`database.php`配置文件并引入。如果配置了DSN或者配置组的名字，则会解析或者查找配置文件中是否存在对应的配置组。所以，假设要编写一个新的DB Driver，需要一些特殊的配置，可以在项目的`database.php`配置中增加一个特定的配置组，增加自己独有的字段。
3. 根据配置的`dbdriver`字段，到框架目录下查找是否存在对应的Driver，之后实例化并连接。由于PDO可以支持多种数据库，所以对于PDO来说，还会根据DSN查找对应的数据库的Driver类型（即在`subdrivers`目录下找到对应的类，mysql则是`CI_DB_pdo_mysql_driver`）完成实例化。
4. 通过`CI_DB_query_builder`的各个方法（`select()`/`where()`）构造出SQL语句之后，通过类中的`get()/insert()/update()/replace()/delete()`等方法对数据库进行操作，实际上都是交由`query()`方法对对应的SQL语句进行执行。
5. `query()`方法会通过`simple_query()`这一个方法执行SQL语句，`simple_query()`这一方法首先会检测是否已经初始化数据库连接对象，即成员变量`conn_id`，如果没有则通过`initialize()`方法进行连接。初始化完成后，执行上就需要各个DB Driver实现一个`_execute`方法了。
6. PDO的driver实现的`_execute`方法则是直接调用内部成员属性`conn_id`的`query()`方法，即我们熟知的PHP MySQL PDO的`query()`方法，这一方法会返回一个 [PDOStatement](http://php.net/manual/zh/class.pdostatement.php) 对象。SQL语句的执行结果会存储到`result_id`成员变量之中。
7. 针对不同的数据库，会对加载不同的结果集Driver，即我们调用`get()/get_where()`等等操作之后返回的对象，我们通过这一个Driver对象，调用`result()/result_array()/row()/unbufferred_row()`等方法得到期望查询的结果。

多说一点，`result()/result_array()/result_object()/custom_result_object()`等方法会把所有匹配的记录都读出，存储到当前结果对象的一个成员变量数组之中，在大结果集的情况下，可能会带来内存的大量使用，这里也是手册中提及过的，同样的使用`row()/row_array()/row_object()/first_row()/last_row()/next_row()/previous_row()`等方法也会调用上述方法之一，所以调用了这些方法，实际上也会触发大量的内存使用的情况。而`unbufferred_row()`这个方法实际上是通过逐个获取`PDOStatement`对象的每一个结果数据，在内存占用上会有较好的表现。编写实例测试：

数据库中存在2条记录。

***Controller:***

```
class Test1 extends CI_Controller {
    const TABLE = 'ci3_test1';
    /**
     * PHPStorm IDE hint
     * @var CI_DB_mysqli_driver $db
     */
    public $db = null;

    public function __construct()
    {
        parent::__construct();
        $this->load->database();
    }

    public function fetch_one()
    {
        $result = $this->db->get(self::TABLE);
        while ($result->unbuffered_row()) {
            printf("%s\n", __FUNCTION__ . ':' . memory_get_usage());
        }
        printf("%s\n", __FUNCTION__ . ':' . memory_get_usage());

    }

    public function fetch_all()
    {
        $result = $this->db->get(self::TABLE);
        $result->result();
        printf("%s\n", __FUNCTION__ . ':' . memory_get_usage());
    }
}
```

***routes.php***

```
$route['test1/fetch_one']['GET']   = '/test1/fetch_one';
$route['test1/fetch_all']['GET']   = '/test1/fetch_all';
```

***/test1/fetch_one***

```
fetch_one:2104664
fetch_one:2104920
fetch_one:2104920
```

***/test1/fetch_all***

```
fetch_all:2106256
```

显然`fetch_all`操作占用的内存***2106256 > 2104920***。




