+++
title = "理解 ISO 基本媒体文件格式 (ISOBMFF)"
date = 2024-03-23T20:12:00+08:00
lastmod = 2024-03-26T11:46:35+08:00
tags = ["parser", "multimedia", "rust"]
draft = false
+++

前段时间用 Rust 写了一个 Exif/Metadata 解析库 [nom-exif](https://crates.io/crates/nom-exif)，里面涉及到对 ISOBMFF 的解析，趁着还有点印象，总结一下这种文件格式。

ISOBMFF 英文全称 _ISO Base Media File Format_ ，顾名思义主要用于封装多媒体文件。
ISOBMFF 最初直接基于 Apple 的 QuickTime 容器格式，然后由 MPEG 开发并标准化为
[ISO/IEC 14496-12](https://www.iso.org/standard/83102.html) 。

该格式已广泛用于媒体文件存储，并作为各种其他媒体文件格式（例如 MP4 和 3GP 容器格式）的基础。


## 哪些文件类型在使用 ISOBMFF? {#哪些文件类型在使用-isobmff}

下面这个表格列举了常见的使用了 ISOBMFF 格式的文件：

| 文件类型    | 互联网媒体类型（MIME） | 常用扩展名                                            |
|---------|---------------|--------------------------------------------------|
| QuickTime   | video/quicktime        | .mov, .movie, .qt                                     |
| HEIF        | image/heif, image/heic | .heif, .heifs; .heic, .heics; .avci, .avcs; .HIF      |
| MP4         | video/mp4, audio/mp4   | .mp4, .m4a, .m4p, .m4b, .m4r, .m4v                    |
| 3GP         | video/3gpp             | .3gp, 3g2                                             |
| JPEG 2000   | image/jp2, image/jpx   | .jp2, .j2k, .jpf, .jpm, .jpg2, .j2c, .jpc, .jpx, .mj2 |
| Flash Video | video/x-flv            | .flv, .fla, .f4v, .f4a, .f4b, .f4p                    |


## ISOBMFF 文件结构概述 {#isobmff-文件结构概述}

ISOBMFF 文件由 Box (也叫 Atom) 组成，并且每个 Box 内部都可以任意嵌套，组成一棵
Box 树。顶层 Box 可以有多个。

一个典型的 ISOBMFF 文件结构如下图所示（以 MP4 文件为例）：

{{< figure src="/ox-hugo/isobmff-mp4.png" link="/ox-hugo/isobmff-mp4.png" >}}

如上图所示，一个正常的 ISOBMFF 文件，第一个顶层 Box 一般是 type 为 "ftyp" 的
Box&nbsp;[^fn:1]。通过解析 ftyp Box, 我们可以：

-   检测该文件是否是 ISOBMFF 格式。
-   识别该文件的具体文件类型。例如是 QuickTime, 或者 HEIF 等。

具体怎么做我们在[ _ftyp_ Box ](#ftyp-box)一节中介绍。

除了约定第一个 Box 是 ftyp Box, 后面的 Box 就没有任何限制了，不同的文件类型都可能不同。


### 基本 Box 结构 {#基本-box-结构}

下面是一个最简单的 Box 结构：

| bytes        | description   | example        |
|--------------|---------------|----------------|
| 4            | box_size, 含自身 |                |
| 4            | box_type      | "ftyp", "meta" |
| box_size - 8 | box_data      |                |


### _ftyp_ Box {#ftyp-box}

对于 ftyp Box 来说，box_data 里面存放的就是该文件的文件类型信息，其结构如下表所示：

| bytes            | description                            | example                         |
|------------------|----------------------------------------|---------------------------------|
| 4                | Major brand, 文件格式代码              | "qt  ": QuickTime, "heic": HEIC |
| 4                | Minor version, 文件格式规范版本        | 目前很少使用                    |
| box_size - 8 - 8 | Compatible brands, 兼容的文件格式，四字节一个，可能有多个 | "mif1MiHEmiafMiHBheic"          |

有了上述 ftyp box_data 的结构，我们就可以很容易对 ftyp Box 进行解析，从而判断该文件是否是 ISOBMFF, 以及识别出具体的文件类型。


## 几种特殊的 Box 类型 {#几种特殊的-box-类型}

除了[基本的 Box 结构](#基本-box-结构)，还有几种比较特殊的 Box 类型：

-   size 为 1 的 Box (扩展尺寸 Box)
-   size 为 0 的 Box
-   _wide_ Box

下面我们逐一介绍。


### size 为 1 的 Box (扩展尺寸 Box) {#size-为-1-的-box--扩展尺寸-box}

如果 box_size == 1, 则表示 box_size 超过 4 字节上限，Box 结构变成：

| bytes              | description   | example        |
|--------------------|---------------|----------------|
| 4                  | box_size == 1 |                |
| 4                  | box_type      | "ftyp", "meta" |
| 8                  | extended_size |                |
| extended_size - 16 | box_data      |                |

从上面的结构中可以看到，实际的 Box 长度变成了 extended_size, 长度的存储空间扩展为 8 字节。


### size 为 0 的 Box {#size-为-0-的-box}

如果 box_size == 0, 并不意味着该 Box 长度为 0。这种 Box 只能是顶层 Box, 并且只能是最后一个 Box, 其长度一直延伸至文件末尾。其结构如下所示：

| bytes | description           | example        |
|-------|-----------------------|----------------|
| 4     | box_size == 0         |                |
| 4     | box_type              | "ftyp", "meta" |
| \*    | box_data, 长度一直延伸到文件末尾 |                |


### _wide_ Box {#wide-box}

_wide_ Box 比较有意思，其结构如下：

| bytes | description        | example |
|-------|--------------------|---------|
| 4     | box_size == 8      |         |
| 4     | box_type == "wide" | "wide"  |

可以看到，wide Box 的长度固定为 8, box_type 固定为 "wide", 且没有 box_data。

wide box 的作用是为了预留出 8 个字节的空间，这个 8 字节空间可能用来干嘛呢？答案就在名字上 —— "wide", 顾名思义，是用于扩展 Box 长度的。下面我们具体解释一下。

wide Box 后面紧挨着的 Box (命名为 Box B 吧), 一般是一个普通的 size 为 4 字节的
Box。如果将来某一天，B 的长度不够，需要括容，则可以按照下面的步骤操作：

1.  将 wide Box 的 size 改成 1
2.  将 wide Box 的 type 修改为 B 的 type
3.  将 B 的 size 填充到 B 的头 8 个字节（即 size + type 部分）

这样，就完成了 Box B 从 4 字节长度到 8 字节长度的原地括容。这种方式的好处是，B
的 data 部分的 offset 可以保持不变。


## Full Box {#full-box}

有些 Box 为了进一步控制其 data 部分的结构和处理方式，会在 size/type 后面增加四个字节，其中一个字节为 version, 三个字节为 flags, 这类 Box 我们称为 Full Box。

Full Box 结构示意：

| bytes         | description | example        |
|---------------|-------------|----------------|
| 4             | box_size    |                |
| 4             | box_type    | "ftyp", "meta" |
| 1             | version     |                |
| 3             | flags       |                |
| box_size - 12 | box_data    |                |

version 和 flags 的含义根据 Box 类型的不同而不同。

除了增加了 version 和 flags 字段之外，Full Box 和普通 Box 一样，例如 size 为 1
时也表示该 Box 是一个扩展尺寸的 Box。

扩展尺寸 Full Box 的结构示意：

| bytes              | description   | example        |
|--------------------|---------------|----------------|
| 4                  | box_size == 1 |                |
| 4                  | box_type      | "ftyp", "meta" |
| 8                  | extended_size |                |
| 1                  | version       |                |
| 3                  | flags         |                |
| extended_size - 20 | box_data      |                |


## 在 HEIC/HEIF 文件中查找 Exif 信息 {#在-heic-heif-文件中查找-exif-信息}

本节介绍一下在 HEIC/HEIF 中定位 Exif 信息的过程，算是对上述知识点的一个综合应用吧。

先看一下一个典型的 HEIC/HEIF 文件的结构示意图：

{{< figure src="/ox-hugo/isobmff-heic.png" link="/ox-hugo/isobmff-heic.png" >}}

由于 Box 的嵌套特性，每个顶层 Box 都可以视为一棵 Box 树。因此，我们可以采用类似文件路径的方式，来标识某个 Box, 路径名即为 Box type。例如，在上图中，我们可以用
`/meta/iinf` 表示顶层 Box `meta` 下面的 `iinf` Box。

有了上述背景信息，我们可以按照下面这个步骤来查找 Exif 信息：

1.  查找 Exif `infe`

    `infe` Box 的路径是 `/meta/iinf/infe`, 但 `infe` Box 不止一个， `iinf` 下面挂着一系列的 `infe`, 我们需要找的是 Exif `infe`, 因此需要对 `infe` list 进行遍历查找。

    `infe` 的解析过程有点复杂，这里不展开讨论，这里只提两个字段，一个是 `item_id`, 另一个是 `item_type` 。我们在遍历 `infe` list 时, 需要找到 `item_type == "Exif"` 的
    `infe`, 然后记录其 `item_id` 。

2.  解析 Exif 的偏移和长度信息

    得到 Exif `infe` 的 `item_id` 之后，可以根据该信息在 `iloc` Box 中获取到 Exif 的偏移和长度信息。其过程如下：

    1.  找到 `/meta/iloc` Box。
    2.  `iloc` 的 data 部分包含一系列的 `ItemLocation`, 需要根据 `item_id` 找到 Exif 所对应的 `ItemLocation` 。
    3.  解析 Exif `ItemLocation`, 从中提取出 Exif 偏移和长度信息。

3.  根据 Exif 偏移和长度信息，获取 Exif 数据

当然，上述步骤是一个简化描述，实际情况会复杂很多，其中不少 Box 都涉及
version/flags 的复杂的行为控制，以及嵌套数据结构的处理，更多细节可参考 [nom-exif
的源码实现](https://github.com/mindeng/nom-exif/blob/5c4220ebbf4965a3a25e8031a9036f92b62d57b2/src/heif.rs#L50)。


## 参考资料 {#参考资料}

-   [ISO base media file format - Wikipedia](https://en.wikipedia.org/wiki/ISO_base_media_file_format)
-   [QuickTime File Format](https://developer.apple.com/documentation/quicktime-file-format)
-   [GitHub - mindeng/nom-exif](https://github.com/mindeng/nom-exif)
-   [mp4box.js - file inspection](https://gpac.github.io/mp4box.js/test/filereader.html)

[^fn:1]: 这里其实有一个特例，从 iOS 实况图片中导出的 .mov 视频文件，第一个 Box 不是 ftyp, 需要稍微注意一下。
