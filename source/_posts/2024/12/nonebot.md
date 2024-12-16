---
title: NoneBot部署指南
filename: nonebot.md
date: 2024/12/16 12:00:00
updated: 2024/12/17 3:00:00
categories:
  - NoneBot
tags:
  - NoneBot
  - QQ机器人
index_img: https://dodo939.oss-cn-beijing.aliyuncs.com/blog/2024/12/nonebot-0.png
banner_img: https://dodo939.oss-cn-beijing.aliyuncs.com/blog/2024/12/nonebot-0.png
---

# NoneBot部署指南

## 前言

第一次搭建 QQ 机器人大概是在一年前，当时用的是 [go-cqhttp](https://docs.go-cqhttp.org/) 和一个什么野生机器人框架来着已经忘记了，后来腾讯加强了客户端的加密协议，非官方的客户端通过自建签名服务器的方式已经几乎不可能，所以当时放弃了继续维护机器人（毕竟当时只是跟着视频看一步做一步，对原理没有深刻的理解）。当时在 issues 里看到了有人提到无头 QQNT 的替代方案，不过由于也是一头雾水，就放弃了。

最近，应群友要求，想要一个 QQ 机器人，就去学习了一下，经过一番折腾，终于是稳定地运行了。

## 运作原理

俗话说，授人以鱼不如授人以渔。不能只会按部就班地随便网上找篇教程跟着做，到后面出了奇怪的问题，或者这种方案不再可用，就麻烦了。理解 QQ 机器人的运作原理，在你遇到问题时，就可以自己定位到问题的根本，也更方便向他人求助。

> 提问的智慧：https://github.com/ryanhanwu/How-To-Ask-Questions-The-Smart-Way/blob/main/README-zh_CN.md
>
> **本指南不提供此项目的实际支持服务！**

首先，先放张图上来：

![NapCatの魔法](https://dodo939.oss-cn-beijing.aliyuncs.com/blog/2024/12/nonebot-1.png)

正常情况下，是我们的 QQNT 客户端和腾讯服务器进行通信。NapCat 是接下来要用到的一个妙妙工具，她通过 JavaScript 交互可以改变 QQNT 客户端的运行逻辑，比如控制客户端发送消息等等，因此，从腾讯服务器那边看来，NapCat 就像一个正常的用户一样操作，极其难以被检测。但是只有 NapCat 也不行，她并不知道我们的目的，因此就需要一个角色来告诉她要做什么，我们的 NoneBot 就登场了，他对应的就是图中的“插件框架”。NoneBot 和 NapCat 通过 WebSocket 协议进行通信，NoneBot 用于处理事务逻辑。

说了这么多，举个例子：假如有个机器人它的功能是计算两个数之和，那么当机器人收到"1+1"时，就会自动发送"2"。这期间都发生了些什么呢？

首先，QQNT 客户端正常地从腾讯服务器获取消息"1+1"，此时 NapCat 通过 ws 协议向 NoneBot 汇报自己收到了"1+1"，NoneBot 知道自己的任务是计算两个数之和，于是他算出了"2"，并通过 ws 向 NapCat 命令：让客户端发送"2"，于是 NapCat 就控制 QQNT 客户端发送"2"，在腾讯服务器看来，就像我们自己发了一个"2"过去一样。

## 搭建部署

下面内容适用于有基本的计算机操作技能的人群。

### 购买服务器

机器人肯定要 24h 在线工作的，所以拥有一台服务器是必须的，除非你没这个需求或者你的电脑能 7*24h 不关机。如果你没有，那么就需要购买一台，像阿里云，腾讯云，甲骨文，Azure 等等这些提供云服务的商家，都可以成为你的选择。配置要求非常低，不过建议至少有 1G 的内存。

总之现在假设你已经有了一台服务器，以 Ubuntu 系统为例。

### 安装配置 NoneBot

NoneBot 官方网站：https://nonebot.dev

#### 安装脚手架

首先确保你已经安装了 Python 3.9 及以上版本，然后在终端运行以下命令：

1. 安装 [pipx](https://pipx.pypa.io/stable/)

   pipx 是一个比 pip 更好用的包管理器。

   ``````bash
   python -m pip install --user pipx
   python -m pipx ensurepath
   ``````

   如果在此步骤的输出中出现了“open a new terminal”或者“re-login”字样，那么请关闭当前终端并重新打开一个新的终端。

2. 安装脚手架

   ``````bash
   pipx install nb-cli
   ``````

安装完成后，你可以在命令行使用 `nb` 命令来使用脚手架。如果出现无法找到命令的情况（例如出现“Command not found”字样），请参考 [pipx 文档](https://pipx.pypa.io/stable/) 检查你的环境变量。

#### 创建项目

使用脚手架来创建一个项目：

```bash
nb create
```

你会看到一些询问：

```plaintext
[?] 选择一个要使用的模板: simple (插件开发者)
[?] 项目名称: test-bot
[?] 要使用哪些适配器? OneBot V11 (OneBot V11 协议)
[?] 要使用哪些驱动器? FastAPI (FastAPI 驱动器)
[?] 请输入插件存储位置: 1) 在 "test_bot" 文件夹中
[?] 立即安装依赖? (Y/n) Yes
[?] 创建虚拟环境? (Y/n) Yes
[?] 要使用哪些内置插件? echo
```

项目名称自定义，内置插件选择 `echo` 插件作为示例。这是一个简单的复读回显插件，可以用于测试你的机器人是否正常运行。

#### 安装插件

访问 [官方插件商店](https://nonebot.dev/store/plugins)，复制安装命令在项目根目录运行即可。

#### 运行项目

进入 test-bot 目录，首先将 `.env` 文件第一行修改为 `ENVIRONMENT=prod`，使环境变量修改为生产环境。接着打开 `.env.prod`，加入两行内容：

```ini
HOST=0.0.0.0
PORT=11451
```

其中 `PORT` 的值随意，但最好是大于 10000 避免与常用端口冲突，且一定不能超过 65535。接着运行：

``````bash
np run --reload
``````

推荐使用 [screen](https://www.runoob.com/linux/linux-comm-screen.html) 命令来实现后台运行。

### 安装配置 NapCat

NapCat 官方网站：https://napneko.pages.dev/

#### 安装

这里以 Ubuntu 为例，其他系统可参照 [官方文档](https://napneko.pages.dev/guide/start-install) 进行安装。

对于 Ubuntu 20+/Debian 10+/Centos9，可以直接使用 [NapCat.Installer](https://github.com/NapNeko/NapCat-Installer) 安装脚本一键安装。

``````bash
curl -o napcat.sh https://nclatest.znin.net/NapNeko/NapCat-Installer/main/script/install.sh && sudo bash napcat.sh --cli y
``````

#### 启动

``````bash
napcat start XXXXXXXXXX(机器人QQ号)
``````

如需配置开机自启：

``````bash
napcat startup XXXXXXXXXX
``````

#### 配置

首先查看日志：

``````bash
napcat log XXXXXXXXXX
``````

按照提示扫码登陆，之后在其中找到“WebUi Local Panel Url: ......”这一行，进入这个网址，注意 `127.0.0.1` 要换成你服务器公网 IP 再访问，记得在服务器防火墙放行对应端口。

进入面板后选择左侧“网络配置”，添加 wsClient 配置，如图所示进行配置，其中 URL 的端口号要和上面 NoneBot 环境变量设置的端口一致。

![NapCat配置](https://dodo939.oss-cn-beijing.aliyuncs.com/blog/2024/12/nonebot-2.png)

完成之后，NapCat 应当已经与 NoneBot 成功连接。

## 结束

恭喜你，完成以上步骤，你的机器人应当已经正常运行。试着在群里@Ta或私聊使用 `/echo hello` 指令吧！可以去[安装插件](#安装插件)来丰富机器人的功能了。

如果有任何问题，请正确地寻求帮助，同时你也可以在下边留言。
