title: PHP多进程中使用file_put_contents安全吗？
date: 2017-11-13 23:07:11
tags: [PHP]

---

# TL;DR

PHP多进程使用 `file_put_contents()` 方法记录日志时，使用追加模式（`FILE_APPEND`），日志内容不会重叠，即能安全的记录日志内容。

<!-- more -->

# 参考文档

[Is lock-free logging safe?](https://www.jstorimer.com/blogs/workingwithcode/7982047-is-lock-free-logging-safe)


