---
layout: post
title:  "Create ramdisk on Mac OS X"
date:   2015-11-09
tags:   [os-x, ramdisk, tmpfs]
---

Creating a 1000MB ramdisk:

```
$ hdiutil attach -nomount ram://$((2048 * 1000))
/dev/disk3

$ diskutil eraseVolume HFS+ RAMDisk /dev/disk3
Started erase on disk3
Unmounting disk
Erasing
Initialized /dev/rdisk3 as a 1000 MB case-insensitive HFS Plus volume
Mounting disk
Finished erase on disk3 RAMDisk

$ hdiutil detach /dev/disk3
```
