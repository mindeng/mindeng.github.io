
---
title: "Linux Tips: 只读文件 Immutable files"
date: 2022-12-03T10:28:00.000Z
lastmod: 2022-12-04T13:09:00.000Z
tags: ['linux', 'tools']
draft: false
---
  
  
-   查看文件属性    
    
    ```bash
    $ lsattr filename
    ----i--------  filename
    
    ```    
    
    字母 ``i`` 意味着该文件是一个 Immutable （只读）文件。  
-   设置只读属性：    
    
    ```bash
    $ chattr +i filename
    
    ```  
-   去掉只读属性：    
    
    ```plain text
    $ chattr -i filename
    ```    
    