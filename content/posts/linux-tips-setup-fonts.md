
---
title: "Linux Tips: 安装新字体"
date: 2022-12-03T10:15:00.000Z
lastmod: 2022-12-04T13:09:00.000Z
tags: ['tools', 'linux']
draft: false
---


Debian 上安装字体，就这么简单：

```bash
mkdir ~/.fonts
cp simsun.ttf ~/.fonts/
sudo fc-cache -fv
```

搞定！优雅～