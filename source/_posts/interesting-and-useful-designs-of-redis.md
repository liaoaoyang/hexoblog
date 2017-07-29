title: Redis里那些有用又有趣的设计
date: 2017-07-29 11:34:44
tags: [Redis]
categories: 系统
---

# TL;DR

[Redis](https://redis.io/) 单进程单线程的运行模式，保证了操作的原子性。丰富的数据结构（LIST/HSET/ZSET/KV）以及一些功能(如PUB/SUB)的提供，在日常应用开发过程中可以为MQ和Cache存在。排序、主从与持久化等功能使得 Redis 一定程度上可以作为数据库进行运用。

# 有用又有趣的设计

## 内置字符串类型SDS

### 结构体设计

Redis基于C语言编写，C语言字符串一般以`\0`结尾。

Redis存储的数据是二进制安全的，即对于存储的数据进行原样存储，不会改变数据的内容。从简单的实现角度，通过`\0`字符来界定字符串长度是一个相当简单的办法，但是倘若我要存储的数据是`ab\0cd\0`呢？这一字符串的长度显然是6而不是2。Redis如果这么实现的话显然不能保证二进制安全。

让我们来看Redis内部的字符串结构的实现：
在2.8版本时，结构实现如下：

```
struct sdshdr {
    unsigned int len;
    unsigned int free;
    char buf[];
};
```

在4.0中，结构实现有所变化，即：

```
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

同样的，还有`sdshdr32`/`sdshdr16`/`sdshdr8`等类型，区别在于`len`与`alloc`等成员的类型。

Redis在存入数据时，将数据按照`原样`存入`buf`之中。如同之前的例子，如何界定字符串长度的工作，交给了`len`，`len`无论在哪一版本的实现中，都代表了已存入的字符串长度，对于数据的读取，可以直接通过`len`得知需要读取的长度，做到了`原样`获取数据。

通过一个变量记录已使用的字符数量，还可以将常见的`strlen`操作时间复杂度从`O(n)`降低到`O(1)`，这是一个空间换时间的操作。

### 空间管理

作为使用者，无需关心Redis内部的内存管理。但是如果作为软件的实现者，这一部分就需要动动脑筋了。

分配空间是个两难的问题，多了浪费，少了触发再分配影响效率。Redis中通过`free`（2.8）或者`alloc - used`（4.0）表征剩余空间，当需要进行空间分配时，如果剩余空间仍然充足，那么直接修改成员变量中的表征空间的数值即可。如果空间不足，那么根据改变存储内容所需空间的大小的变化有如下规则：

+ 所需空间小于`SDS_MAX_PREALLOC`（一般为1MB）的，直接将空间设定为2倍所需空间大小
+ 所需空间大于`SDS_MAX_PREALLOC`的，在所需空间基础之上再增加`SDS_MAX_PREALLOC`的空间
+ 所有分配都会多分配1个字节

