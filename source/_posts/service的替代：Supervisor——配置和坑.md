title: service的替代：Supervisor——配置和坑
author: gwy15
date: 2019-09-14 21:32:40
tags:
---
对于一个想要长期在后台跑的服务，比如一个 web server，一般有这几种方法：
+ 使用 nohup + &
+ 做成一个 service，使用 `sudo service <servicename> <command>` 控制
+ 自行实现 daemon
+ 利用 screen 软件，放到一个 screen 里面跑

第一个方法实用性太差；第二个和第三个在能处理好日志时不错，但是其维护起来较为麻烦；最后一个方法是笔者以前常用的方法。

supervisor 是用 python 编写的程序，其能够将非 daemon 程序放到后台运行并监控，还能做到检测到程序挂掉时重新拉起。它使用了 client/server 架构，因此提供了 web UI 以便远程控制（其实有坑）。

![web UI](/images/2019/09/supervisor.png)

<!-- more -->

# 安装

Ubuntu: `sudo pip install supervisor`

（不建议使用 apt 安装）

# 配置
安装完成后建立配置文件夹，按照惯例我们放到 `/etc/supervisor` 路径下。
```bash
# 将默认设置导入路径
sudo mkdir -p /etc/supervisor/conf.d
sudo sh -c "echo_supervisord_conf > /etc/supervisor/supervisord.conf"
```
下面编辑配置：`vim /etc/supervisor/supervisord.conf`

大部分配置都不需要改，直接将最后的一行改一下：
```ini
[include]
files=/etc/supervisor/conf.d/*.ini
```
对于想要交由 supervisor 托管的程序，在 `/etc/supervisor/conf.d` 路径下建立文件即可，如
```ini
[program:webserver]
directory=/path/to/webserver
command=bash ./run.sh
autostart=true     # 是否随 supervisor 启动
autorestart=true   # 是否挂掉自动拉起
startsecs=3        # 在此时间内退出会被认为失败
startretries=3
user=<username>
```
配置完成后，启动 supervisor 的 server 端：

`sudo supervisord -c /etc/supervisor/supervisor.conf`

supervisor 提供了两种方式进行控制：
+ 命令行式，使用 `sudo supervisorctl` 进入交互模式，也可后跟 command，如：
`sudo supervisorctl status/start/stop/reload/update`
+ Web UI，在坑的部分讲

需要注意的是，`supervisor` 只支持非 daemon 的程序，也就是说假如你想托管本身就支持 daemon 模式的程序（如 docker），你需要去除运行时的 `-d` 参数。

# 坑

下面介绍一下 supervisor 的坑。

它提供了一个 web UI，支持 unix socket 和 http 两种方式。开放之前，先修改一下用户名和密码（`*_http_server.username/password` 字段)。注意这里有个坑，要同步改一下下面的 `supervisorctl.username/password` 字段，否则每次 `supervisorctl` 都要输密码。
（吐槽：明文/裸hash 保存密码……）

改完密码 nginx 直接配置：
```
location / {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unix:/tmp/supervisor.sock;
}
```
配置完一刷新，网关错误，一查log，发现自动建立的文件是 `root|root 700`的，nginx 没有 socket 的访问权限。进配置文件把权限改成 `777`，刷新，终于出来了界面。

点一下界面的 refresh 按钮，发现请求被 301 重定向到 http 了……而且是在 header 里面写死了协议，完全忽略了我们传入的 `proto` 字段。这个问题我翻阅了其[文档](http://supervisord.org/)也没有找到解决方法，最后在 [GitHub](https://github.com/Supervisor/supervisor) 发现很多人也遇到这个问题，提出了很多 issue 和 PR，但是不知道为什么，开发者到现在也不愿意解决这个问题。

最后的结果是，这个所谓的 web UI 实际上只能拿来看，并不能实际控制后台的程序……至少，不能安全地在 https 下使用。