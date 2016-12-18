title: 《事务处理原理》
date: 2016-11-30 23:50:31
tags: [Transaction]
categories: Reading
---

# 概述

年中参与了一个售卖系统的开发工作，开发过程中涉及较多的数据库事务操作以及多层系统之间的状态同步问题。对于事务大方向上的理解，个人觉得需要加强。

查阅相关书籍，发现首推的都是Jim Grey编写的[《事务处理》](https://book.douban.com/subject/1144543/)。此书虽然经典，但是篇幅较大（800+页），希望快速了解一些共性原理的情况下，同事推荐了这本[《事务处理原理》](https://book.douban.com/subject/5412835/)。

在工作之余，挑选了自己较为关注的章节进行阅读并做了一些笔记。

# 分章节笔记

## 第1章 简介

> 事务处理(Transaction Processing, TP)

对于真实世界中的交换产生的记账行为，通过网络连接起来计算机进行处理。

> 在线事务(Online Transaction)

在线用户执行程序，操作数据库。

>  ACID

原子性(Atomicity)：事务或全部执行，或不执行，
一致性(Consistency)：事务保持数据库的内部一致性
隔离性(Isolation)：每个事务不影响其他事务
持久性(Durability)：故障发生时事务不丢失

通过ACID测试才能称之为一个事务系统。

> 原子性(Atomicity)

成功称为提交`commit`，事务异常终止则是`abort`。

对于`已提交`的事务发现错误的，需要有事务补偿（compensating transaction）机制，考虑到任何事务都有可能失败，设计良好的TP系统需要对各类事务有补偿机制。但是补偿仍有可能不够完备，可以通过其他策略进行修正。

> 一致性(Consistency)

数据库事务的一致性指的是“内部一致性”，即满足所有逻辑上的约束，不能出现不合理的值。

一致性不仅仅需要事务系统负责，更需要应用程序参与，而且应用程序负主责。

> 隔离性(Isolation)

隔离性的关键是可串行化。即执行事务集和按顺序逐个执行事务结果一致。

加锁机制是大多数数据库系统保持隔离性的一种方法。

> 持久性(Durability)

事务执行完成，并一定存入稳定存储器，才能称作实现了持久性。

持久性的主要实现方式之一即将事务更新的副本追加到日志文件中获得的，事务提交时，系统会确保写入日志已经写入磁盘，之后返回给应用程序事务已被提交。如果是数据库程序，实际入库时机不定。事务系统崩溃之后，会通过日志进行恢复，且必须通过日志进行恢复。

> 两阶段提交(2PC two-phase commit)

解决涉及多个数据库系统的数据不同步的问题。

由事务管理模块控制。执行两阶段提交时，事务管理器发出两轮消息，第一轮通知所有资源将事务结果写入稳定存储中，所有资源均确认已写入后，发出第二轮消息，通知资源管理器实际提交事务。写入的数据量要足以恢复提交前的状态。

## 第2章 事务处理抽象

> 集合事务（Transaction Bracketing）

个人理解：集合事务是一组标准的编程模式，提供启动(Start)、提交（Commit）与异常终止（Abort）三个事务命令。

> 组合事务的两种解决方法

+ 实现Start时只执行第一个Start操作，实现Commit操作时只执行最后一个Commit操作
+ 实现每一个事务组合实现包装器代码，在包装器中进行Start/Commit操作，事务操作编写为元方法

> 事务标识符

每个事务都应该有个唯一的事务标识符，启动时分配，通过标识符关联全部事务操作。

> 嵌套事务

嵌套事务在大多数商用系统中不被支持，其表示模式如下：

+ 事务中使用Start命令，则这一事务是其所在事务的子事务
+ 不在事务中使用Start，则这一事务成为顶级事务
+ 顶级的Commit和Abort操作可以影响子事务
+ 子事务Abort只通知父事务，父事务酌情处理
+ 子事务之间是隔离的

> 必备的异常处理

事务故障的恢复，系统故障恢复。

> 保存点（Savepoints）

保存点是通过通知资源记录操作完成状态实现回退部分事务的一种机制。保存点只会保存顶级事务带来的影响，当子事务调用保存点记录状态时，子事务存在并发操作的情况下，使用保存点无法回退单一子事务带来的影响。

## 第3章 事务处理应用程序体系结构

> 事务处理系统的影响因素

+ 组件是否共享地址空间
+ 地址空间是否有多个线程中的一个线程在执行
+ 是否有硬件、OS或者语言机制保护共享地址空间的程序，免于意外修改内存

> 事务中间件

提供了API以及各类工具，提供与DB和应用程序的交互功能，对开发人员屏蔽事务操作中的操作系统级别的问题，如多线程、通信和安全性。

> 存储过程VS事务服务器/中间件

存储过程可以完成事务服务器的工作，然而为什么事务服务器/中间件仍然有市场？

原因在于：

+ 数据库对于新的通信协议、特性的添加相当缓慢且稳定
+ 事务服务器/中间件对开发者更友好（语言，特性，规避问题）
+ 事务服务器可水平扩展性远优于数据库（通过事务服务器更容易扩容）

## 第4章 队列化的事务处理

> 队列化的事务处理

+ 队列分为请求队列以及应答队列
+ 队列必须写入到非易失性存储之中
+ TP程序从请求队列获取任务，处理后写入应答队列
+ 客户端和服务端需要在会话中提供足够的上下文，用于恢复和按序处理

> 队列化的事务处理对于不可撤销操作的处理

对于输出，如果只是涉及到非实物的情况，那么影响并不大。
但是当涉及金钱出入等问题时，需要处理，方法是在出应答队列之前先写入日志，如果没有前期的写入日志的情况，那么说明并没有执行过这一步骤，如果存在，需要人工告知或者通过其他手段确认到底是否可以执行这一操作。总而言之，即先记录动作，再实际操作（带来的问题可能是对用户的不良体验，但是保证了TP系统不会遭受损失）。

个人理解：比如ATM出钞操作，步骤如下：

1. 由于已经扣款，那么取款机启动一个事务，从应答队列中，取出此次取款操作的返回值，即成功取款；
2. 本地查看是否有本次取款的物理出钞记录，没有则进行到3，有则终止
3. 记录本次出钞
4. 实际打开取钞盒，吐出钞票
5. 提交事务

## 第6章 锁定

> 两阶段锁定（two-phase locking）

一个事务必须在释放其获得的锁之前得到他的锁定。

目的在于不留出因为先解锁后加锁带来的可供其他事物修改已锁定目标的时间段。两阶段锁定生效的有效的前提是事务只通过事务内读取和写入两种操作进行交互，如果借助了其他手段，那么会破坏可靠性。

如果一个执行中的事务都遵循两阶段锁定，此执行就是可串行化的。

> 死锁解决的唯一方法

终止死锁涉及到的事务之一。

> 死锁检测的技术

`基于超时`与`基于图形`的检测。

`基于超时`的检测易于实现，但是可能会终止实际上并没有死锁的事务，同样的由于基于时间进行控制，也会让死锁持续过久。

`基于图形`的检测，通过`等待图`(waits-for graph)这一工具进行，从直观上，节点表示事务，边根据方向A->B，表示A上等待着B上的一个锁定被解除，当锁定解除时，边会被删除，当出现闭环时，说明存在死锁。检查可以是定期检测。同时，基于图形的死锁检测同时也可以在边上设定允许的等待时间，超时的循环存在则可以有效的检验死锁。

然而基于超时的检测并非一无是处，在异构拓扑，以及分布式网络环境中，效果较好。

> 死锁牺牲品的选择

+ 产生了循环的事务（最易于发现）
+ 锁数量最少的事务（完成最少工作量）
+ 生成日志最少的事务（成本最低）
+ 写入锁数量最少的事务（成本最低）

> 死锁的循环式重新启动问题（cyclic restart）

被牺牲的事务因为重新启动又触发死锁，导致无法进行。解决办法是可以将启动时间作为牺牲的考虑因素，牺牲最晚出现的事务。

> 死锁触发的因素

+ 锁转换（lock conversion）：多个事务同时要求将同一位置上的读锁转换为写锁，由于两阶段锁定的限制，需要先解锁读锁，但是多个事务互相等待其他事务上读锁的释放，于是死锁；解决方法是对于影响范围先上写锁，确定不需要更改时降级为读锁定，或者使用不和读锁冲突的共享锁模式，先上共享锁，会比先上写锁的方式提高一定并发度。共享锁因为和读锁不冲突，不需要进行解锁等操作，所以不会触发死锁。
+ 锁抖动（lock thrashing）：启动太多事务，由于请求锁的数量倍增，会阻塞住大量事务，造成吞吐量减小的情况。

## 第7章 系统恢复

> 系统故障的衡量指标MTBF与MTTR

MTBF即Mean Time Between Failures，即系统出现故障前的平均运行时间，MTTR则是Mean Time To Repair，即系统出现故障之后进行修复所用时间。

可用性可以定义为MTBF/(MTBF+MTTR)。

> TP系统服务端恢复的原则

非幂等的操作，恢复时对于没有完成的任务需要执行完成，而不是重试。

系统恢复的关注点应该是已执行的事务，这样会简化系统恢复（基于事务的服务器恢复无需复制运行时内存状态，只需通过增量的改动日志进行恢复，不完整的事务不会被理会）。

> 基于保存点的非幂等操作服务端恢复

基于保存点的非幂等操作的服务端恢复，需要在执行非幂等操作之前，将内存状态保存到非易失性存储中。

这样操作可以使得系统在恢复过程中确定执行非幂等操作时数据的状态，以便从指定的时间点完成任务，防止出现错误的重复操作。

> 数据库的故障类型

+ 事务故障：事务运行异常
+ 系统故障：主存发生问题
+ 媒介故障：持久化存储器发生问题

> 数据库的恢复策略

+ 事务故障：恢复到事务执行之前的数据状态
+ 系统故障：中断所有未提交的事务，保证写入已提交事务的结果
+ 媒介故障：与系统故障基本相同

> 数据库错误恢复的日志格式要点

+ 被更新的页面地址
+ 执行更新事务的标识符
+ 页面被写入后的值（后像）
+ 页面被写入前的值（前像）
+ 冲突操作的实际执行顺序

> 数据库恢复管理器应该具有的三大操作

+ 提交：原子的将更新后数据写入数据库，不可撤销
+ 中断：将事务带来的数据影响还原，不可撤销
+ 重启：中断故障时所有未提交的的事务，写入已提交的事务尚未同步到数据文件中的数据。这一个操作必须是幂等的

> 预写日志协议

先改动数据文件，后写入前像日志，在第二步出现错误时，系统恢复，会引起前像丢失的问题，无法恢复。

预写日志协议的定义是：不要讲未提交的更新输出到稳定数据库中，直至其前像日志记录被输出到日志之中。

通过这一方式，数据库可以实现异常终止，明确恢复时可以关闭的事务，以及需要补充写入数据文件的部分。

No-Steal方法是一个简化的规避上述问题的方法，即未提交的改动全部记录到日志之中，事务提交之后才将改动写入到DB之中（问题：如何保证提交之后，数据写入稳定存储和其他事务读取了正确的数据？事务已提交，结果就能被读取，未写入到稳定存储中，也需要能被读取？）。

> 提交时的强制规则

与预写日志协议相冲突，这一规则的定义为：在事务所有已更新数据的后像写入到稳定存储之前，不能提交事务。

> 故障恢复方法（Shadow-paging）

核心思想是维护两份数据，数据库中的数据只存储已提交的事务，提交事务的操作即将数据指向已写入稳定存储中的状态副本（shadow page）。

所谓Shadow page可以看做是数据的目录，会指向实际的数据页面文件，在Shadow page完成指向之后，数据库会更换shadow page的指向，之前的关联关系就会被废弃。

中断事务因为没有做实际的改变（指向关系没有变动），所以可以直接通过废弃主存中的修改数据以及已写入稳定存储的数据，无需做特别恢复操作。

Shadow-paging方法在书中提到在商业系统不常用的原因，虽然简单，但是提交时直接将整个页面写入稳定存储，需要耗费大量的IO操作，涉及大量的随机写，在机械盘时代必然会有其劣势。

> 基于日志完成中断

基于日志完成中断操作时，事务管理器需要从日志记录的的最后一个更新开始，用前像替换已更改的数据文件。

实现中断操作的问题在于，顺序查找日志中的更新事务效率低下，所以在需要实现一份事务与事务最后一条执行日志的匹配关系，同时事务的每条日志也要记录事务的上一条位置的指针（文件指针？），便于从后往前还原前像。

> 基于日志完成恢复

基于日志完成恢复，仍然是通过从后向前扫描日志进行，直到完成到检查点。

原因在于，最后一条日志可以明确事务是否已提交。中断的事务会被丢弃，并恢复前像；已提交的事务则会检查并写入数据；而运行中的事务，则会检查并写入后像，继续运行这一事务。

恢复时会阻塞所有新事务请求。

> 日志恢复速度优化——模糊检查点

核心在于，正常运行时写入检查点之前，脏页面已经全部写入。写入操作在系统空闲时进行。由于写入新检查点的前提是之前一个检查点之前数据已经完成写入，所以检查需要从第二个检查点之后进行。

## 第8章 两阶段提交

> 两阶段提交的处理对象

应对多个资源管理器的事务操作。

> 2PC的四个通信指令

REQUEST-TO-PREPARE：协调者->参与者
PREPRARED：参与者->协调者（失败则发送NO）
COMMIT：协调者->参与者
DONE：参与者->协调者

然而，任意阶段出现问题时，协调者可能还会发出ABORT指令，让参与者放弃事务，解锁资源。

> 2PC PREPARE

2PC中PREPARE所做的操作，是持久存储了事务的余像（afterimages）。

> 参与者与协调者的等待时机

参与者需要在不明确收到COMMIT/ABORT指令时进行等待（参与者可以请求协调者要求指令状态）。
协调者则需要在未收到DONE消息时进行等待（然而协调者还会定时要求参与者反馈状态）。

> 参与者与协调者写入日志的时机

参与者：PREPARED与DONE发送之前
协调者：REQUREST-TO-PREPARE与COMMIT发送之前（REQUREST-TO-PREPARE可以优化不写入，COMMIT日志存在的前提下，可以给参与者请求执行指令时有明确回复）

> 2PC可以使用三轮通信的条件

+ 只有一个参与者且参与者没有独立的准备阶段

可以在REQUEST-TO-PREPARE这一个阶段完成之后，将协调者的角色转移到参与者，通过新的协调者（即参与者）完成本地事务之后，发送COMMIT，驱动新参与者（原协调者）发送DONE消息，完成整个事务流程。由于角色变更，新协调者（即参与者）必须要等待DONE消息

> 合作终止协议

处理参与者者宕机恢复而此时协调者也宕机的情况，通过了解其他参与者收到的通知，决定本参与者执行的操作。参与者告知自己收到的状态，未收到REQUREST-TO-PREPARE的情况的，需要回复ABORT指令，而已PREPARED的，因为不知道原协调者的指令，需要回复UNCERTAIN。

