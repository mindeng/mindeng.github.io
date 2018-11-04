---
layout: post
title:  "How to fix: Python version 2.7 requried, which wasn't found in the registry"
date:   2016-06-06
tags:   [python]
---

I encounter this problem when I install PIL-1.1.7 on my 64-bit Windows 7 platform.

Run the following command in administrator mode, and the problem fixed:

```
reg copy HKLM\SOFTWARE\Python HKLM\SOFTWARE\Wow6432Node\Python /s
```

Thanks to the first comment under the (http://stackoverflow.com/a/15802648)[answer] .
