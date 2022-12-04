
---
title: "Security Tips：ELF 破解及几点启示"
date: 2022-12-04T11:44:00.000Z
lastmod: 2022-12-04T13:02:00.000Z
tags: ['security']
draft: false
---


如何破解一个 ELF 文件：[hackme: Deconstructing an ELF File](http://manoharvanga.com/hackme/)

几点启示：  
  
-   编译时要使用 strip 选项  
-   避免在代码中使用字符串常量存放敏感信息，``strings`` 工具可以轻易 dump 出来  
-   在调用库函数、系统函数时，避免在参数中传递明文的敏感信息， ``ltrace``, ``strace``  等工具可以轻易调试出来


另外，在Java代码中，也要避免使用明文字符串保存敏感信息，尤其是不要用 ``String`` 来保存密码，原因主要有以下几点：  
  
-   String 在 java 中是不可变的，而且会一直保留在内存中，直到垃圾收集器将其回收。而由于字符串被放在字符串缓冲池中以方便重复使用，所以它就可能在内存中被保留很长时间；  
-   如果使用 char [] 来保存密码，在用完之后可以立即将其中所有的元素清空。所以将密码保存到字符数组中很明显的降低了密码被窃取的风险；

参考 [为什么使用字符数组保存密码比使用String保存密码更好？](http://my.oschina.net/jasonultimate/blog/166968)。