
---
title: "Android Tools"
date: 2022-06-27T06:19:00.000Z
lastmod: 2023-02-05T10:44:00.000Z
tags: ['tools', 'android']
draft: false
---



## 常用工具  
  
-   scrcpy：PC 端镜像手机端屏幕，可操作、截屏、录屏等  
-   UI 分析工具  
      
    -   codeLocator：实时抓取 viewTree      
    -   Android Studio：Android Device Monitor      
    -   UIAutomatorViewer      
    -   Appium Inspector      
    -   Espresso      
    -   adb  
-   [flipper](https://github.com/facebook/flipper)：A desktop debugging platform for mobile developers.


## adb 命令备忘

```sql
# 查看当前 Activity 是否使用了 SurfaceView。如果有 SurfaceView 字样，则表示使用了 SurfaceView
adb shell dumpsys SurfaceFlinger|grep 'HWC layers' -A10

# 查看最近的 crash 日志
adb logcat -b crash

# 查看当前 Activity
adb shell dumpsys activity top | grep ACTIVITY

# 查看当前获得焦点的 Window 所在的 Activity
adb shell dumpsys window | grep mCurrentFocus

# 通过 Action 隐式拉起页面，例如通过 schema 拉起页面
adb shell am start -a android.intent.action.VIEW -d 'kwai://live/play/RZA8aukYCco'

# 通过 包名/组件名 显示拉起页面
adb shell am start -n com.android.camera/com.android.camera.Camera
adb shell am start -n com.android.browser/com.android.browser.BrowserActivity

```
