
---
title: "Create Ramdisk on macOS"
date: 2022-12-04T12:17:00.000Z
lastmod: 2022-12-04T12:18:00.000Z
tags: ['macos', 'tools']
draft: false
---


Creating a 1000MB ramdisk:

```bash
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
