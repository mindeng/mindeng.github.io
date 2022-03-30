+++
title = "Emacs 的本地变量 (Local Variables)"
date = 2022-03-12T00:00:00+08:00
tags = ["emacs", "elisp", "tools"]
draft = false
+++

## Emacs 中的变量 {#emacs-中的变量}

要解释本地变量，先解释一下 Emacs 里面变量的作用。

Emacs 中有很多功能（内置的或者插件提供的）都可以通过设置一些变量的值来
进行一些个性化的定制。

这里举几个例子加以说明：

indent-tabs-mode
: 是否使用 tab来做缩进

fill-column
: 设置自动换行的长度，默认 70

org-download-image-dir
: 设置 org-download 插件用来存储图片的路径，
    默认为文档所在目录

org-confirm-babel-evaluate
: 设置 babel 执行代码时是否需要确认

一般情况下，我们可以通过在 `.emacs` 文件中对这些变量进行全局配置。但如
果你有进一步的诉求，例如希望针对某个项目（或者某个目录）有一些不同的定
制，或者甚至对某个文件进行单独的配置呢？

这时候 Local Variables 就派上用场了。

简单来说，你可以有两个选择：

directory-local variables
: 在目录中创建一个叫 `.dir-locals.el` 的文件，该目录下（以及子目录下）
    的所有文件都会应用该文件的配置。出于性能考虑，远程文件默认不开启父目
    录搜索功能。

file-local variables
: 可以在文件的第一行，或者文件末尾处定义，仅对该文件有效，具体如何定义
    后文有介绍。针对同一个变量，file-local 会覆盖 directory-local 的定义。


## directory-local variables 本地目录变量 {#directory-local-variables-本地目录变量}

除了 `.dir-locals.el` 文件，还可以定义额外的 `.dir-locals-2.el` 文件。
当 `.dir-locals.el` 文件在 git 仓库中作为共享文件时，可以通过这第二个
文件来进行一些本地化配置。

`.dir-locals.el` 文件示例：

```emacs-lisp
(
 ;; 这些配置会应用到任意模式（即所有文件）
 (nil . ((indent-tabs-mode . t)
	 (fill-column . 80)
	 (mode . auto-fill)))

 ;; 这些配置只会应用到 c-mode 的文件中
 (c-mode . ((c-file-style . "BSD")
		;; 下面这句指定该配置仅应用到当前目录，不应用到子目录
		(subdirs . nil)))

 ;; 这些配置只会应用到 org 文件中
 (org-mode . ((org-download-image-dir . "./images")))

 ;; 这些配置会应用到 src/imported 目录下的所有文件中
 ("src/imported"
  . ((nil . ((change-log-default-name
	  . "ChangeLog.local"))))))
```

另一种配置 directory-local variables 的方式，是定义一个 `directory
class`, 然后指定某个目录使用该 `directory class` 。举例说明：

```emacs-lisp
(dir-locals-set-class-variables 'unwritable-directory
   '((nil . ((some-useful-setting . value)))))

(dir-locals-set-directory-class
   "/usr/include/" 'unwritable-directory)
```

如果你不能直接在目录下创建 `.dir-locals.el` 文件的话，上面这种方式就很
有用。


## file-local variables 本地文件变量 {#file-local-variables-本地文件变量}

可以在文件的第一行定义，可以以注释的形式出现，例如 elisp 文件可以这么
定义：

```emacs-lisp
;; -*- mode: Lisp; fill-column: 75; comment-column: 50; -*-
```

org 文件指定 org-download 的图片保存路径：

```emacs-lisp
# -*- mode: Org; org-download-image-dir: "./images"; -*-
```

其中 `mode` 变量比较特殊，是用来指定 major mode 的，类似的特殊变量还有
`eval`, `coding` 等，后文有介绍。

如果是 shell 脚本，由于第一行用于指定脚本解释器，因此可以写在第二行。

如果不想手写，可以通过如下几个命令来自动编辑：

add-file-local-variable-prop-line
: 增加变量定义

delete-file-local-variable-prop-line
: 删除变量定义

copy-dir-locals-to-file-locals-prop-line
: 拷贝 directory-local
    variables 到第一行

另一种方式，是在文件的末尾附近（离末尾不超过3000行）定义 local
variables list:

```c
/* Local Variables:  */
/* mode: c           */
/* comment-column: 0 */
/* End:              */
```

如果你在文件中有类似的文本，但是不希望被当作 local variables list，可
以通过插入一个换页符（^L, 参考[Pages](https://www.gnu.org/software/emacs/manual/html_node/emacs/Pages.html)）来撤销该功能，因为 Emacs 只会在最
后一页查找 local variables list。


### 一些特殊变量 {#一些特殊变量}

mode
: 指定 major mode

eval
: 运行 lisp 表达式，返回值会被忽略

coding
: 指定文档的编码系统，详情参考 [Coding Systems](https://www.gnu.org/software/emacs/manual/html_node/emacs/Coding-Systems.html)

unibyte
: 是否采样 unibyte mode 加载或者编译 Emacs Lisp 文件，参考
    [Disabling Multibyte Characters](https://www.gnu.org/software/emacs/manual/html_node/elisp/Disabling-Multibyte.html#Disabling-Multibyte)

注意，请不要使用 `mode` 变量来指定 minor mode, 而应该使用 `eval` 运行
lisp 代码来启用或关闭 minor mode。例如下面的例子，会打开 eldoc-mode,
并关闭 font-lock-mode：

```emacs-lisp
;; Local Variables:
;; eval: (eldoc-mode)
;; eval: (font-lock-mode -1)
;; End:
```


最后，如果你通过命令做了一些乱七八糟的配置，想复原的话，可以通过 `M-x
normal-mode` 命令来根据文件名，以及文件内容，重置本地变量和 major mode
设置。


## 安全性问题 {#安全性问题}

由于加载本地变量存在一定的安全隐患，所以除非是一些 Emacs 认为是安全的
变量，否则 Emacs 将向你确认是否执行。

当然，你可以通过一些配置来指定你认为是安全的变量或者命令，详情请参考
[Safe-File-Variables](https://www.gnu.org/software/emacs/manual/html_node/emacs/Safe-File-Variables.html) 。