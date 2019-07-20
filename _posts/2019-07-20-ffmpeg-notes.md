---
layout: post
title:  "ffmpeg notes"
date:   2019-07-20
tags:   [ffmpeg, ffplay]
---

# How to complie `ffplay` on macOS

先克隆代码，并安装依赖库 `sdl2`。如果没有安装 `sdl2` 则无法构建出 `ffplay`:

```sh
git clone https://git.ffmpeg.org/ffmpeg.git && cd ffmpeg
brew install sdl2
```

然后在`ffmpeg`根目录下运行命令：

    ./configure --enable-ffplay | grep "Programs:" -A1

确保输出里面有 `ffplay`:

    Programs:
    ffmpeg                  ffplay                  ffprobe

最后运行：

    make && make install

成功！

