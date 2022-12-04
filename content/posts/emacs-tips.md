
---
title: "Emacs Tips"
date: 2022-12-03T09:59:00.000Z
lastmod: 2022-12-04T12:36:00.000Z
tags: ['emacs', 'tools', 'editor']
draft: false
---



## 常规编辑功能


### 换位操作 Transpose  Objects

``C-t``  transpose-chars

``M-t``  transpose-words

``C-x C-t``  transpose-lines

Type ``M-x transpose SPACE`` to see more transposing commands.


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


## Mark

``C-c l`` mark-line


## Jump

``C-SPC C-SPC``
Set the mark, pushing it onto the mark ring, without activating it.

``C-u C-SPC``
Move point to where the mark was, and restore the mark from the ring of former marks.

``M-e`` isearch-edit-string, edit the search string in the minibuffer when isearch is activated.


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
-   Edit
