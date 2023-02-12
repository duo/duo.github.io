---
title: "Telegram, QQ and Wechat"
date: 2023-02-12T10:59:08+08:00
lastmod: 2023-02-12T15:22:08+08:00
tags: ["telegram", "qq", "wechat", "go"]
categories: ["Telegram"]
draft: false
---

## 背景

嗯, 我又滚回 Telegram 了...

使用 [Matrix](https://matrix.org/) 的时候有很多不爽的地方, 比如 [<<通过 Matrix 协议实现代理接收 QQ 和微信消息 —— 从入坑到入土>>](https://blog.arisa.moe/blog/2023/230114-matrix-qq-wechat-bridge/) 这里的一些吐嘈, 我也有遇到. 对我来说, 问题遇到了能修我就动手了, 但是客户端的易用性实在很糟心, 总不至于去重写一个客户端吧...

看到 Telegram 的 Bot API 支持了群组话题功能, 这样之前困扰的消息归档可以比较好的解决. 至于没有虚拟帐号, 无法很好区分发言人这个问题, 等以后有自定义表情的 API 后, 应该能变相解掉吧. 像 At 用户, 同步头像等一些功能, 我自己很少用, 就当不存在了 XD

元宵在老家把之前 Telegram 方案的陈年代码找了出来, 改一改把话题相关功能加上, 就开始用起来了, 相关代码放到了 [octopus](https://github.com/duo/octopus). 虽然没有 [EH Forwarder Bot](https://ehforwarderbot.readthedocs.io/en/latest/) 那么强大, 不过自用够了.

自己用的时候, 把常用的联系人和群, 链接到 Telegram 的群里, 一些频次低的, 链接到了话题群, 然后其它的群就落到配置里的归档话题群, 最终的效果如下图:

{{< image src="/images/telegram-qq-wechat.png" >}}

## 部署

先放一下最终的目录结构 (完整版 {{< link href="/files/octopus/docker-compose.yml" content=docker-compose.yml >}})
```sh
.
├── docker-compose.yml
├── octopus
│   └── configure.yaml
├── octopus-qq
│   └── configure.yaml
├── octopus-wechat
│   └── configure.yaml
└── octopus-wechat-web
    └── configure.yaml
```

### Bot

通过 @BotFather 创建一个 bot, 得到对应的 token , 然后使用 `/setjoingroups` 开启 bot 加入群组的权限, 以及使用 `/setprivacy` 禁用该 bot 的隐私限制以使其能读取群内消息.

然后在 Telegram 上给该 bot 发条消息, 之后访问 `https://api.telegram.org/bot$token/getUpdates` (将 $token 替换成刚才得到的 bot token), 通过消息的 from 属性得到自己 Telegram 帐号的 id (用做后面配置的 `admin_id`)

### Octopus

编辑以下文件内容
**octopus/configure.yaml**
```yaml
master:
  admin_id: <Telegram user id>
  token: <Telegram bot token>
  #proxy: http://1.2.3.4:7890 # proxy for connecting to Telegram

service:
  addr: 0.0.0.0:11111
  secret: <your secret key>

log:
  level: info
```
{{< admonition type=tip >}}
如果直连 Telegram 有问题, 开启 proxy 配置
{{< /admonition >}}

**docker-compose.yml**
```yaml
version: "3.8"

services:
  octopus:
    hostname: octopus
    container_name: octopus
    image: lxduo/octopus:latest
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
    volumes:
      - ./octopus:/data
    networks:
      - octopus-net      

networks:
  octopus-net:
    attachable: true
```

执行 `docker-compose up` 启动看看是否正常

### Octopus QQ

编辑以下文件内容
**octopus-qq/configure.yaml**
```yaml
limb:
  account: <your qq number>
  password: <your qq password>

service:
  addr: ws://octopus:11111
  secret: <your secret key>

log:
  level: info
```

{{< admonition type=tip >}}
不配置 account 和 password, 则启用手机 QQ 扫码登陆
{{< /admonition >}}

编辑 **docker-compose.yml**, 加入
```yaml
  octopus-qq:
    hostname: octopus-qq
    container_name: octopus-qq
    image: lxduo/octopus-qq:latest
    restart: unless-stopped
    depends_on:
      - octopus
    environment:
      TZ: Asia/Shanghai
    volumes:
      - ./octopus-qq:/data
    networks:
      - octopus-net
```

启动 `docker-compose up`, 如果提示密码登陆验证, 就 attach 上去输入

### Octopus WeChat

微信的话, 有 Web 的版本 (使用 [openwechat](https://github.com/eatmoreapple/openwechat)) 和 PC Hook 的版本 (使用 [ComWeChatRobot](https://github.com/ljc545w/ComWeChatRobot))

PC Hook 的版本相对功能较全, 但是比较吃资源, 然后 Web 版本的 openwechat 库最近很多人用于接入 ChatGPT, 容易被风控, 就看怎么取舍了

#### WeChat Web
编辑以下文件内容
**octopus-wechat-web/configure.yaml**
```yaml
limb:

service:
  addr: ws://octopus:11111
  secret: <your secret key>

log:
  level: info
```

编辑 **docker-compose.yml**, 加入
```yaml
  octopus-wechat-web:
    hostname: octopus-wechat-web
    container_name: octopus-wechat-web
    image: lxduo/octopus-wechat-web:latest
    restart: unless-stopped
    depends_on:
      - octopus
    environment:
      TZ: Asia/Shanghai
    volumes:
      - ./octopus-wechat-web:/data
    networks:
      - octopus-net
```

启动 `docker-compose up`, 然后扫码登陆

#### WeChat PC Hook
编辑以下文件内容
**octopus-wechat/configure.yaml**
```yaml
limb:
  version: 3.8.1.26
  listen_port: 22222
  hook_port: 33333

service:
  addr: ws://octopus:11111
  secret: <your secret key>

log:
  level: info
```

编辑 **docker-compose.yml**, 加入
```yaml
  octopus-wechat:
    hostname: octopus-wechat
    container_name: octopus-wechat
    image: lxduo/octopus-wechat:latest
    restart: unless-stopped
    depends_on:
      - octopus
    environment:
      TZ: Asia/Shanghai
      VNCPASS: <your vnc password>
    ports:
      - 5905:5905
    volumes:
      - ./octopus-wechat/configure.yaml:/home/user/configure.yaml
    networks:
      - octopus-net
```

启动 `docker-compose up`, VNC 连上去进行扫码登陆

{{< admonition type=tip >}}
也可以在 windows 下运行, 从 [octopus-wechat](https://github.com/duo/octopus-wechat/releases) 下载执行文件以及从 [ComWeChatRobot](https://github.com/ljc545w/ComWeChatRobot/releases) 下载对应的动态库, 和 configure.yaml 放在一起即可

安装的微信版本要求是 3.7.0.30, 以及可能需要安装 Visual C++ Redistributable (https://docs.microsoft.com/en-US/cpp/windows/latest-supported-vc-redist?view=msvc-170)
{{< /admonition >}}

## 使用

### 链接

从端的消息默认都会落到和 admin 的会话里, 可以在 Telegram 新建群, 把 bot 加进去后, 执行 `/link` 来将选中的从端会话进行链接, 这样该会话的消息就会落到链接的群里.

{{< admonition type=tip >}}
1. 可以通过 `/link xxx` 过滤名字为 xxx 的从端会话
2. 可以在话题群的 `# General` 下链接多个从端会话, 匹配的消息会自动创建对应的话题进行归类 (请确保 bot 有该群的管理话题权限)
{{< /admonition >}}

### 归档
算是个 one more thing 的环节吧, 未链接的从端消息默认会落到和 admin 的会话里, 显得比较杂乱, 所以可以使用 Archive 的功能

具体操作如下:
 * 首先创建一个开启话题功能的群, 然后把 bot 加进去, 授予 bot 管理话题的权限
 * 从 octopus 的日志里的类似 `LimbClient(qq;1234567) websocket connected` 信息可以得到登陆的从端的 uid (比如这里的1234567)
 * 在该群里发条消息, 然后访问 `https://api.telegram.org/bot$token/getUpdates` (将 $token 替换成 bot token), 通过消息的 chat 属性拿到该群的 id
 * 编辑 **octopus/configure.yaml**, 在 master 配置下加入
```yaml
  archive:
    - vendor: qq
      uid: <your qq uid>
      chat_id: <your chat id>
```
 * 重启 octopus

这样从登陆了该 uid 的 qq 从端来的消息, 如果未找到任何对应链接的群, 就会发到该 chat_id 的话题群里, 并自动创建对应的话题进行归类.
