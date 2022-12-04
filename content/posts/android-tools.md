
---
title: "Android Tools"
date: 2022-06-27T06:19:00.000Z
lastmod: 2022-11-28T11:29:00.000Z
tags: ['tools', 'android']
draft: false
---



## 常用工具  
  
-   scrcpy：PC 端镜像手机端屏幕，可操作、截屏、录屏等  
-   codeLocator：实时抓取 viewTree  
-   [flipper](https://github.com/facebook/flipper)：A desktop debugging platform for mobile developers.


## adb 命令备忘

```sql
# 查看当前 Activity 是否使用了 SurfaceView。如果有 SurfaceView 字样，则表示使用了 SurfaceView
adb shell dumpsys SurfaceFlinger|grep 'HWC layers' -A10

# 查看最近的 crash 日志
adb logcat -b crash
```
