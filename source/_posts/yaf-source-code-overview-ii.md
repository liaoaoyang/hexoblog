title: Yaf源码概览(Part II)
date: 2017-09-19 14:17:39
tags: [Yaf, PHP]
categories: PHP
---

# TL;DR

`Yaf` 版本为 `2.3.0`。

本篇主要简单记录了：

+ yaf_request.c
+ yaf_response.c
+ yaf_router.c
+ yaf_session.c

源码阅读过程中的一些问题和理解。

<!-- more -->

# yaf_request.c

定义了 `Yaf_Request_Abstract` 这一抽象类。同时以及声明了这些类型的 getter / setter 方法。

一个 yaf 的 request 包含了需要调用的 Controller / Action 等等信息。

一个应用场景是在开发过程中，可以通过插件在分发前根据特定情况改动 request 的信息，使得可以更改请求触发的操作对象。

# yaf_response.c

定义了 `Yaf_Response_Abstract` 这一抽象类。

Response 对象中关键的操作是返回内容的操作以及实际返回内容的方法。

返回内容可以在多次的 foward 等过程中，在当前返回数据之前或者之后进行追加操作，也可以直接替换最终返回的数据。这些操作都通过 `yaf_response_alter_body()` 函数实现，这一函数可以支持：

+ YAF_RESPONSE_PREPEND 添加到头部
+ YAF_RESPONSE_APPEND  追加到尾部
+ YAF_RESPONSE_REPLACE 修改数据

相对应的就是 response 对象中的 `prependBody()` / `appendBody()` / `setBody()` 方法。

对于 HTTP 协议上诸如 Header 等内容的操作，也在这个文件中进行了定义。

# yaf_router.c

这是 yaf 框架最为重要的组成部分之一。

在此文件中，定义了 `Yaf_Router` 这一 class。同时也定义了 `static` / `simple` / `supervar` / `rewrite` / `regex` / `map` 这几个路由规则。

路由过程与添加路由规则的顺序相反，在上一篇文章中有所提及。

无论是内置路由规则，还是新增的路由规则，都需要实现 `Yaf_Route_Interface` 这一个接口，实现 `route()` 方法，接受 request 对象，认定为匹配之后，修改当前请求对象的 `module` / `controller` / `action`，并返回真值。

如果没有设定，则使用 `static` 路由规则。

实际使用过程中，可以通过当前 app 的 dispatcher 中的 router 添加一个实例化的路由规则，实现自己路由的目的。

## assemble()

`assemble()` 方法是一个根据自身路由规则拼装出合理 url 的工具，每一个路由类型都需要结合自身的规则，来实现这个方法。

Yaf 路由时需要知道 `module` / `controller` / `action`，所以在调用 `assemble()` 时，自然也要通过数组的方式，传递这些参数，即源码中的：

+ YAF_ROUTE_ASSEMBLE_MOUDLE_FORMAT     :m
+ YAF_ROUTE_ASSEMBLE_CONTROLLER_FORMAT :c
+ YAF_ROUTE_ASSEMBLE_ACTION_FORMAT     :a

这是一个了解各个路由的手段。

关于具体路由的使用，另起篇幅。

# yaf_session.c

定义了 `Yaf_Session` 这一个类。

可以通过 `getInstance()` 方法获得单例。

`Yaf_Session` 类只是对 `$_SESSION` 进行了封装，实际上操作的还是 `$_SESSION` 变量，在初始化过程中将 `$_SESSION` 变量和成员属性 `sess` 进行了关联。

# 参考资料

+ [phpbook](https://github.com/walu/phpbook)

