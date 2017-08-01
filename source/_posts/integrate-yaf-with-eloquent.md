title: Yaf集成Eloquent
date: 2017-07-31 22:48:17
tags: [PHP, Yaf, Eloquent, Laravel]
categories: PHP
---

# TL;DR

在Yaf中集成Eloquent ORM需要通过Composer管理相关依赖，并完成Eloquent的初始化。

# Yaf

[Yaf](https://github.com/laruence/yaf) 是以性能优秀著称的通过PHP扩展实现的框架。

与常规的框架使用上不同，这一框架需要安装PHP扩展才可以使用。安装方法不再赘述。

Yaf 的优点自然是性能，抽象出了常规框架必备的加载、路由等功能，提供了插件机制，对一个请求的一些阶段可以进行hook，实现对输入输出的控制，可谓是极简框架。

但是极简框架带来的问题在于，一些便于代码编写的工具并不会被包含到框架之中，虽然与框架的设计目的相违背，但是对于使用者来说，也增加了一些成本。

# Eloquent

[Eloquent](http://laravel.com/docs/eloquent) 是Laravel框架操作数据库的利器，已被抽出成为独立组件[illuminate/database](https://github.com/illuminate/database)，对于基本的查询工作，省去了编写SQL的时间，同时提供自动修改时间戳，软删除等功能也让开发成本大大降低。`Seeder`与`Migration`工具更是开发过程的有力工具。`Seeder`结合`Faker`方便制造假数据供接口测试，`Migration`则帮助建立库表结构。

# Yaf集成Eloquent

使用Yaf作为PHP开发框架，又想要享受Eloquent便捷的数据库操作方式，唯一的办法自然是将二者集成。




