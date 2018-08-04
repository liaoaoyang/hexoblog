title: Redis中的Lua
date: 2018-08-02 23:42:23
tags: [Redis, Lua]
categories: 系统
---

# TL;DR

Redis 中使用 Lua 的相关笔记。

<!-- lua-in-redis -->
<!-- more -->

# 原理

Lua 这门语言的运行时库大小相当之小，可以嵌入由 C 语言实现的应用程序之中，动态的执行功能。

具体设计与实现在文档中十分清晰，传送门：

+ http://redisbook.readthedocs.io/en/latest/feature/scripting.html
+ http://redisdoc.com/script/eval.html

简而言之，即 Redis 集成了 Lua 的运行时库，加载部分 Lua 的基本库函数，通过伪客户端完成脚本与数据库的交互，转换数据格式。

# Redis + Lua 适合做什么

个人理解，这个组合赋予了 Redis 更强大的服务端计算能力，适合如下特点的任务：

1. 带有特定判断逻辑的操作
2. 耗时低的操作

## 为什么适合带有特定判断逻辑的操作？

### 原子操作里的逻辑判断

Redis 的原子性在 [Redis里那些有用又有趣的设计](https://liaoaoyang.cn/articles/2017/08/06/interesting-and-useful-designs-of-redis/) 一文中有过讨论，单进程单线程按序执行（即按客户端可读可写事件触发事件处理顺序）保证了原子性。

Redis 本身也可以通过 MULTI/EXEC 等命令实现一组原子的操作。

然而，需要注意的是，MULTI 是在所有指令发送完成之后，一次性执行，在执行的过程中，客户端是没有办法及时获得反馈的。如果需要根据某一指令的操作结果，决定下一步执行何种操作，是无法实现的。

Lua 脚本则可以带有判断逻辑，在客户端指令发出后，根据入参可以控制指令的运行分支。

### CAS

Redis 的 CAS 操作可以通过 WATCH 来实现，但是也可以直接通过 Lua 脚本的形式完成，如果还想根据操作的结果完成其他工作，Lua 脚本可能就更为适合了。

希望对事务进行干预的操作，可以考虑使用 Lua 脚本完成。

## 为什么适合耗时低的操作

Redis 的运行模型，决定了强计算任务会阻塞住其他指令的执行，在通过 Lua 扩展功能时也需要注意。

Redis在执行 Lua 脚本时，最大执行时间的长短由 lua-time-limit 选项来控制(以毫秒为单位)，可以通过编辑 redis.conf 文件或者使用 CONFIG GET 和 CONFIG SET 命令来修改它。

当执行超时时，只是重新开始接受新的操作，但是只能处理 `SCRIPT KILL` 和 `SHUTDOWN NOSAVE` 两个命令。如果已经执行了写操作，只能通过 `SHUTDOWN NOSAVE` 重启服务器完成工作了。

所以，从性能和安全角度出发，短小精悍的 Lua 脚本是值得推荐的。

# 如何使用

在 Redis 中通过 Lua 扩展自身的功能，可以有两种方式：直接提供脚本，先导入后使用。

以 [incrby_not_over.lua](https://github.com/liaoaoyang/toolbox/blob/master/softwares/redis/incrby_not_over.lua) 为例：

既可以通过：

```
redis-cli eval "$(cat incrby_not_over.lua)" 1 key_name max_value [step expire]
```

也可以：

```
redis-cli evalsha xxx 1 key_name max_value [step expire]
```

当然，在使用 [EVAL](http://redisdoc.com/script/eval.html) 命令一次之后，可以直接通过 [EVALSHA](http://redisdoc.com/script/evalsha.html) 结合当前脚本的 sha1 值进行调用。

执行过一次 EVAL 和 [SCRIPT LOAD](http://redisdoc.com/script/script_load.html) 的效果基本是一致的，后者不会实际执行脚本。

可以通过 [SCRIPT EXISTS](http://redisdoc.com/script/script_exists.html) 指令判断特定的 sha1 值对应的脚本是否存在。

在 Redis 重启之后，脚本需要重新导入。

# 脚本 sha1 值的生成方式

想要迅速求得 Lua 脚本（假设文件名为`script.lua`）导入之后 sha1 值，可以在 Shell 中使用如下方法：

```
printf "$(cat script.lua)" | sha1sum
```

或者：

```
echo -n "$(cat script.lua)" | sha1sum
```

获取了对应的 sha1 值，可以在编程时预先在代码中写入 sha1 值。

# 实例

## 生产者生产速度控制

在生产者-消费者模型中，假若下游消费速度较慢，可以使用队列等方式暂存生产者生产的任务，起到削峰填谷的作用。

然而，在上游生产速度极快时，队列容量过大也会造成问题。

首先，可以判断队列现有容量，如果队列已经过长，则暂停生产。然而在并发较大的情况下，同一瞬间读取到的队列长度相同且小于最大长度，那么生产者仍会继续生产，给下游带来巨大的压力。

如果直接使用计数器呢？同样的，在同一个瞬间有多个请求一起请求生产，计数器递增，如果计数器值超过了限定的最大值，将计数器回退。如果后续各个请求出现异常，计数器将不会回退值，导致上游无法继续生产。

### 带有超时效果的脚本

计数器不自动回退的问题，可以通过超时进行处理。由于即要计数，又要拥有超时，Redis 原有的 Hash 结构不能满足需求，可以通过 ZSET 模拟超时以及完成计数。将时间戳作为分数即可，每个请求需要有唯一标示。

利用 Redis 命令的原子性，通过 Lua 脚本实现相关逻辑，控制速度。

```
--[[
Usage:
redis-cli eval "$(cat incrby_not_overby_zset.lua)" 1 zset_name max_value client_req_id client_req_timestamp [expire_second]

or

redis-cli evalsha xxx 1 zset_name max_value client_req_id [expire_second]
--]]
if #ARGV >=4 then
    redis.call("ZREMRANGEBYSCORE", KEYS[1], 0, tonumber(ARGV[3]) - tonumber(ARGV[4]))
end

if tonumber(redis.call("ZCARD", KEYS[1])) >= tonumber(ARGV[1]) then
    return -tonumber(ARGV[1])
end

redis.call("ZADD", KEYS[1], ARGV[3], ARGV[2])

return tonumber(redis.call("ZCARD", KEYS[1]))
```

Mac上可以通过如下方式测试命令：

```
redis-cli eval "$(cat incrby_not_over_by_zset.lua)" 1 ks1 10 `uuidgen` `date +%s` 10
```

即 10 秒内只能允许 10 次生产。

但是上述方法存在一个隐患，即各个生产者的时间戳并不一定同步，有可能出现超前或者靠后的情况，造成意外的过期问题。

此处有人会疑问，为何不通过 `redis.call("TIME")[1]` 获取时间戳，一旦使用了这个方法，在 3.2 以下版本的 Redis 中将无法继续执行任何写入操作（参见SO上的 [Why redis cannot execute non deterministic commands in lua script
](https://stackoverflow.com/questions/40013598/why-redis-cannot-execute-non-deterministic-commands-in-lua-script) 答案）。

> The scripts are replicated to slaves by sending the script over and running it on the slave, so the script needs to always produce the same results every time it's run or the data on the slave will diverge from the data on the master.

> You could try the new 'scripts effects replication' in the same link if you need perform non deterministic operations in a script.

脚本中的操作尚不能像 MySQL 的 Binlog 一样采用混合模式传输，对于随机等操作的值直接传输值。

### 根据队列长度进行控制

直接将判断队列长度和 PUSH 操作合二为一：

```
--[[
Usage:
redis-cli eval "$(cat incrby_not_over_by_llen.lua)" 2 queue_name max_value msg [LPUSH]

or

redis-cli evalsha xxx 2 queue_name max_value msg [LPUSH]
--]]
local PUSH = "LPUSH"

if #ARGV >= 3 then
    PUSH = "RPUSH"
end

if redis.call("LLEN", KEYS[1]) >= tonumber(ARGV[1]) then
    return -tonumber(ARGV[1])
end

return redis.call("LPUSH", KEYS[1], ARGV[2])
```

这个方法的问题是每次都会带上需要加入的数据，如果生产者单次需要PUSH数据量小，并且获取数据的成本比较低，可以考虑直接如此改造。

# 参考

+ http://redisbook.readthedocs.io/en/latest/feature/scripting.html
+ http://redisdoc.com/script/eval.html
+ https://github.com/hellokangning/Redis-note/blob/master/20.%20Lua%E8%84%9A%E6%9C%AC.md



