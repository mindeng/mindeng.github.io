
---
title: "用 Emacs 分析日志文件"
date: 2023-03-14T11:47:00.000Z
lastmod: 2023-03-15T03:32:00.000Z
tags: ['emacs', 'tools', 'regexp']
draft: false
---


日常开发工作中，经常会需要分析日志文件，有一件趁手的工具会高效很多。

Emacs 正是这样一个工具。

> Vim 也有类似的功能（参考 [Vim Tips]({{< relref "vim-tips" >}})），但就分析日志来说，似乎没有 Emacs 来得方便。  


## 统计 & 搜索

``M-x count-matches`` 可输入正则表达式，统计正则匹配到的次数

``C-M-s`` 正则搜索

> 在输入正则表达式时，如果需要匹配换行符，请输入 ``C-j`` 。  


## 替换

``C-M-%`` ``query-replace-regexp`` 正则替换


## 过滤

``M-s o pattern RET`` 列出所有匹配行

``keep-lines`` 仅保留匹配的行

``flush-lines`` 剔除匹配的行


## 提取

``C-u M-s o pattern RET`` 将所有匹配的文本，导出到 **Occur* buffer *中。这个功能挺方便的，可以快速将匹配项提取到另一个 buffer 中。
