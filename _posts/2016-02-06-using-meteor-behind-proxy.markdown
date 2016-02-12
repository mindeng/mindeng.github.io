---
layout: post
title:  "Using Meteor Behind HTTP Proxy"
date:   2015-11-09
tags:   [meteor, proxy, polipo]
---

由于众所周知的原因，国内使用 meteor 时经常会碰到网络连接不上的问题, 尤其在添加包时，例如：

```
meteor add twbs:bootstrap
```

官方 [wiki](https://github.com/meteor/meteor/wiki/Using-Meteor-behind-a-proxy) 提到可以
通过设置环境变量 *HTTP_PROXY*, *HTTPS_PROXY* 来解决。在实践当中，发现 meteor 不像其他工具
（例如浏览器，或者curl）那样，可以接受 SOCKS 代理，必须使用 http 代理。

这里记录下实践过程及最终解决方案。

动态端口转发
=============

首先尝试使用动态端口转发做代理：

```
$ ssh -CNf -D7777 server
$ export HTTP_PROXY=localhost:7777
$ export HTTPS_PROXY=localhost:7777
```

测试 `meteor add` 命令，连接失败。

远程 squid 代理
=================

接下来尝试另一种方式，在远程服务器搭建一个 squid 服务（搭建方法可参考
[Use Squid Server as HTTP Proxy](http://www.minotes.net/notes/45) ），
并将本地端口 7778 映射到远程服务 squid 的 3128 端口:

```
$ ssh -CNf -L 7778:localhost:3128 server
$ export HTTP_PROXY=localhost:7778
$ export HTTPS_PROXY=localhost:7778
```

经测试还是不行。

squid + polipo
================

想起之前用过一个轻量级的 http 代理服务器 polipo，于是想到通过
polipo 把 SOCKS 代理转成 http 代理。

OS X 上可以使用 `brew install polipo` 安装 polipo 。
polipo 的配置有两种方式，一种是配置文件，还有一种是配置在内存中（用来测试非常方便）。

```
$ ssh -CNf -D7777 server
$ polipo
```

这里介绍下内存配置方式。启动 polipo 进程后，在浏览器打开 http://localhost:8123 并点击
 *The configuration interface* -> *Current configuration* 进入配置页面，找到 *socksParentProxy*
配置项，在右边输入框内填入 `localhost:7777` 并按回车键即可。

配置完 polipo 后，运行如下命令：

```
$ export HTTP_PROXY=localhost:8123
$ export HTTPS_PROXY=localhost:8123
```

测试 `meteor add` 命令，成功！
