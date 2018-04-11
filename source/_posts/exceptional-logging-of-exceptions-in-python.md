---
title: Python 中的异常日志（译）
date: 2018-04-10 21:02:43
tags: 
- python
- logging
---
本文的作者 Aaron Maxwell 同时也是 [Powerful Python](https://powerfulpython.com/) 的作者.

异常在代码中无处不在。 作为一个开发者，我们总得面对异常处理这一问题。就算是在写一个帮人们找墨西哥卷饼的工具。

哈哈，回到正题。就像我常说的：如何处理异常取决于你使用的编程语言。在运维有一定规模的应用程序时, 日志是最强大最有价值的工具之一。 我们来看看有哪些记录异常日志的方式。

## “Big Tarp” 不管三七二十一全 catch 了式

我们从一个比较极端的例子开始讲起：

```python
try:
    main_loop()
except Exception:
    logger.exception("Fatal error in main loop")
```

这个 try-except 会捕获所有异常。 这种异常捕获方式适合用在那些可能会抛出一些你无法预知的异常的代码块上 (比如 main_loop())。与其让应用就这样崩溃，你可以选择捕获并用日志记录下这一异常，继续让应用运行下去。

这里比较 magic 的地方是选择使用 `logger.exception` 方法 (logger 是调用 logging.getLogger() 返回的一个实例)。这个棒棒的方法会记录下代码块完整的异常栈追踪信息并完整的输出出来。

可以注意到的是这里我们并不需要将异常对象传给 exception 方法，只需要传一个 logging message 字符串。message 会出现在第一行，然后输出完整的异常栈。异常发生时，代码的输出看起来会像是这样：

```
Fatal error in main loop
Traceback (most recent call last):
  File "bigtarp.py", line 14, in
    main_loop()
  File "bigtarp.py", line 9, in main_loop
    print(foo(x))
  File "bigtarp.py", line 4, in foo
    return 10 // n
ZeroDivisionError: integer division or modulo by zero
```

这里的演示都是示例用简单代码的输出，细节并不重要。 只需要注意到第一行是我们传给 `logger.exception()` 方法的 `message`，后面那些行都是异常栈，最后一行是异常类型 (例子中是 `ZeroDivisionError`)。这种捕捉记录方式会捕获任何异常并这样打印出来。

`logger.exception` 方法默认的日志级别是 ERROR。你也可以使用其他常用的 logging 方法 `logger.debug()`、 `logger.info()`、 `logger.warn()` 等等，并传一个 `exc_info` 参数，值为 `True`：

```python
while True:
    try:
        main_loop()
    except Exception:
        logger.error("Fatal error in main loop", exc_info=True)
```

将 `exc_info` 参数设置为 `True` 会导致 logging 记录下完整的 stack trace…。 和 logger.exception() 的行为完全一致. 唯一的不同是这样你可以更方便的指定日志级别。

有趣的地方： 与 Big Tarp 形式对应的，有一个几乎完全相反的形式，接下来你就会看到。

## “Pinpoint” 细粒度式

现在我们来看另一个比较极端的例子。 假设你在写一个叫做 OpenBurrito 的 SDK -- 一个来解决“如何在深夜的时候在你的附近找到一家墨西哥卷饼摊”这个棘手问题的库。假设它有一个函数叫做 `find_burrito_joints()`，它可以返回一个包含有合适餐馆信息的列表。在极少的情况下，这个函数会抛出一个异常：`BurritoCriteriaConflict`。

```python
from openburrito import find_burrito_joints, BurritoCriteriaConflict
# "criteria" is an object defining the kind of burritos you want.
try:
    places = find_burrito_joints(criteria)
except BurritoCriteriaConflict as err:
    logger.warn("Cannot resolve conflicting burrito criteria: {}".format(err.message))
    places = list()
```

这里的方法是通过加上一个 try-except 块来乐观的去执行函数 find_burrito_joints()。 执行的过程中抛出了一个 `BurritoCriteriaConflict` 异常，你用日志记录下了这个异常，解决了问题，继续执行下去。

关键的不同点是这里的 except 语句. 在 Big Tarp 形式中，你基本上是捕获了任意类型的异常；在 Pinpoint 形式下，你捕获了一个非常明确类型的异常，这个异常很语义化的反映了代码中的一个部分。

同样也注意到，这里我是用了 `logger.warn()` 而不是 `logger.exception()`（这篇文章中的 `warn()` 都可以被替换为 `info()`，或 `error()` 等等）。换句话说，我在一个确认的日志级别下记录了异常发生的信息，而不是记录下一整个异常栈。

为什么我不去记录下整个异常栈呢？因为在这种情况下，异常栈并没有什么用处。我捕获的是一个非常明确的异常类型，它在代码逻辑中有非常明确的含义。比如，下面这一小段代码：

```python
characters = {"hero": "Homer", "villain": "Mr. Burns"}
# Insert some code here that may or may not add a key called
# "sidekick" to the characters dictionary.
try:
    sidekick = characters["sidekick"]
except KeyError:
    sidekick = "Milhouse"
```

这里，`KeyError` 不仅仅是一个错误，当它被抛出的时候意味着一个非常明确的条件被触发了：“sidekick” 在我定义的 characters 中是不存在的，所以我选择取一个默认值。 让日志中充满异常栈在这种情况下并没有任何帮助，这时候你就需要使用 Pinpoint 形式了。

## “Transformer” 转换异常类型式

捕获一个异常，给它打日志，然后重新跑出另一个类型的异常。

首先，在 Python 3 中它是这样工作的：

```python
try:
    something()
except SomeError as err:
    logger.warn("...")
    raise DifferentError() from err
```

在 Python 2 中你得去掉 `from err`：

```python
try:
    something()
except SomeError as err:
    logger.warn("...")
    raise DifferentError()
```

(这里蕴含了很多信息，多思考一会儿...) 在原先代码抛出的异常不是很符合你程序的逻辑的时候，你会需要这种异常处理方法。这经常发生在库与库的边界上。

比如，假设你在使用 openBurrito SDK 制作一个帮人们寻找深夜卷饼摊的“杀手级”应用，如果我们过于挑剔 `find_burrito_joints()` 函数可能会抛出一个 `BurritoCriteriaConflict` 异常。这是 SDK 暴露出的 API，但是它不能方便的对应到你代码中高层的逻辑。更合适的是在这里能抛出一个你自己定义的异常类型，叫做 ` NoMatchingRestaurants`。

这种情况下，你需要采用下面这种形式 (针对 Python 3):

```python
from myexceptions import NoMatchingRestaurants
try:
    places = find_burrito_joints(criteria)
except BurritoCriteriaConflict as err:
    logger.warn("Cannot resolve conflicting burrito criteria: {}".format(err.message))
    raise NoMatchingRestaurants(criteria) from err
```

这会输出一个单行日志，并抛出一个新的异常。如果这个新异常没有被捕获，那么错误输出看起来会是这样：

```
Traceback (most recent call last):
  File "transformerB3.py", line 8, in
    places = find_burrito_joints(criteria)
  File "/Users/amax/python-exception-logging-patterns/openburrito.py", line 7, in find_burrito_joints
    raise BurritoCriteriaConflict
openburrito.BurritoCriteriaConflict

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "transformerB3.py", line 11, in
    raise NoMatchingRestaurants(criteria) from err
myexceptions.NoMatchingRestaurants: {'region': 'Chiapas'}
```

这里就有趣了。输出中包含了 `NoMatchingRestaurants` 的异常栈，同时也输出了造成这一异常的 `BurritoCriteriaConflict` 的异常栈…… 非常清楚的表明了谁是最初的异常。

在 Python 3 中，`raise … from … 语法使异常可以被链起来。当你写 `raise NoMatchingRestaurants(criteria) from err`，会抛出一个 `NoMatchingRestaurants` 类型的异常，这个异常对象有一个叫做 `__cause__` 的属性，值指向它的父 exception。 Python 3 内部在报告异常的时候会默认采取这种方式。

能不能在 Python 2 中实现这一特性呢 ？好吧，你不能。这就是那些你需要升级才能享受到的好东西。Python 2 不支持 `raise … from` 这样的语法, 你的异常输出只会包括 `NoMatchingRestaurants` 的异常栈。当然，“Transformer” 仍然能够完美的使用。

## “Message and Raise” 记一下再抛出来式

这种形式下，你在某个点给发生的异常打一个日志，不作处理直接抛出，将异常交给更外层的代码来处理：

```python
try:
    something()
except SomeError:
    logger.warn("...")
    raise
```

你并没有真正意义上的“处理”这个错误，你只是暂时中断了异常抛出并打了一个日志。 当你在外层代码的确有处理这个错误的时候，你想把代码交给外层处理，但又想在代码特定的位置用日志记录这次异常的发生，或者这个异常的含义，可以用这种形式。

这在你查找异常原因，尝试去理解异常发生的上下文的时候非常有用。你可以插入这种 logging 语句来提供有用的信息，或者你想在真实的场景下安全的监控代码的执行结果。

## “Cryptic Message” 打出迷之日志的反形式
现在我们把注意力放到一些“反形式”上… 请不要在你的代码里用这种形式。

```python
try:
    something()
except Exception:
    logger.error("...")
```

假设你团队的其他成员写了这种代码，半年后，你看到一个搞笑的日志消息，像这样：

```
ERROR: something bad happened
```

我希望和祈祷你在你真实的代码中不会遇到 “something bad happened”。然而，你可能会碰到一样很莫名其妙的日志消息，你会怎么办呢？

好吧，首先你需要找到到底是哪一行代码抛出了这个模糊的信息。如果你运气好，grep 以下代码就找到了确切的那一行；你可能在好几个完全不同的地方找到可能抛出这个异常的代码。这会带给你好几个问题：

到底是哪一行触发了异常？

或者是其中几行触发的？

或者是所有这些？

这个日志代表了代码中的哪里？

然而有些时候你根本无法通过 grep 找到异常发生的地方，因为 log 信息是生成出来的。比如：

```python
    what = "something"
    quality = "bad"
    event = "happened"
    logger.error("%s %s %s", what, quality, event)
```

你怎么通过日志来搜索？如果不是你搜遍了代码也没找到匹配日志信息的那一行，你可能根本不会想到这个问题。就算你找到了，也很可能是个错的位置。

最好的办法是传一个 `exc_info` 参数：

```python
try:
    something()
except Exception:
    logger.error("something bad happened", exc_info=True)
```

这样做可以让日志中包含异常栈追踪，告诉你到底是代码中的那部分抛出了异常，谁调用的它 等等… 你排错需要的所有信息都在这。

## 最危险的 Python 反形式

如果让我看见你写这样的东西，我会冲到你家没收你的电脑，然后黑进你的 Github 账号删掉你所有的代码。

```python
try:
    something()
except Exception:
    pass
```

在我的文章，我成这个为 “最危险的 Python 反形式（The Most Diabolical Python Antipattern）”。 这段代码在发生异常的时候不会给你任何信息，而且源头上隐藏了可能发生的任何错误。你可能都不会知道你敲错了一个变量名，其实这最终会造成一个 NameError —— 当你凌晨两点收到告警告知你的生产环境崩掉了，然后花掉你一个通宵来找寻问题原因，因为定位问题需要的信息都被隐藏掉了。

就记住不要干这种事，如果你觉得你需要简单的捕获所有异常并忽略掉，最少用个 “big tarp” （比如使用 `logger.exception()` 而不是 `pass`）

## 更多异常日志形式

Python 中还有更多处理异常的形式，各有各的代价，各有各的优缺点。你知道还有什么有用的形式么？欢迎在文章下面留下评论分享。

原文链接：[https://www.loggly.com/blog/exceptional-logging-of-exceptions-in-python](https://www.loggly.com/blog/exceptional-logging-of-exceptions-in-python)
