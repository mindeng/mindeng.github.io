
---
title: "Emacs Tips"
date: 2022-12-03T09:59:00.000Z
lastmod: 2023-04-25T13:43:00.000Z
tags: ['emacs', 'tools', 'editor']
draft: false
---



## Macro 宏录制 & 回放  
  
1.  开始录制： ``C-x (``  
1.  结束录制： ``C-x )``  
1.  回放宏： ``C-x e`` 


## Regexp 相关

``M-x`` ``count-matches`` 统计正则匹配到的次数

``C-M-s`` 正则搜索

``M-x`` ``replace-regexp`` 正则替换

```bash
# 示例：在每行后面追加文本 xyz
M-x replace-regexp RET $ RET xyz RET

mauth\([^-]\) → mauth-front\1
```

``C-M-%`` or ``query-replace-regexp`` 正则查找 & 替换

高级用法示例：

```bash
# 执行替换：auth\b\([^-]\) → auth-front\1
C-M-% auth\b\([^-]\) RET auth-front\1 RET
```

上述示例的执行效果：  
  
1.  首先匹配到所有 auth 开头的单词，并且排除掉 "auth-" 的 case；  
1.  将匹配到的字符串，例如 “auth:”、“auth+” 等，后面插入 “-front”，例如替换为：”auth-front:”、”auth-front+”

``M-s`` ``o pattern RET`` 列出所有匹配行

``C-u`` ``M-s`` ``o pattern RET`` 将所有匹配的文本，导出到 **Occur* buffer 中。*

> 在输入正则表达式时，如果需要匹配换行符，请输入 ``C-j`` 。  

``M-x`` ``keep-lines`` 仅保留匹配的行

``M-x`` ``flush-lines`` 剔除匹配的行

``C-u C-x =`` 显示光标所在字符的详细信息，包括正则分组、语法等信息。


### 在多个文件中查找、替换  
  
1.  ``M-x find-name-dired``: 会提示你选择一个目录，以及一个 filename wildcard 用于匹配你想要查找的所有文件，例如 ``*.el`` 会匹配所有扩展名为 ``.el`` 的文件；  
1.  按 ``t`` 选中所有匹配到的文件；  
1.  按 ``Q`` 执行 "Query replace regexp in marked files"，即正则查找和替换；  
1.  接下来的流程和 ``query-replace-regexp`` 类似，跟着提示走即可；  
1.  按 ``C-x s`` 保存所有修改过的 buffers，按 ``y`` 保存, ``n`` 跳过,  ``!`` 保存所有文件。


### 正则表达式语法

