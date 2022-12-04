
---
title: "Vim Tips"
date: 2022-09-26T10:59:00.000Z
lastmod: 2022-12-04T13:18:00.000Z
tags: ['tools', 'editor']
draft: false
---



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
