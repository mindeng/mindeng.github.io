+++
title = "在 Emacs 中实现 VSCode 的 terminal 快捷键功能"
date = 2022-03-09T00:00:00+08:00
tags = ["emacs", "elisp", "terminal", "vscode", "tools"]
draft = false
+++

VSCode 有一个比较方便的快捷键：Ctrl-\` ，可以一键拉起 terminal 。

这里在 Emacs 中模拟一下这个功能，而且还是增强版本，可以在不同文件中拉
起不同的 terminal，拉起的 terminal 路径和当前文件的目录一致。

```emacs-lisp

;; emacs 自带的 term 命令只支持单个 terminal，因此该功能依赖
;; multi-term 。这里先检查有没有安装，没有的话先安装 multi-term
(unless (package-installed-p 'multi-term)
  (package-install 'multi-term))

;; 定义拉起 multi-term 的函数，如果当前目录已经拉起过 terminal，则直接
;; 跳转到该 terminal。由于我的屏幕比较宽，这里会自动将 terminal 分屏到
;; 右侧
(defun open-or-jump-to-multi-term ()
  (interactive)
  (if (string-prefix-p "*terminal<" (buffer-name))
  (delete-window)
	(progn
  (setq bufname (concat "*terminal<" (directory-file-name (file-name-directory (buffer-file-name))) ">"))
  (if (get-buffer-process bufname)
	(switch-to-buffer-other-window bufname)
  (progn

	(split-window-right)
	(other-window 1)
	(multi-term)
	(rename-buffer bufname)
	)
  )))
  )

;; 定义快捷键，和 VSCode 一致
(global-set-key (kbd "C-`") 'open-or-jump-to-multi-term)
```

Emacs 最大的魅力，也许就是可以按照自己的喜好，随心定制各种功能吧！另外
各种文本编辑的快捷键，也是越用越顺手。