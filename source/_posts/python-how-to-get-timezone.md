date: 2014-05-23 00:00:00
title: Python如何获取时区
categories: Learning
tags: [Python]
---


想要在Python中获取时区信息，其实完全可以通过两句简单地代码就能实现(Python运行时复制内容)：

```
>>> import time
>>> print(time.strftime("%z", time.gmtime()))
+0800
```

参见**[Python2的doc][1]**，其实就是利用**[time.strftime][2]**格式化输出**[time.gmtime][3]**返回的数值。

格式串中的小写z表示的时`+0800`格式的时区数据。

[1]: https://docs.python.org/2/library/time.html
[2]: https://docs.python.org/2/library/time.html#time.strftime
[3]: https://docs.python.org/2/library/time.html#time.gmtime
