+++
title = "常用命令行工具"
date = 2022-02-22T00:00:00+08:00
tags = ["tools", "command", "scripts", "linux", "macos", "perf"]
draft = false
+++

记录一些常用的命令行工具，方便随时取用。


## 查看 CPU 总核心数 {#查看-cpu-总核心数}

```sh
grep -c 'model name' /proc/cpuinfo
```


## find printable strings 查找二进制文件中的字符串 {#find-printable-strings-查找二进制文件中的字符串}

例如查看 /bin 目录下的程序可能会创建哪些临时文件:

```sh
strings /bin/* | grep tmp
```


## 查看、配置资源限制 {#查看-配置资源限制}

```sh
ulimit -a
```

```text
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
file size               (blocks, -f) unlimited
max locked memory       (kbytes, -l) unlimited
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 1
stack size              (kbytes, -s) 9788
cpu time               (seconds, -t) unlimited
max user processes              (-u) 2784
virtual memory          (kbytes, -v) unlimited
```


## 测试硬盘速度 {#测试硬盘速度}

```sh
# bs 默认为 512 字节，单位支持 k/m （macOS 平台如此，Linux 上不一样）
# write
dd of=${path_on_disk} if=/dev/zero bs=1m count=1000
# read
dd if=${path_on_disk} of=/dev/null bs=1m
```

```text
1000+0 records in
1000+0 records out
1048576000 bytes transferred in 0.619836 secs (1691698844 bytes/sec)
1000+0 records in
1000+0 records out
1048576000 bytes transferred in 0.246972 secs (4245726816 bytes/sec)
```


## 监控磁盘 IO 情况 {#监控磁盘-io-情况}

```sh
iostat
```

```text
              disk0               disk2               disk3       cpu    load average
    KB/t  tps  MB/s     KB/t  tps  MB/s     KB/t  tps  MB/s  us sy id   1m   5m   15m
   26.18   95  2.42    25.87    0  0.00   287.74    0  0.00   4  2 94  1.95 1.64 1.56
```


## 下载脚本并立即执行 {#下载脚本并立即执行}

以下命令会自动安装 yarn 包管理工具：

```sh
curl --compressed -o- -L https://yarnpkg.com/install.sh | bash
```

参数解释：

--compressed
: 请求压缩方式传输，节省流量、提高速度

-o-
: -o 指定保存的文件路径，参数指定为 `-` 则强制输出到 stdout

-L
: 跟随重定向