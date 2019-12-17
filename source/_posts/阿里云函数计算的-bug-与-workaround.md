title: 阿里云函数计算的 bug 与 workaround
author: gwy15
tags:
  - Python
  - web
  - 阿里云
  - WSGI
  - http
categories:
  - Python
date: 2019-12-17 14:38:00
---
去年写的阿里云函数计算是直接用 WSGI 接口撸的，最近引入 flask 框架重构了。但是重构并在本地测试通过后，放到云上遇到了问题。函数计算会对 url 中存在非 ascii 字符的情况报错。

<!-- more -->

## 错误 demo

对于这样一个简单的代码：

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/<string:name>')
def echo(name):
    return jsonify({'name': name})

def handler(environ, start_response):
    return app(environ, start_response)
```

在 python2 下，

```
GET /test
GET /测试
```

都可以通过，但是对于 python3 环境，

```
GET /test
GET /测试
```

第二个却会抛出 `UnicodeDecodeError` 异常。



## 错误原因

通过翻阅 WSGI 接口规范 [pep-3333](https://www.python.org/dev/peps/pep-3333/)，可以发现阿里云函数计算实现的 WSGI 接口不规范。先看看 pep-3333 怎么定义的字符串和编码问题：

> But in many Python versions and implementations, strings are Unicode, rather than bytes. This requires a careful balance between a usable API and correct translations between bytes and text in the context of HTTP... especially to support porting code between Python implementations with different `str` types.
>
> WSGI therefore defines two kinds of "string":
>
> - "Native" strings (which are always implemented using the type named `str`) that are used for request/response headers and metadata
> - "Bytestrings" (which are implemented using the `bytes` type in Python 3, and `str` elsewhere), that are used for the bodies of requests and responses (e.g. POST/PUT input data and HTML page outputs).
>
> Do not be confused however: even if Python's `str` type is actually Unicode "under the hood", the *content* of native strings must still be translatable to bytes via the Latin-1 encoding! (See the section on [Unicode Issues](https://www.python.org/dev/peps/pep-3333/#unicode-issues) later in this document for more details.)

可以看到，pep-3333 定义的 string 都是 `latin-1` 编码的 `str`，为了将其转换为 python3 中的字符串，werkzeug 库是这样实现的：

> `return s.encode('latin-1').decode('utf8)`

而阿里云函数计算传入的是一个 unicode string，所以没法正确被 werkzeug 处理。

## 解决方法

知道了错误原理，我们就可以进行暂时的 workaround：

```python
# aliyun FC workaround
def handler(environ, start_response):
    environ['PATH_INFO'] = environ['PATH_INFO'].encode('utf8').decode('latin1')
    return app(environ, start_response)
```

我已经将这个 bug 通过工单系统提交给了阿里云，不知道他们愿不愿意修复。