参考：[https://www.emacswiki.org/emacs/RegularExpression](https://www.emacswiki.org/emacs/RegularExpression)

Emacs 的正则表达式和 [Python 的规则](https://docs.python.org/3/library/re.html)有些类似，但还是有不少差异。

一些值得注意 or 有意思的示例：  
  
-   ``\` \'`` 分别表示 buffer/string 的开始和结束  
-   ``\s-`` 或者 ``[[:space:]]`` 表示空白字符  
-   ``\w\{20,\}`` 表示长度大于等于20的单词  
-   ``[-+[:digit:]]`` 表示数字或者 “-” 或者 “+”  
-   ``\w+er\>`` 表示 er 结尾的单词  
-   ``\(19\|20\)[0-9]\{2\}`` 匹配 1900 ~ 2099


### 使用到正则表达式的命令合集

```plain text
 C-M-s                   incremental forward search matching regexp
 C-M-r                   incremental backward search matching regexp
 replace-regexp          replace string matching regexp
 query-replace-regexp    same, but query before each replacement
 align-regexp            align, using strings matching regexp as delimiters
 highlight-regexp        highlight strings matching regexp
 occur                   show lines containing a match
 multi-occur             show lines in all buffers containing a match
 how-many                count the number of strings matching regexp
 keep-lines              delete all lines except those containing matches
 flush-lines             delete lines containing matches
 grep                    call unix grep command and put result in a buffer
 lgrep                   user-friendly interface to the grep command
 rgrep                   recursive grep
 dired-do-copy-regexp    copy files with names matching regexp
 dired-do-rename-regexp  rename files matching regexp
 find-grep-dired         display files containing matches for regexp with Dired
```


## 括号相关  
  
-   ``C-M-u``  backward-up-list, 跳到上一个括号处  
-   ``C-M-d`` down-list，向内跳  
-   ``C-M-u C-M-SPC`` 选中整个括号区域  
-   ``C-M-n`` forward-list，跳到下一个括号处  
-   ``C-M-p`` backward-list，跳到上一个括号处


## Org-mode

``C-c C-, s``  ``org-insert-structure-template``  快捷插入代码块

``C-c <`` ``org-date-from-calendar`` 插入或更新日期


## 常规编辑功能


### 换位操作 Transpose  Objects

``C-t``  transpose-chars

``M-t``  transpose-words

``C-x C-t``  transpose-lines

Type ``M-x transpose SPACE`` to see more transposing commands.

``C-x DEL`` ``backward-kill-sentence`` 删除到行首


### 列编辑 Rectangle

``C-x r r``
:  Copy rectangle to region.

``C-x r i``
:  Insert region.

``C-x r k``
:  Kill the text of the region-rectangle, saving its contents as the “last killed rectangle” (kill-rectangle).

``C-x r d``
:  Delete the text of the region-rectangle (delete-rectangle).

``C-x r y``
:  Yank the last killed rectangle with its upper left corner at point (yank-rectangle).

``C-x r o``
:  Insert blank space to fill the space of the region-rectangle (open-rectangle). This pushes the previous contents of the region-rectangle rightward.

``C-x r c``
:  Clear the region-rectangle by replacing all of its contents with spaces (clear-rectangle).

``M-x delete-whitespace-rectangle``
:  Delete whitespace in each of the lines on the specified rectangle, starting from the left edge column of the rectangle.

``C-x r t string <RET>``
:  Replace rectangle contents with string on each line (string-rectangle).

``M-x string-insert-rectangle <RET> string <RET>``
:  Insert string on each line of the rectangle.


### 其他

``M-x untabify`` Convert all tabs in region to multiple spaces, preserving columns.

``C-x C-v`` Find alternate file, 也可以用来 reload file.


## Search

``C-s C-w`` search for the word after the current mark
``C-s C-y`` searches for the rest of the line after the current mark
``C-s C-M-y`` searches for the character after the mark

``M-%`` query-replace
``C-M-%`` query-replace-regexp

regexp notes:  
  
-   spaces: "\s-"


## Jump

``C-SPC C-SPC``
Set the mark, pushing it onto the mark ring, without activating it.

``C-u C-SPC``
Move point to where the mark was, and restore the mark from the ring of former marks.

``M-e`` isearch-edit-string, edit the search string in the minibuffer when isearch is activated.

``M-.`` ``xref-find-definitions`` 跳到方法、变量的定义处

``M-,`` ``xref-go-back`` 跳回去

``M-?`` ``xref-find-references`` 列出该符号的引用处

``C-c l`` org-store-link


## Bookmark

``C-x r m`` - set a bookmark at the current location (e.g. in a file)
``C-x r b`` - jump to a bookmark
``C-x r l`` - list your bookmarks
``M-x bookmark-delete`` - delete a bookmark by name


## Macro

``C-x (`` Start recording macro
``C-x )`` End recording macro
``C-x e`` Run macro


## Mode  
  
-   CC Mode
``C-c .`` c-set-style, switching CC Mode style  
-   ``fido-mode``  
      
    -   该模式开启时，在执行 ``make-directory`` 时，要创建的目录很容易匹配到文件名导致无法创建，这时可以输入文件名，并直接按下 ``M-j`` 即可立即创建该文件夹。  
-   ``hl-line-mode`` 高亮当前行


## 特殊字符  
  
-   **``C-x 8 R``** 输入 ®   
-    **``C-x 8 o``** 输入 °  
-    **``C-x 8 C-h``** 获得一份完整的列表


## 帮助文档  
  
-   **``C-h f``** 查看某个函数的文档  
-   **``C-h v``** 查看某个变量的文档  
-   **``C-h a``** 使用正则表达式来查找命令（**``M-x apropos``**可查找函数或变量）  
-   **``C-h k``** 查看快捷键绑定的命令  
-   **``C-h l``** 显示最近的 100 个键入动作  
-   **``C-h m``** 描述当前的 mode  
-   **``C-h i``** 查看 info 文档  
-   **``C-h C-h``** 获取完整列表