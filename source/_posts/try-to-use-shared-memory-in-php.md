title: PHP 使用共享内存
date: 2017-01-04 00:55:40
tags: [PHP]
---

# 概述

[共享内存](https://zh.wikipedia.org/wiki/%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98)指在多处理器的计算机系统中，可以被不同中CPU访问的大容量内存。是众多进程间通信（IPC）方式中最快的一种，无需进行数据拷贝等操作，即可在各个进程之间共享数据。

PHP同样也提供了对共享内存操作的可能性，通过`shmop`扩展实现。

对于多进程写入的并发控制，可以尝试通过`sysvsem`扩展进行控制。

<!-- more -->

# 应用

结合一个实际使用的场景，PHP daemon多进程从上游获取用户ID，需要与指定数据集去重，用户ID大约为12位的数字。要求是不能多去重（即不能存在误伤的判断），但是也不能少去重。

考虑到多进程处理时，需要考虑如何实现便捷，快速的进行判断用户ID已存在于数据集之中。在数据结构上可以使用的几种方法有：

+ Bitmap
+ Hash
+ Array

下面分别说明三者的优缺点：

## Bitmap

Bitmap（位图，以下不加区分的使用）的特点在于数据密度大时（即已排序的情况下，相邻数字间隔不大），是极为节省内存用量的数据结构。无论数字多大，在内存中只通过`1 Bit`进行表示。假设用`1 Byte`的空间进行数字的表示，数据集为[1,3,4,5,7]，则这一组数据的位图可以表示为：

```
01011101
```

即便数字相当巨大，在已知最小值的情况下，同样通过同样大小的空间也可表示，如数据集为[100000001,100000003,100000004,100000005,100000007]，同样也可以用同样一个位图进行表示，因为所有数字的都可以认为相对于`100000000`进行了偏移操作。

Bitmap操作起来速度与便利性也相当令人满意，只需要根据数字大小，找到对应的位，判断当前位的0/1值即可，位运算操作的速度之快无需多言。

然而，Bitmap的最大问题在于，如果存储的是数值的顺序信息，那么整个Bitmap的数据才是最有效的。即，如果已经数字本身是有序的，如从1开始，一直到10000，或者是有办法迅速的知道位置是1的数字的具体字面值，那么存储的位会更加的高效。

在存储用户ID这一个场景时，Bitmap不一定适用，因为用户ID可能会长于10位，如果把用户ID当成数字来看，同时考虑到可能存在的一些业务形态（6，8之类的靓号逻辑，4之类的避讳逻辑），可能得到的Bitmap就相当的“稀疏”（0过多，1过少），造成的结果就是内存的有效使用率降低。假设用户ID最大值是9999999999，仅仅在10位数的用户ID的情况下，为了包含所有的数字，需要开辟约**1192.09MB**大小的内存空间，当然实际应用中很可能会比这个要小，通过找到偏移值（获取比较集的最大最小值，确定偏移值，减少无用0的内存占用）、数据分块（比如前5位相同的比率大，把数字前5位作为一个集合，只生成后5位的Bitmap提高表示有效程度）等方式，但是又会引入一些其他的问题，或者效果不佳（比较集合中存在1和9999999999两个用户ID）或者是难于管理（前5位有上万种组合
）。

如果在可以接受误伤的情况下，有一个更优的方案，即布隆过滤器（Bloom filter），这是通过Bitmap可以完成的，综合速度和存储压力都较优方案。

## Hash

Hash（哈希表）的特点在于快，理论上来说，Hash的查询和写入时间复杂度都是***O(1)***。

编程语言如果提供了Hash的数据结构，那么应用数据结构就成为了一个看起来不错选择了。

实际实现中，Hash存在的第一个可能存在的问题是Hash冲突的解决，如果用[开放地址法（Open addressing）](https://en.wikipedia.org/wiki/Hash_table#Open_addressing)进行解决，可能的问题在于冲突之后查询下一个可用地址的次数过多，而用[链表(Separate chaining)](https://en.wikipedia.org/wiki/Hash_table#Separate_chaining)解决，则会存在退化的情况（所有值都hash到一个链表中）。不过这些问题一般来说都应该是在应用过程中由编程语言关心的。

这次应用限定了编程语言为PHP，那么从PHP的实现上来看看应用Hash是否可行。

PHP里的数组就提供了Hash的功能，PHP数组的便利程度无需多言，在去重这一个场景上，完全可以通过用户ID作为key，写入布尔值作为value，通过`isset()`方法快速的进行去重操作。

速度上我们可能不再担忧了，但是内存占用上呢？

目前PHP的最新版本为`PHP 7.1.0`，常规编译安装后，通过如下脚本获取0~1000000在数组中的内存占用情况：

```
<?php
	ini_set('memory_limit', '512M');

	$repeat = isset($argv[1]) ? $argv[1] : 0;

	echo 'memory usage(B):' . memory_get_usage() . "\n";
	$a = [];

	for($i = 0; $i < $repeat; $i++) {
		$a[sprintf("%07d", $i)] = '1';
	}

	echo 'memory usage(B):' . memory_get_usage() . "\n";
```

运行结果为：

```
$ /usr/local/php7/7.1.0/bin/php array_mem_usage.php 1000000
memory usage(B):350688
memory usage(B):358099536
```

仅仅1000000的7位字符串作为hash key的数据集，就需要耗费超过**341MB**的内存。可是即便是作为文本文件，这些数据集在通过换行符号分隔的情况下，完全加载到内存只需要大约**8MB**的内存使用。是什么造成了如此大的差距呢？

从PHP的源码来看(源码目录下的`Zend/zend_types.h`)，PHP数组通过`HashTable`这一个struct实现：

```
typedef struct _zend_array HashTable;

struct _zend_array {
    zend_refcounted_h gc;
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    flags,
                zend_uchar    nApplyCount,
                zend_uchar    nIteratorsCount,
                zend_uchar    consistency)
        } v;
        uint32_t flags;
    } u;
    uint32_t          nTableMask;
    Bucket           *arData;
    uint32_t          nNumUsed;
    uint32_t          nNumOfElements;
    uint32_t          nTableSize;
    uint32_t          nInternalPointer;
    zend_long         nNextFreeElement;
    dtor_func_t       pDestructor;
};
```

数据的实际存储部分即`Bucket`，对内存占用影响起到决定性作用的也正是`Bucket`这个数据结构，Bucket的定义为：

```
typedef struct _Bucket {
    zval              val;
    zend_ulong        h;                /* hash value (or numeric index)   */
    zend_string      *key;              /* string key or NULL for numerics */
} Bucket;
```

在`Bucket`中包含`zval`，`zend_ulong`，以及`zend_string`三种数据结构，下面分别来看看这几个数据结构。

### zval

`zval`的定义如下：

```
typedef struct _zval_struct     zval;

struct _zval_struct {
    zend_value        value;            /* value */
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    type,         /* active type */
                zend_uchar    type_flags,
                zend_uchar    const_flags,
                zend_uchar    reserved)     /* call info for EX(This) */
        } v;
        uint32_t type_info;
    } u1;
    union {
        uint32_t     next;                 /* hash collision chain */
        uint32_t     cache_slot;           /* literal cache slot */
        uint32_t     lineno;               /* line number (for ast nodes) */
        uint32_t     num_args;             /* arguments number for EX(This) */
        uint32_t     fe_pos;               /* foreach position */
        uint32_t     fe_iter_idx;          /* foreach iterator index */
        uint32_t     access_flags;         /* class constant access flags */
        uint32_t     property_guard;       /* single property guard */
    } u2;
};
```

`zval`中又包含`zend_vlaue`，那么通过`zend_value`的定义：

```
typedef union _zend_value {
    zend_long         lval;             /* long value */
    double            dval;             /* double value */
    zend_refcounted  *counted;
    zend_string      *str;
    zend_array       *arr;
    zend_object      *obj;
    zend_resource    *res;
    zend_reference   *ref;
    zend_ast_ref     *ast;
    zval             *zv;
    void             *ptr;
    zend_class_entry *ce;
    zend_function    *func;
    struct {
        uint32_t w1;
        uint32_t w2;
    } ww;
} zend_value;
```

可以看出`zend_value`作为一个union至少要占用8个字节（最大的内存占用来自于当中包含的`zend_long`，在`Zend/zend_long.h`中被定义为`typedef int64_t zend_long;`），所以，作为一个struct，`zval`会占用sizeof(value)+sizeof(u1)+sizeof(u2)=8+4+4=16 Bytes。

### zend_ulong

在`Zend/zend_long.h`中被定义为`typedef int64_t zend_ulong;`，所以会占用8 Bytes。

### zend_string

来看`zend_string`的定义：

```
struct _zend_string {
    zend_refcounted_h gc;
    zend_ulong        h;                /* hash value */
    size_t            len;
    char              val[1];
};
```

其中`zend_refcounted_h`的定义为：

```
typedef struct _zend_refcounted_h {
    uint32_t         refcount;          /* reference counter 32-bit */
    union {
        struct {
            ZEND_ENDIAN_LOHI_3(
                zend_uchar    type,
                zend_uchar    flags,    /* used for strings & objects */
                uint16_t      gc_info)  /* keeps GC root number (or 0) and color */
        } v;
        uint32_t type_info;
    } u;
} zend_refcounted_h;
```

可以看到`zend_refcounted_h`的空间占用为sizeof(refcount)+sizeof(u)=4+4=8 Bytes。

`size_t`这里需要注意，因为在64位OS上编译，这里会占用8 Bytes。

结构成员val用于存储key的值，由于key都是7位长的字符串，所以这里会占用7 Bytes。

所以，在这里，空间占用为8+8+8+7=31 Bytes。

综上，直接统计数据结构的大小已经能明显看到PHP提供Hash是相当占用内存的，出于这个方面的考虑，基本可以认定Hash不适合当前的应用场景。

## Array

可能有人会说，Array算是什么办法，但是对于特定情况，Array确实可以使用。

比如C语言的数组，是在内存中的一片连续空间，也就是说，实际上内存的占用就是数据个数乘以单个数据需要占用的空间。从这个角度来看，是没有太多的内存浪费的。

当然PHP的数组不能这么看，因为PHP的数组仍然有太多的冗余信息。

数组的“缺点”在于查找的耗时。线性查找的耗时几乎让人无法接受，那么如果数据可以是有序的，通过二分查找耗时将会大大降低。

1M的数据，只需要通过至多10次查找即可判断对应的值是否存在。

## Bitmap/Hash/Array?

从数据特点和存储用量来考虑，同时考虑到查询速度，Array在这个场景下胜出。

## 存储

PHP可以使用共享内存，作为最简单的IPC方式，并且使用方式相当简单，PHP的`shmop`扩展中提供了对共享内存的操作能力。

### 创建/打开

共享内存在PHP的创建和打开工作是通过`shmop_open`方法实现的。定义如下：

```
int shmop_open ( int $key , string $flags , int $mode , int $size )
```

PHP的共享内存的创建实际上是通过`shmget`这一系统调用实现的，参见`shmop`扩展源码：

```
PHP_FUNCTION(shmop_open)
{
	zend_long key, mode, size;
	struct php_shmop *shmop;
	struct shmid_ds shm;
	char *flags;
	size_t flags_len;

	if (zend_parse_parameters(ZEND_NUM_ARGS(), "lsll", &key, &flags, &flags_len, &mode, &size) == FAILURE) {
		return;
	}

// 其他代码

	if (shmop->shmflg & IPC_CREAT && shmop->size < 1) {
		php_error_docref(NULL, E_WARNING, "Shared memory segment size must be greater than zero");
		goto err;
	}

	shmop->shmid = shmget(shmop->key, shmop->size, shmop->shmflg); // 创建共享内存
	if (shmop->shmid == -1) {
		php_error_docref(NULL, E_WARNING, "unable to attach or create shared memory segment '%s'", strerror(errno));
		goto err;
	}

// 其他代码

	shmop->addr = shmat(shmop->shmid, 0, shmop->shmatflg); // 申请or打开的共享内存映射到当前进程的虚拟内存空间
	if (shmop->addr == (char*) -1) {
		php_error_docref(NULL, E_WARNING, "unable to attach to shared memory segment '%s'", strerror(errno));
		goto err;
	}

	RETURN_RES(zend_register_resource(shmop, shm_type));
err:
	efree(shmop);
	RETURN_FALSE;
}
```

`shmget`返回的值是一个类似文件描述符的存在，因为它并不是一个真正的文件描述符，所以我们实际上的操作依据仅仅上是一个全局唯一的数字，用来表示共享内存。同时通过`shmat`系统调用将共享内存映射到当前进程的地址虚拟空间之中。

共享内存打开方法中的第一个参数指定的key，是标识这一共享内存片段的依据，需要全局唯一，一个方法就是通过一个确实存在的文件，利用`ftok`系统调用，生成一个全局唯一的key。文件名不是决定key值的决定因素，决定因素是文件的inode号。

### 关闭

```
void shmop_close ( resource $shmid )
```

关闭的实现的是通过`shmdt`系统调用完成的。在扩展的`MINIT`阶段，通过`zend_register_list_destructors_ex`方法注册了资源析构方法为`rsclean`，对共享内存的的ID看做是资源（Resource）。

```
PHP_MINIT_FUNCTION(shmop)
{
	shm_type = zend_register_list_destructors_ex(rsclean, NULL, "shmop", module_number);

	return SUCCESS;
}
```

在调用这一方法时，通过资源删除API`zend_list_close`调用注册的`rsclean`方法完成对资源的释放。

```
PHP_FUNCTION(shmop_close)
{
	zval *shmid;
	struct php_shmop *shmop;

	if (zend_parse_parameters(ZEND_NUM_ARGS(), "r", &shmid) == FAILURE) {
		return;
	}


	if ((shmop = (struct php_shmop *)zend_fetch_resource(Z_RES_P(shmid), "shmop", shm_type)) == NULL) {
		RETURN_FALSE;
	}

	zend_list_close(Z_RES_P(shmid));
}
```

而`rsclean`的实现如下：

```
static void rsclean(zend_resource *rsrc)
{
	struct php_shmop *shmop = (struct php_shmop *)rsrc->ptr;

	shmdt(shmop->addr);
	efree(shmop);
}
```

对共享内存ID进行了`shmdt`操作，同时释放了的申请共享内存操作结构体。

### 删除

```
bool shmop_delete ( resource $shmid )
```

这一个方法中，是`shmctl`系统调用的表现的时候了。删除一段共享内存只需要将系统调用中的第二个参数设定为`IPC_RMID`即可。

```
PHP_FUNCTION(shmop_delete)
{
	zval *shmid;
	struct php_shmop *shmop;

	if (zend_parse_parameters(ZEND_NUM_ARGS(), "r", &shmid) == FAILURE) {
		return;
	}

	if ((shmop = (struct php_shmop *)zend_fetch_resource(Z_RES_P(shmid), "shmop", shm_type)) == NULL) {
		RETURN_FALSE;
	}

	if (shmctl(shmop->shmid, IPC_RMID, NULL)) {
		php_error_docref(NULL, E_WARNING, "can't mark segment for deletion (are you the owner?)");
		RETURN_FALSE;
	}

	RETURN_TRUE;
}
```

当然，删除共享内存也可以通过Linux中的ipcrm命令完成，如`ipcrm`的man中提到的，如果知到key（即`创建/打开`部分提到的共享内存全局唯一的标识）则使用`-M`参数，知道ID则使用`-m`参数。

```
     -M shmkey
             Mark the shared memory segment associated with key shmkey for removal.  This marked segment will be destroyed after the last detach.

     -m shmid
             Mark the shared memory segment associated with id shmid for removal.  This marked segment will be destroyed after the last detach.
```

### 读

```
string shmop_read ( resource $shmid , int $start , int $count )
```

读取的方法则是通过共享内存ID以及开始位置以及读取的长度获得一个字符串。

从实现上来看，以下几种情况会返回false并打印WARNING日志：

+ 起始值小于0或者大于共享内存的容量
+ 读取的字节数小于0
+ 起始值大于INT_MAX与读取字节数只差
+ 起始值与读取字节数之和大于共享内存大小

### 写

```
int shmop_write ( resource $shmid , string $data , int $offset )
```

实现上来说，实际上是通过`memcpy`将字符串的值复制到共享内存的指定位置。

#### 多进程写入

考虑到多进程环境下同时写入带来的问题，`sysvsem`扩展提供的[信号量](http://php.net/manual/zh/book.sem.php)可以为我们解决这一问题。

在需要多进程写入的地方，加入数目为1的信号量即可控制当前的共享内存只能有一个程序进行读写。

信号量可以选择非阻塞的方式，即在`sem_acquire`时设置第二个参数为`true`即可。

信号量在进程结束后默认情况下会自动释放，不一定需要通过`sem_release`显式释放。

来看样例程序：

```
<?php
	$sem_count = isset($argv[1]) ? $argv[1] : 1;
	$sleep_s = isset($argv[2]) ? $argv[2] : 1;
	$nonblocking = isset($argv[3]) ? true : false;

	if (pcntl_fork() == 0) {
		$key = ftok(__FILE__, 'r');
		$sem = sem_get($key, $sem_count);
		$pid = posix_getpid();

		echo $pid . "|" . time() . "|TRY GET SEM\n";

		if (sem_acquire($sem, $nonblocking)) {
			echo $pid . "|" . time() . "|GOT SEM\n";
			sleep($sleep_s);
		} else {
			echo $pid . "|" . time() . "|NO SEM\n";
		}
	}
```

同时启动5个进程：

```
→ for i in `seq 1 5`; do php test_sysvsem.php 1 1  ; done
41214|1508595622|TRY GET SEM
41214|1508595622|GOT SEM
41216|1508595622|TRY GET SEM
41218|1508595622|TRY GET SEM
41220|1508595622|TRY GET SEM
41222|1508595622|TRY GET SEM
41216|1508595623|GOT SEM
41218|1508595624|GOT SEM
41220|1508595625|GOT SEM
41222|1508595626|GOT SEM
```

选择非阻塞模式：

```
→ for i in `seq 1 5`; do php test_sysvsem.php 1 1 1 ; done
41432|1508596102|TRY GET SEM
41432|1508596102|GOT SEM
41434|1508596102|TRY GET SEM
41434|1508596102|NO SEM
41436|1508596102|TRY GET SEM
41436|1508596102|NO SEM
41438|1508596102|TRY GET SEM
41438|1508596102|NO SEM
41440|1508596102|TRY GET SEM
41440|1508596102|NO SEM
```


# 参考

+ [Linux进程间通信-共享内存](http://liwei.life/2016/08/08/share_memory/)
+ [PHP7 使用资源包裹第三方扩展的实现及其源码解读](https://mengkang.net/684.html)


