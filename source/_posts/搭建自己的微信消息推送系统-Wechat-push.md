title: 搭建自己的微信消息推送系统 - Wechat-push
author: gwy15
tags:
  - Python
  - vue
categories:
  - Python
date: 2019-11-02 13:23:00
---
服务器上跑的程序出错了想及时收到通知？对于这类业务报警/消息推送，一般的方法是使用邮件。但是邮箱配置起来较为麻烦，对于懒得配置邮箱或是嫌邮件送达率不高的程序员，微信看起来是一个更好的进行提醒推送的通道。

市面上已经有 [Server 酱](http://sc.ftqq.com/3.version) 或是 [WxPusher](http://wxpusher.dingliqc.com/) 这类解决方案，利用微信服务号的模板消息进行消息推送。但很可惜，这两者都不开源，使用也有些限制，同时UI也无法自定义。对于喜欢折腾的程序员，当然是选择轮子造起来~

下面是博主搭建好的结果示意和开源库地址：
[开源库 wechat-push](https://github.com/gwy15/wechat-push)

![示意图](/images/2019/11/wechat-push-demo.png)

# 使用
遵循 RESTful 风格的代码，发送消息只需要一个请求：
```
POST https://domain.com/message
{
    receiver: openid,
    title: title,
    body: optional, markdown msg,
    url: optional, url at end of the body
}
```
获取自己的 openid，访问页面 `https://domain.com/` 并扫码即可。

<!-- more -->

# 架构
+ 前后端分离
+ 前端：Vue + view-design
+ 后端：Python + aiohttp，提供 RESTful API
+ 数据持久化：SQLite/MySQL 等 SQL 数据库
+ 缓存：redis

# 部署

下面介绍系统的部署

## 要求
+ 一个有微信能扫码的手机
+ 一个公网可访问的服务器，装有 nginx 和 python 3.6/3.7
+ （可选）公网可访问的 80/443 端口。若无该端口，微信拒绝发送回调，因此无法实现扫码获取 open ID，需要用户在微信后台手动查询获取。
+ （可选）redis 服务，若无 redis，同样无法实现扫码获取 open ID（后期会考虑完善）

## 微信服务号配置

首先要说明的是，微信服务号【不对个人开发者开放】，只对公司组织开放。但是如果你没有服务号，可以使用微信官方提供的 [**微信公众平台接口测试帐号** ](https://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login)。个人用户直接扫描即可登陆，无需审核。

![微信测试号管理后台](/images/2019/11/wechat-index.png)

登陆后如上图，我们需要关心的只有 appID、appsecret、接口 URL、接口 Token 这四项。其中，接口 URL 是你提供给微信官方的回调接口，接口 Token 是与微信协商的签名口令。Token 建议生成一个复杂的填入即可。需要注意，这里提交 token 时会进行验证，需要等服务器跑起来后再提交。

在下方“模板消息接口”增加一个模板，标题随意，内容填写
```
{{title.DATA}}
{{body.DATA}}
```
即可。

## 服务器配置

首先 [下载对应平台和python版本最新的 release](https://github.com/gwy15/wechat-push/releases/latest)，到你想要的地方解压。

### 环境变量设置

配置 `.env` 文件：
```
$ cp .env.example .env
$ vim .env
```

```
# .env
APP_ID = wxAppIDGoesHere # 你的 app ID
APP_SECRET = wxAppSecretGoesHere # 你的 app Secret
WECHAT_TOKEN = yourSelfDefinedWechatTokenGoesHere # 你填入微信服务号的配置界面的 token

# DB
SQL_DB_URL = sqlite:///yourDBName.sqlite3 # 数据库 url，可以使用 SQLite、MySQL 等 SQL 数据库
REDIS_URL = redis://@localhost:1234/0 # redis 数据库

# Url Settings
VUE_APP_ROOT_URL = https://your-domain.com/ # 你设置的外网可访问的域名，以 / 结尾
SERVER_API_ROOT = / # 见下方高级配置

LOGGING_CONFIG = logging.json # 日志设置，默认将 DEBUG 级别输出到 stdout
```
### 日志设置

```
$ cp logging.json.example logging.json
$ vim logging.json # 如果你想调整日志设置
```

## nginx 配置
按照 `/config/your-site.conf` 配置即可。

## 启动

不需要配置虚拟环境，linux 下可以直接运行，选择一个端口或是 unix socket 即可
```
$ ./wechat_push.....pex --port 8135
$ ./wechat_push.....pex --sock /tmp/wechat-push.sock
```
对于 Windows 用户，需要在前面加一个 `python3`
```
> python3 wechat_push.....pex --port 8135
```

启动后，如果有 80/443 端口，即可在微信管理后台提交回调接口，
默认回调接口是
`https://your-domain.com/callback`

token可以自定义，保证提交给微信的和环境变量中的一致即可。

## 高级配置

### API 服务器部署在某个路径下
如果你想配置在某个子路径下，譬如将 API 服务器部署在 `POST https://domain.com/wechat-push/message`，需要修改两处环境变量并 **重新编译前端界面**，同时在 nginx 配置中修改 API 地址。
```
# .env
SERVER_API_ROOT = /wechat-push/
```
```
# wechat-push-vue/.env.production
VUE_APP_API_ROOT_URL = https://domain.com/wechat-push/
```
### 前端界面部署在某个路径下
如果你想将前端界面部署在某个子路径下，或是想利用 CDN 对资源进行加速，需要修改两处环境变量并 **重新编译前端界面**，同时在 nginx 配置中修改前端界面地址。
```
# .env
VUE_APP_ROOT_URL = https://domain.com/front-end/
```
```
# wechat-push-vue/.env.production
VUE_APP_ROOT = /front-end/
```