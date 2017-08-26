date: 2014-09-07 00:00:00
title: 又写了一个Hello world
categories: Node.js
tags: [Node.js]
---

<style>
img {
	max-width: 600px;
}
</style>

近期有两天没有需求的日子，下周leader说让我去和前端学习一下，不过要先写个通关作业简单了解一下js。于是花两天时间又写了个Hello world。

简单记录一下，代码包含大量的调试语句，暂时不贴代码，后期还得试着填坑，写出一个完善的hello world。

<!-- more -->

## 要求

这个通关作业要求主要有下面一些东西：

+ 用Node.js自己实现一个简单的Http服务，实现BigPipe输出页面
+ 通过$Import加载js依赖模块
+ 通过register方式注册js模块
+ 实现自定义事件的支持，以及广播与事件代理

时间所限，自己只在chrome上完成了测试。

## 实现

### 整体设计

虽然是个Hello world，但是还是简单的思考了一下结构，大致的流程如下图：

![流程图][1]


### Http服务

Node.js的Http模块的使用例子几乎是学习Node.js各类教程的标配了（比如**[七天学会Node.js][2]**……），实现的过程中基本上也是沿用了基本的写法。

其实感觉直接使用**[Express][3]**完成。

### 加载依赖模块

js代码如果能够把每一个部分代码拆分出来单独开发，只加载所需模块，那么对于开发维护，以及网络开销以及用户的体验上个人觉得都是有很大的帮助。自己理解是这一方式可以在浏览器端完成（比如**[requirejs][4]**最近同学推荐的**[seajs][5]**），也可以在服务器端通过主动的合并成单一文件返回给用户。

作业里面的方式是通过在服务器端合并的方式进行的。

由于对Node.js的异步编程思想并没有完全掌握，原先希望的通过异步的加载依赖文件的方式实现了一个下午也没有实现，眼看deadline就要到了，只能换成递归的方式进行处理，依赖文件首先读取并输出。

中间使用了**[UglifyJS2][6]**先对代码进行压缩操作，减小返回数据的大小，顺路去掉注释，简化匹配**$Import**语句使用的正则表达式。

在所有依赖完成读取之后，添加业务逻辑代码。

### BigPipe

**[BigPipe][7]**工作中经常接触，这里的实现并没有特别之处，仍然是首先输出骨架，之后输出各个Pagelet。

具体来说这里使用了Node.js提供的信号功能，骨架输出完成之后会发出骨架已输出（设为`skeleton_ready`），同时计数器设定为Pagelet的数目。之后监听这一信号的各个Pagelet开始执行计算，完成之后直接输出用于渲染的`<script>`标签。骨架部分同时会监听Pagelet输出完成信号(设为`pl_ready`)，之后计数器-1，当计数器为0时输出封闭标签`</body></html>`。

Node.js通过repsonse对象的write方法返回值就能确定缓冲区是否已经刷新（参见**[文档][8]**），类似的，通过PHP实现的话还需要手动的进行ob_flush && flush。

从页面代码上看，大致是这样的一个效果：

![代码效果][9]

### register

这里方法名任意，register的意义在于将当前的Pagelet（以下简称pl）的业务代码加入到当前运行环境中，register作为一个方法，第一参数为模块名称，也就是代码的路径，第二参数即为对应的回调方法。

register方法根据参数1，将回调方法绑定到指定的节点上，例如当前js对象的名称为`Xe`，pl的名称为`pl.index.top`，回调方法为`func1(){...}`，那么在register操作完成之后就可以通过`Xe.pl.index.top()`调用`func1(){...}`。

所有的依赖模块都通过这一方式进行组织。

### 自定义事件

对于自定义事件，目前了解到的应用场景，主要是单个pl中的模块之间的通信，实现上简单的通过保存自定义事件以及对应的回调方法的对应关系，通过对象的call方法调用自定义事件对应的回调方法。

为了能让回调能够异步的执行，还需要使用setTimeout方法。

### 广播

广播用于pl之间的通信，实现方式与自定义事件类似，但是不同点在于，广播中是通过频道对象调用回调方法，在每个频道中维护广播事件与回调之间的关系。

同理，也需要结合使用setTimeout方法。

使用上，每个模块都需要import这一个频道，同时向这个频道订阅指定的事件，绑定对应的回调方法。

### 事件代理

目前事件代理做法是将事件绑定到pl的顶部节点中，同时阻止事件进一步冒泡，由pl的顶级节点处理。

每个事件代理上都有一个对象维护回调方法与节点的对应关系。自己的实现中所有的节点绑定方法都会绑定一个共有的事件处理方法，在这一个方法中获取节点类型以及事件信息，之后调用具体的事件处理回调方法。

## Todo

+ js依赖文件的异步加载

## 其他

有关静态服务器的实现，读了**[@朴灵][10]**大神文章**[《Node.js静态文件服务器实战》][11]**感觉有些启发。



[1]: https://blog.wislay.com/wp-content/uploads/2014/09/node_hello_world_struct.png
[2]: http://nqdeng.github.io/7-days-nodejs
[3]: http://expressjs.com
[4]: http://requirejs.org
[5]: http://seajs.org
[6]: https://github.com/mishoo/UglifyJS2
[7]: https://www.facebook.com/notes/facebook-engineering/bigpipe-pipelining-web-pages-for-high-performance/389414033919
[8]: http://nodejs.org/api/http.html#http_response_write_chunk_encoding
[9]: http://anyofme.qiniudn.com/wp-content/uploads/2014/09/bpout.png
[10]: http://weibo.com/shyvo
[11]: http://www.infoq.com/cn/news/2011/11/tyq-nodejs-static-file-server


