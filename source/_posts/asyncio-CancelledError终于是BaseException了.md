title: asyncio.CancelledError终于是BaseException了
author: gwy15
date: 2019-09-04 10:44:27
tags:
---
经常写 Python 异步代码的人可能都知道，当前 Python 版本（`3.7.4-`）中，，当一个 coro 被取消时，会抛出 `CancelledError`，而这个 `CancelledError` 定义为：
```Python
# asyncio/exceptions.py
class CancelledError(concurrent.futures.CancelledError):	
    """The Future or Task was cancelled."""
```
<!-- more -->

`concurrent.futures.CancelledError` 定义在 `lib/concurrent/futures/_base.py`中：
```Python
class Error(Exception):
    """Base class for all future-related exceptions."""
    pass

class CancelledError(Error):
    """The Future was cancelled."""
    pass
```
也就是说，`CancelledError`的继承链是
```
Exception
↑
concurrent.futures.Error
↑
concurrent.futures.CancelledError
↑
CancelledError
```

问题就出在这个 Exception 身上。平时我们写代码，在不关心异常的情况下经常会有这样的设计：
```Python
while True:
    try:
        res = await download()
        break
    exception Exception as ex:
        logger.warning(ex)
        continue
await handle(res)
```
如果运行时，任务在 `download()` 中就被取消，那么此时抛出的 `CancelledError` 会被 `exception Exception` 捕捉到，从而无法正确取消。所以，为了满足设计要求，我们实际上需要把上面的代码修改成：
```Python
try:
    res = await download()
    break
exception CancelledError:
    raise
exception Exception as ex:
    logger.warning(ex)
    continue
```
这就非常不合理了，每一个涉及到捕捉异常的部分都需要如此处理。而如果按照类似 `KeyboardInterrupt` 或是 `SystemExit`的继承链，直接继承 `BaseException`，显然就不会出现这种问题。

在2018年1月，有人在 BPO 提出了这个问题：[BPO32528](https://bugs.python.org/issue32528)，在经过几番争吵后，终于这个看上去有点破坏向后兼容的设计失误（就我看来并不会有大的影响）在2019年5月被[修改了](https://github.com/python/cpython/pull/13528)，并应该会在 Python 3.9.0 跟我们见面（预计[2020年6月](https://www.python.org/dev/peps/pep-0596/)）🤦‍


