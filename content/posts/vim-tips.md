
---
title: "Vim Tips"
date: 2022-09-26T10:59:00.000Z
lastmod: 2023-01-09T07:03:00.000Z
tags: ['tools', 'editor']
draft: false
---



## 日志过滤


### global

```sql
# [d]elete all lines not(!) matching patterns
:g!/pattern/d
:v/pattern/d

# 匹配多个单词
:v/onStart\|onStop/d

# 忽略大小写，可以直接 `:set ignorecase`，或者：
# 强制忽略大小写：\c
:g/\cpattern/z#.1|echo "================================"
# 强制匹配大小写：\C
:g/\Cpattern/z#.1|echo "================================"

# 更多信息可查看帮助 `:help /ignorecase`

# 将匹配的行移动到最后吗
:g/pattern/m0
# 将匹配的行移动到最前面（顺序会变成倒序）
:g/pat/m$

# 展示匹配该正则表达式的列表
:g/regular-expression/p
```


### [I, ilist

```sql
# Display all lines that contain the keyword under the cursor.
[I

# Like "[I" and "]I", but search in [range] lines	(default: whole file).
:il /pattern1\|patter2\|pattern3/
```


### quickfix, location list

```sql
:vimgrep pattern %
:lvim pattern %
```


## 文件编辑相关


### Json 格式化

```python
# V 选中 json 字符串，然后调用 json.tool 格式化 json 字符串
:'<,'>!python3 -m json.tool
```


### 二进制编辑

使用 ``xxd`` 命令，切换到二进制模式： ``:%!xxd``

退出二进制模式： ``:%!xxd -r``


### 使用指定的 encoding 重新加载文件

``:e ++enc=gbk``


## 搜索

```shell
# 搜索时使用智能大小写
set ignorecase smartcase

# 增量搜索，光标自动跳转到匹配的位置
set incsearch
```


## 配置相关


### 重新加载 .vimrc

配置修改后，执行 ``:source ~/.vimrc`` 即可生效。
