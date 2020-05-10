---
layout: post
title:  "Linux tips"
date:   2020-05-10
tags:   [linux, admin]
---

1. `script /dev/null`  
    该命令可以解决 su 切换用户后，运行 screen 程序时报错：

    ```
    Cannot open your terminal '/dev/pts/0' - please check.
    ```

    的问题。原理是 `script` 命令会开启一个新的伪终端（pseudo-terminal），用来 dump 终端上的所有输出。