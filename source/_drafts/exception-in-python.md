---
title: Python 中的异常
date: 2018-04-12 11:12:48
tags:
---

承接上篇 [Python 中的异常日志（译）](/exceptional-logging-of-exceptions-in-python)

本文将会补充一些 python exception 的基础知识，错误处理和记录的方式，外加上一些关于 try-except 的技巧。

## Python Exception 基础

在 Python 中，我们可以使用 `raise` 来抛出一个异常，将控制流程切换到异常处理流程

用形如

```python
try:
    # do something
except exception_class as e:
    # handle error here
else:
    # do something when no exception raised
finally:
    # do something whether exception raised or not
```

的语句来处理异常。（`else` 和 `finally` 是可以省略的）

