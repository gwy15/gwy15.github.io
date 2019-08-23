title: 一些Python的snippets
author: 'gwy15, gwy15thu@gmail.com'
tags:
  - Python
categories:
  - Python
date: 2019-08-22 17:18:00
---
### log 以天为单位分文件保存

以前我用的比较多的是 [`RotatingFileHandler`](https://docs.python.org/3.7/library/logging.handlers.html#rotatingfilehandler)，这个根据文件大小进行分割。有的时候我们对文件大小限制并不大，而对日期更敏感一点。这个时候可以用官方库里面的 [`TimedRotatingFileHandler`](https://docs.python.org/3.7/library/logging.handlers.html#timedrotatingfilehandler)。

```Python
from logging.handlers import TimedRotatingFileHandler
handler = TimedRotatingFileHandler(
    logFileName, encoding='utf8',
    when='midnight', atTime=time(4, 0, 0), # generate backup file at 4 am
    backupCount=31)
```

### 缓存函数结果一段时间

Python 库里面有一个 [`@functools.lru_cache(maxsize=128, typed=False)`](https://docs.python.org/3.7/library/functools.html#functools.lru_cache) 修饰器，可以缓存最近的`maxsize`个结果。但是有的时候需要修饰的函数并不是一个纯函数，譬如其可能请求一个列表；但是我们对它的结果的即时性要求不高，这个时候可以考虑使用一个缓存将其结果缓存一段时间，以减少调用该函数的时间。

使用 [`cachetools`](https://cachetools.readthedocs.io/en/stable/) 包，该包里面实现了该功能。

```Python
import time
from cachetools import func as functools

@functools.ttl_cache(maxsize=128, ttl=600)
async def doSomething(args):
    pass
```

