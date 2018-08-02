title: Redis中的Lua
date: 2018-08-02 23:42:23
tags: [Redis, Lua]
categories: 系统
---

# TL;DR

Redis 中使用 Lua 的相关笔记。

<!-- lua-in-redis -->
<!-- more -->

# 脚本 sha1 值的生成方式

## 在 Redis 中使用 Lua

## SCRIPT LOAD 导入的 Lua 脚本的 sha1

想要迅速求得 Lua 脚本（假设文件名为`script.lua`）导入之后 sha1 值，可以在 Shell 中使用如下方法：

```
printf "$(cat script.lua)" | sha1sum
```


