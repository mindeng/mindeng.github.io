+++
title = "Doom Emacs 的基本用法"
date = 2023-12-08T16:49:00+08:00
lastmod = 2023-12-08T17:01:09+08:00
tags = ["emacs"]
draft = false
+++

## 安装 {#安装}

```bash
git clone --depth 1 https://github.com/doomemacs/doomemacs ~/.config/emacs
~/.config/emacs/bin/doom install
```


### 环境变量 {#环境变量}

执行 `doom env` 命令，可以 dump 一份当前的 shell 环境变量，Doom 启动时会加载该环境变量。如果你的环境变量配置发生变化（例如修改了 PATH 配置），则应该重新执行一次该命令。


## 配置 &amp; 同步 {#配置-and-同步}

配置主要是修改两个文件：

`init.el`
: 主要用于配置 Doom 的模块，可以开启、关闭模块，也可以修改模块的一些选项。

`packages.el`
: 主要用于安装一些第三方的 Emacs 插件，语法类似于
    `straight-use-package` 。其底层就是通过 `straight.el` 来实现的。

> ‼️ 上述两个文件修改之和，都需要执行 `doom sync` 命令，并重启 Emacs 才能生效。


## 升级 {#升级}

升级分两部分：

-   doom 本身及其管理的 pacakges
-   `packages.el` 里面安装的第三方 pacakges


### 升级 doom 本身及其管理的 pacakges {#升级-doom-本身及其管理的-pacakges}

1.  `doom upgrade` 升级 doom 及其安装的 pacakges
2.  `doom build` 重新编译所有的 pacakges
3.  然后重启 Emacs 即可。


### 升级第三方 pacakges {#升级第三方-pacakges}

1.  在 Emacs 中执行 `M-x straight-pull-all` 升级所有的 pacakges 。
    -   或者，执行 `M-x straight-pull-package-and-deps` 升级指定的 package 及其倚赖。
2.  在命令行中执行 `doom build` 重新编译所有的 pacakges 。
3.  最后，重启 Emacs 即可。


## 诊断 {#诊断}

执行 `doom doctor` 命令，可以对系统环境和配置进行诊断。


## 参考 {#参考}

-   [GitHub - doomemacs/doomemacs: An Emacs framework for the stubborn martian hacker](https://github.com/doomemacs/doomemacs)
