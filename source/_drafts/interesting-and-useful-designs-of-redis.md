title: Redis里那些有用又有趣的设计
date: 2017-07-29 11:34:44
tags: [Redis]
categories: 系统
---

# TL;DR

[Redis](https://redis.io/) 单进程单线程的运行模式，保证了操作的原子性。丰富的数据结构（LIST/HASH/ZSET/KV）以及一些功能(如PUB/SUB)的提供，在日常应用开发过程中可以为MQ和Cache存在。排序、主从与持久化等功能使得 Redis 一定程度上可以作为数据库进行运用。

<!-- more -->

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

通过一个变量记录已使用的字符数量，还可以将常见的`strlen`操作时间复杂度从`O(n)`降低到`O(1)`，这是一个空间换时间的操作。同样的做法还出现在链表结构中有所体现。

### 空间管理

作为使用者，无需关心Redis内部的内存管理。但是如果作为软件的实现者，这一部分就需要动动脑筋了。

分配空间是个两难的问题，多了浪费，少了触发再分配影响效率。Redis中通过`free`（2.8）或者`alloc - used`（4.0）表征剩余空间，当需要进行空间分配时，如果剩余空间仍然充足，那么直接修改成员变量中的表征空间的数值即可。如果空间不足，那么根据改变存储内容所需空间的大小的变化有如下规则：

+ 所需空间小于`SDS_MAX_PREALLOC`（一般为1MB）的，直接将空间设定为2倍所需空间大小
+ 所需空间大于`SDS_MAX_PREALLOC`的，在所需空间基础之上再增加`SDS_MAX_PREALLOC`的空间
+ 所有分配都会多分配1个字节

### 兼容已有功能

在SDS的空间分配过程中，总会多分配一个字节，这是由SDS相关的API操作影响的。SDS相关的一些API会在写入数据时，将`\0`写入`buf`的末尾。通过这些操作，在一定程度上可以复用原有的一些C语言的字符串函数。

### 小结

对于日常的开发工作来说，SDS中可以学习到的思路有：

+ 对高频操作/耗时操作可以做空间换时间的选择
+ 最大程度兼容已有工具

## 字典DICT

Redis中的字典实现了键值对映射的功能，也就是常用的Hash等方法的底层实现。

### 结构体设计

Redis的dict这一数据结构由几个部分组成，先来看看2.8版本的实现。

最顶层的数据结构为`dict`：

```
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;
```

Hash数据的入口由`dictht`实现：

```
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

而每一项数据的节点则由`dictEntry`实现：

```
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

从结构体来说，一个dict的查询过程可以简单的描述为：

+ 进入dict的`dictht ht[i]`成员中
+ 根据key值调用hash方法之后，在`dictht ht[i]`中的`dictEntry **table`中找到相应的位置
+ 取出`dictEntry table[j]`中存储的值

部分数据结构设定得较为奇怪，下面来逐个说明。

#### dictEntry中的next指针

Hash的实现过程中，必然需要解决的问题就是Hash冲突，即两个不同的key值通过Hash算法处理之后得到的值，极有可能是同一个值。

对于Hash冲突，常规的解决方案是：

+ `开放地址`：发生冲突时，尝试下一个单元（或者随机一个单元，或者在冲突位置左右逐个尝试）。优点是简单，缺点是存在再冲突的可能性，结果分布可能比较集中；
+ `重新Hash`：发生冲突时，通过准备好的下一个Hash函数重新计算值，得出位置。优点是不容易产生聚集，缺点是耗时；
+ `链表法`：在冲突位置使用链表记录所有的冲突项。优点也是简单，但是缺点是极端情况退化较为严重，查询操作时间复杂度上升；

Redis显然是选择了链表法解决这一个问题，通过dictEntry中的`next`指针，指向下一个满足这个hash值的项目。

那么问题就来了：

1. 在极限情况下，新增的值应该加入链表头还是链表尾呢？
2. 如何防止退化？

对于问题1，Redis会将最新值放在链表头部，因为新值被访问的概率更高。
对于问题2，则交给剩下的内容进行解释。

#### dict中的两个dictht

dict结构中有一个两单元的数组`dictht ht[2]`，让人困惑的是，作为hash表的实际载体，为什么需要在一个字典里存储两个这样的对象？

***首先从上一个部分提出的问题开始，如何防止退化？***

Hash结构之所以快，是因为可以直接的找到对应key的值，如果现在存储的键值数量大于Hash结构中Hash结果值数组的容量，即必定存在了Hash冲突。Redis通过链表法解决了冲突，必然会让`O(1)`的查找速度退化为`O(n)`的线性查找速度。此时此刻，我们为了维持效率，需要做的是什么？

答案当然是对Hash结果值结构`dictht`的对象进行扩容。在分配数据时，一般按照 2^n 的个数进行分配，这给扩容带来了极大的方便。Redis在对dict进行扩容时，容量直接X2，使得可用的空间从 2^n 升级到 2^n+1。

描述了这么多内容，和两个`dictht`有什么关系呢？当然有关系。在扩容的时候，这两个`dictht`，一个表示正在进行存储的操作单元，一个表示正在进行扩容操作的单元，在扩容完成后，将扩容操作的单元变为实际存储单元。

第二个好处还在于，如此实现之后，扩容时无需阻塞其他操作，等待扩容操作完成。Hash结构相当复杂，存储的数据可能相当巨大，如果用阻塞其他操作的办法进行扩容，应该是最简单做法了，但是作为高性能的内存数据库，Redis显然不会选择这样的方式。两个`dictht`在扩容时进行查询，只需要在两个成员中都进行查找即可，写入数据时，只需要要向正在扩容的`dictht`成员写入即可，删除数据类似查询操作，更新亦然。通过这种渐进式的rehash操作，让数据逐步的迁移到新的`dictht`之中。

综上，设定两个`dictht`就不用新增一套空间变化这类操作时所需内存管理的体系，简化了操作步骤，无需通过阻塞其他请求的方式完成响应的操作。

***再来说说空间管理的问题。***

Hash表如果之前有大量的KV数据存在，现在做了各类操作之后，可能KV值的数量骤减，此时是继续让Hash表占用空间吗？作为一个内存数据库，效率和空间占用是必须考虑的问题。此时此刻应该缩容。

缩容的条件一般是`使用空间`和`已申请空间`比例小于0.1时触发，此时，也会利用两个`dictht`的数组进行和扩容类似的操作。

### 小结

对于日常的开发工作来说，dict中可以学习到的思路有：

+ 将耗时操作分而治之，均摊到一些日常操作上，降低成本

## 跳表

跳表是在接触Redis之后了解到的一个有趣的数据结构，简而言之，跳表是在链表的基础上，增加层级的概念，每个层级只有特定的一些元素，一旦某层存在一个元素，所有低于这一层次的层级都会有这一元素。查找时通过最高层到最底层逐个查找，可以跳过一定个数的元素，这就是名字中“跳”字的由来。

### 应用范围

Redis中ZSET就是了跳表作为基础之一的数据结构，ZSET即有序集合，可以实现排行榜等功能。

为什么说跳表只是构成ZSET的基础数据结构之一，我们来看一下ZSET的数据结构（位于`redis.h`中）：

```
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

当中除了跳表`zskiplist`之外，还有个一个`dict`成员。

这一dict成员的作用是记录存入对象的key和value之间的一一对应关系，如果要增加或者删除key值时，会联动的删除dict中的数据。所以`ZSCORE`[手册](http://redisdoc.com/sorted_set/zscore.html)提到的*O(1)*时间复杂度也不难理解了。查询一个有序列表中KEY的值是一个高频操作，冗余了数据，但是缺提高了性能。

跳表的优势在于便于理解，实现难度低，然而查询时间复杂度近似于平衡二叉树的效果，带来的成本是空间的增加，即增加了大量的指针用于每层元素之间的关联。从实际使用上来看，带来的空间成本增加其实相对其他数据结构操作上的难度和代码复杂度来说，使用跳表是一个可以接受的方案。

个人通过Java实现了一个跳表的[样例](https://github.com/liaoaoyang/LeetCode/blob/master/src/main/java/co/iay/leetcode/DataStructures/MySkipList.java)以及[测试用例](https://github.com/liaoaoyang/LeetCode/blob/master/src/main/java/co/iay/leetcode/Tester/DataStructures/TestMySkipList.java)。

### 小结

对于日常的开发工作来说，跳表中可以学习到的思路有：

+ 在需要速度的场合，如果在空间复杂度允许的前提下，尽量使用更为简单的数据结构
+ 适当的冗余数据，提高高频操作的便捷程度

## 操作的原子性

一个操作是原子的，说明这个操作的结果只有两个：`成功`或者`失败`。不会存在说存在部分修改这样的一个中间状态。

Redis的单个操作比如`INCR/HINCRBY`等是原子的，这说明使用Redis对某个值做+1操作时，不会出现增加0.5这样的情况。

MySQL InnoDB的事务中的原子性通过REDO/UNDO日志来记录所有操作，在发生意外情况时保证了一个事务之内操作的原子性。对于出现竞争的情况（多个操作者修改同一个数据），通过锁定和MVCC等方法，保证每一个操作的原子性（有明确的状态迁移过程）。

Redis作为高性能的内存数据库，同一时间内并发操作的客户端请求必定是一个常见的应用场景，那么Redis是怎么实现操作的原子的呢？个人认为Redis实现得相当简单：`单进程单线程按序执行`。

首先从执行者上，`redis-server`主进程只有一个，也并没产生多个线程，执行者层面上不会出现竞争的关系。

其次从调用者上，虽然客户端会有多个，并且所有操作都是并发的，但是Redis通过自己实现的事件处理`ae`系列函数（姑且如此称呼），串行化的接收各类文件描述符的读写事件，并根据注册的回调进行处理。

如此一来，一个操作必定在完成之后才能进行下一个操作，而每一个操作在没有竞争的情况下，必定只有成功与失败两种结果，Redis基本操作的原子性可以保证。


### Redis的命令执行过程

这里稍微展开说一下Redis的单个命令从客户端到服务器执行的过程。以下篇幅不会讨论`epoll/select/kqueue`等IO多路复用在`ae`中的具体实现细节，所有描述以`ae`提供的API为基础，同时也不会讨论ae中的定时事件。

#### redis-server初始化

一个网络服务器，必定要经过bind->listen->accept->read->write这一系列步骤，才能完成和客户端的一次交互。

作为一个内存数据库，Redis客户端发出网络连接，完成后发送请求，等待服务器处理，处理完成后Redis服务器会返回执行结果。

Redis服务器的初始化可以在源码的`redis.c`文件中的`initServer()`函数中看到，在网络方面主要的操作步骤是：

+ 创建ae事件循环
+ 根据配置中的侦听端口，初始化各个侦听端口，对各个侦听端口完成bind/listen的操作
+ 将侦听的fd加入到ae事件循环侦听的fd列表中，通过`aeCreateFileEvent`注册`AE_READABLE`事件到回调函数`acceptTcpHandler`之上，即当侦听的fd可读时，调用`acceptTcpHandler`，即执行accept方法

在ae事件循环过程中，会无限的调用`aeProcessEvents`获取现在可以执行的事件，然后会逐个进行处理，这些步骤都是串行的，这里是所有数据库操作起点，所以每一个redis操作也都是串行的。

`aeProcessEvents`实际调用的是`aeApiPoll`获取可供操作的事件，由于每个事件除了fd之外还有其他的一些属性，ae的事件循环中以fd作为下标，在一个大小为`maxclients + 96`的类型为`aeFileEvent`数组中，记录了fd对应的ae事件结构，每次在通过`aeApiPoll`获取有事件产生的fd之后，可以直接通过fd作为下标，找到对应的附加信息（需要处理的事件类型和对应的回调函数，事件的附加信息）；此外会用了一个同样大小的数组`aeFiredEvent`数组，记录事件的触发执行情况，防止读事件在处理过程中被处理两次。

#### redis-server执行命令

Redis服务器在被客户端连接之后，会从连接的fd中读取数据，从accept开始之后的流程如下：

+ 当`acceptTcpHandler`被调用时，accept操作返回的fd会当做参数，传递给`createClient`方法，进行客户端的创建
+ `createClient`创建客户端的过程，会注册这一连接的fd的`AE_READABLE`事件到`readQueryFromClient`回调函数上
+ 即当客户端发送消息时，会在ae事件循环中，发现客户端fd产生了可读事件时，调用`readQueryFromClient`方法，将操作指令读取到客户端结构的缓冲区中并调用`processInputBuffer`执行指令
+ `processInputBuffer`实际上调用`processCommand`解析指令，操作对应的redis内存空间，完成后通过名字为`addReply`开头的各类函数返回数据
+ `addReply`实际完成的工作是向ae事件循环中注册写事件`AE_WRITABLE`的回调方法`sendReplyToClient`，向返回客户端的数据缓冲区中添加数据

#### redis-server返回指令

Redis服务器在执行完成指令之后，由于已经注册了写事件`AE_WRITABLE`，在下一次执行`aeProcessEvents`获取到了某个客户端的fd可写的情况下，就会调用注册的`sendReplyToClient`方法，将从`events`数组中获取的aeFileEvent结构取出，在当中的`client_data`数据项之中得知对应的客户端结构，从而得知需要从客户端结构的缓冲区中的数据是哪些，通过`write`系统调用回写数据到客户端


