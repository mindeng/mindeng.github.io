+++
title = "动态绑定（Dynamic Binding）和词法绑定（Lexical Binding）"
date = 2023-06-26T14:42:00+08:00
lastmod = 2023-06-26T15:24:47+08:00
tags = ["elisp", "emacs"]
draft = false
+++

今天读了一篇讲 dynamic binding 和 lexical binding 的文章： [Dynamic Binding Vs
Lexical Binding](https://www.emacswiki.org/emacs/DynamicBindingVsLexicalBinding)，讲的挺清楚的，这里大致翻译如下。


## 绑定 binding 的概念 {#绑定-binding-的概念}

绑定是名字和值的一种对应关系。在 Lisp 中，可以用 `let` 来创建绑定：

```emacs-lisp
  (let ((a 1)) (print a))
  ;; ==> 1
```

这里将 name `a` 绑定到 value `1` 上。

`let` 表达式其实只是一个“语法糖”，和 `lambda` 表达式是等价的。例如：

```emacs-lisp
  (let ((a 1)
        (b 3))
    (+ a b))
```

等价于：

```emacs-lisp
  ((lambda (a b) (+ a b)) 1 3)
```

当然，除了 `let` 之外，还有很多其他的方法可以创建 bindings, 例如 `defconst`, `defun`,
`defvar`, `flet`, `labels`, `prog`, 等等。


## 动态绑定和词法绑定 {#动态绑定和词法绑定}

在处理变量绑定时，有两种方式：

dynamic
: 动态绑定，所有的变量名及它们的值都存在一张全局表中。

lexical
: 词法绑定，每个绑定作用域（binding scope），包括 defun/let 等，都会创建一张新的表，用于存放变量和值，这些表组织成一个层次结构，被称为 "the enviroment"。

在上面给出的那些简单的例子中，lexical 和 dynamic binding 之间并没有什么区别，返回的结果是一样的。

但是在一些复杂情况下，情况则有所不同。例如：

```emacs-lisp
  (let ((a 1))                            ; binding (1)
    (let ((f (lambda () (print a))))
      (let ((a 2))                        ; binding (2)
        (funcall f))))
  ;; dynamic binding ==> 2
  ;; lexical binding ==> 1
```

-   如果是 **lexical binding** ，在访问变量时，会在 **lexical enviroment** 中查找绑定，也就是说，在变量的代码块范围内查找。当在 **lexical enviroment** 中有多个绑定同时存在时，选择最内层的那个。

    因此，如果是 lexical binding，上述代码会打印 “1”，因为只有 binding (1) 在
    lexical enviroment 中。
-   如果是 **dynamic binding**, 在访问变量时只会在 **dynamic enviroment** 中查找，也就是说，在所有的绑定中查找，包括从程序启动之后创建的所有绑定（只要没被销毁）。如果同时存在多个绑定，则使用运行时最近创建的那个（我想这就是 **dynamic** 一词的由来）。

    因此，如果是 dynamic binding，上述代码会打印 “2”，因为当 `a` 求值时，binding
    (1) 和 binding (2) 都被创建了，但是 binding (2) 才是最近创建的。

    > 在多线程 Lisp 中，关于 dynamic binding 我们需要更加小心一点，因为要确保一个线程不会看到（访问到）另一个线程所创建的 bindings。由于 EmacsLisp 是单线程的，所以不用担心。


## 动态绑定的优点 {#动态绑定的优点}

动态绑定可以很方便的修改子系统的行为。

这里举一个例子。假设你有一个 `foo` 的函数，该函数会利用 `print` 产生一些输出，你希望可以将该输出捕获到一个 buffer 中。通过 dynamic binding，可以很轻松的实现：

```emacs-lisp
  ;; get-buffer-create 获取或创建一个指定名字的 buffer，注意名字前面有一个空格，表
  ;; 示该 buffer 不保留 undo 历史
  (let ((b (get-buffer-create " *string-output*")))
    ;; 修改 standard-output 变量，将标准输出重定向到 buffer b 中。注意：该修改仅在
    ;; 该 let 的作用域范围内生效
    (let ((standard-output b))
      ;; 该输出会重定向到 buffer b 中
      (print "foo"))

    ;; 切换当前 buffer 为 b，仅用于编辑，不会展示该 buffer
    (set-buffer b)
    (insert "bar")
    ;; 返回当前 buffer 的内容
    (buffer-string))
```

> 如果你经常使用类似的功能，你应该将其封装在一个 macro 中 —— 幸运的是，已经有这样的封装了： `with-output-to-temp-buffer` 。

由于 `foo` （这里其实是 `print` 函数）使用的 `standard-output` 是 _dynamic binding_ 的，因此你可以替换成你自己的绑定，以此来修改 `foo` 的行为 —— 以及 `foo` 所调用的所有
functions 的行为。

在一个不支持 dynamic binding 的语言中，你大概需要给 `foo` 增加一个可选参数来指定一个 buffer，然后 `foo` 需要传递该参数给所有的 `print` 调用。如果 `foo` 还调用了其他函数，并且这些函数也调用了 `print` ，那么你同样需要修改所有这些参数（注意：这是一个递归的过程）。

Richard Stallman 在 EmacsLisp 的上下文中解释了动态绑定的优点：[Emacs Paper - Dynamic
Binding](https://www.gnu.org/software/emacs/emacs-paper.html#SEC17) 。另请参阅 Pascal Costanza 写的文章 [Dynamic vs. Static Typing — A
Pattern-Based Analysis](http://www.p-cos.net/documents/dynatype.pdf) 。


## 词法绑定的优点 {#词法绑定的优点}

MilesBader 的这封邮件讲的很清楚，这里摘抄并翻译如下：

> <div class="verse">
>
> From: MilesBader<br />
> Subject: Re: Emacs 22<br />
> Newsgroups: comp.emacs<br />
> Date: Sun, 19 Aug 2001 01:47:53 GMT<br />
>
> </div>
>
> Because it's (1) much easier for the user [that is, programmer], because it
> eliminates the problem of which variables lambda-expressions use (when they
> attempt to use variables from their surrounding context), and (2) much easier
> for the compiler to optimize, because it doesn't need to worry about variables
> escaping their lexical context, and so doesn't need to allow for the possibility
> (this is a big problem with the current compiler).
>
> 因为它（1）对于程序员来讲，要简单很多，因为它消除了 lambda 表达式使用哪些变量的问题（当他们尝试使用外围环境的变量时），（2）对于编译器来说，优化起来要简单很多，因为它不需要担心变量从词法上下文中逃逸出去，因此不需要考虑这种逃逸出去的可能性（这是当前编译器的一个大问题）。


## 语言 {#语言}

大部分语言只支持 lexical binding 。

-   EmacsLisp 从 24.1 版本开始同时支持 dynamic binding 和 lexical binding 。
    Lexical binding 需要在一个文件或 buffer 中显式启用（见下文）。通过 `defvar` 定义的变量为“special”变量，即永远是动态绑定的（即使该文件启用了 lexical binding）。
-   CommonLisp 同时支持 dynamic binding 和 lexical binding 。 默认为 lexical
    binding，通过 `defvar` 或 `declare` 创建的一个变量即为“special”的动态绑定变量。


## 使用词法绑定 {#使用词法绑定}

要在 EmacsLisp 中使用词法绑定，需要在文件头中设置 file-local 变量
`lexical-binding` 为 `t` ，且必须在文件的第一行中设置：

```emacs-lisp
  ;;; -*- lexical-binding: t -*-
```
