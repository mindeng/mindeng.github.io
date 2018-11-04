---
layout: post
title:  "Emacs Notes"
date:   2016-07-11
tags:   [emacs]
---

* Search

`C-s C-w` search for the word after the current mark
`C-s C-y` searches for the rest of the line after the current mark 
`C-s C-M-y` searches for the character after the mark

`M-%` query-replace
`C-M-%` query-replace-regexp

regexp notes:
spaces: "\s-"

* mark

`C-c l` mark-line

* Jump

`C-SPC C-SPC`
Set the mark, pushing it onto the mark ring, without activating it.

`C-u C-SPC`
Move point to where the mark was, and restore the mark from the ring of former marks.

`M-e` isearch-edit-string, edit the search string in the minibuffer when isearch is activated.

* Macro

`C-x (` Start recording macro
`C-x )` End recording macro
`C-x e` Run macro

* Reload file

`C-x C-v` Find alternate file

* CC Mode
`C-c .` c-set-style, switching CC Mode style
