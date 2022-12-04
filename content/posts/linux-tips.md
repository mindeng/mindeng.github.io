
---
title: "Linux Tips"
date: 2022-01-23T04:24:00.000Z
lastmod: 2022-12-04T13:08:00.000Z
tags: ['tools', 'linux']
draft: false
---


记录一些常用的命令行工具，方便随时取用。


# Network

[Network Tools 网络工具箱]({{< relref "network-tools" >}}) 。


## xargs

xargs on Mac OS：

``find . -iname ``*``something``*`` | xargs -I {} mv {} /dest/path``


## Crontab

```shell
# 每 10 分钟运行一次
*/10 * * * * command

# 开机自启动：每次系统重启时，都会运行 ss.sh
@reboot ss.sh
```

参考：[https://crontab.guru/every-10-minutes](https://crontab.guru/every-10-minutes)


## 二进制工具

二进制 dump 工具  
  
-   ``xxd``
以十六进制形式 dump 文件内容  
-   ``xxd -b``
以二进制形式 dump 文件内容  
-   ``xxd -r``
从 dump 内容还原出原始文件，例如 ``xxd file | xxd -r`` 和
``cat file`` 的输出是一致的


查找二进制文件中的字符串（find printable strings），例如：

```bash
# 查看 /bin 目录下的程序可能会创建哪些临时文件
strings /bin/* | grep tmp
```


## 代理相关

如果你的 *http_proxy* 需要登录，可以设置如下：

``export http_proxy=http://username:password@host:port``


## 过滤、报告相同行  
  
-   ``uniq``    
    
    相同行仅打印一次  
-   ``uniq -c``
行首插入该行重复出现的次数  
-   ``uniq -d``
仅输出相同行  
-   ``uniq -u``
仅输出不同行


## 系统管理相关

``script /dev/null``
该命令可以解决 su 切换用户后，运行 screen 程序时报错：

```plain text
Cannot open your terminal '/dev/pts/0' - please check.
```

的问题。

原理是 `script` 命令会开启一个新的伪终端（pseudo-terminal），用来 dump 终端上的所有输出。

```shell
# 查看 CPU 总核心数
grep -c 'model name' /proc/cpuinfo

# 查看、配置资源限制
ulimit -a
```


### 测试硬盘速度

```bash
# bs 默认为 512 字节，单位支持 k/m （macOS 平台如此，Linux 上不一样）
# write
dd of=${path_on_disk} if=/dev/zero bs=1m count=1000
# read
dd if=${path_on_disk} of=/dev/null bs=1m

```


### 监控磁盘 IO 情况

```bash
iostat

```

```plain text
              disk0               disk2               disk3       cpu    load average
    KB/t  tps  MB/s     KB/t  tps  MB/s     KB/t  tps  MB/s  us sy id   1m   5m   15m
   26.18   95  2.42    25.87    0  0.00   287.74    0  0.00   4  2 94  1.95 1.64 1.56

```


### 文件权限

![](/uploads/images/25e5f918-7808-4a5d-805f-2bfb353353b6/Untitled.png)

```bash
# 更改文件属组
chgrp [-R] group filename

# 更改文件属主
chown [-R] owner filename
chown [-R] owner:group filename

# 更改文件属性
chmod u=rwx,g=rx,o=r  test1
chmod a-x test1
```

SUID, SGID, sticky bit

```bash
# 为程序设置 SGID, 程序将以文件所属组的权限运行
chmod g+s executable

# 为程序设置 SUID, 程序将以文件属主的权限运行
chmod u+s executable

# 为目录设置 SGID, 该目录下新建的文件或目录的组将默认为该目录的组
chmod g+s directory

# 设置 sticky bit, 该目录下新建的文件或目录不能被非owner删除（该目录的属主是个例外，可以删除任意文件）
chmod +t directory
```

SUID/SGID 提权

![](/uploads/images/c1a662fe-36db-4c41-b98f-4c554cbc8cf4/Untitled.png)

SUID (Set UID)是Linux中的一种特殊权限，其功能为用户运行某个程序时，如果该程序有SUID权限，那么程序运行为进程时，进程的属主不是发起者，而是程序文件所属的属主。但是SUID权限的设置只针对二进制可执行文件,对于非可执行文件设置SUID没有任何意义.

 在执行过程中，调用者会暂时获得该文件的所有者权限,且该权限只在程序执行的过程中有效. 通俗的来讲,假设我们现在有一个可执行文件``ls``,其属主为root,当我们通过非root用户登录时,如果``ls``设置了SUID权限,我们可在非root用户下运行该二进制可执行文件,在执行文件时,该进程的权限将为root权限.

```bash
# 设置SUID位
chmod u+s filename   

# 去掉SUID设置
chmod u-s filename   
```


以下命令可以找到正在系统上运行的所有SUID可执行文件。准确的说，这个命令将从/目录中查找具有SUID权限位且属主为root的文件并输出它们，然后将所有错误重定向到/dev/null，从而仅列出该用户具有访问权限的那些二进制文件。

```shell
# 列出所有 SUID 文件
find / -user root -perm -4000 -print 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
find / -user root -perm -4000 -exec ls -ldb {} ;

# 列出所有 SGID 文件
find / -type f -perm -04000 -ls

# 列出所有全局可写文件
find / -perm -2 -type f -print
```



### 文件时间戳  
  
-   atime  
      
    -   Access Time      
    -   出于性能考虑，一般 mount 的时候会增加如下选项，导致 atime 表现和预期不一致：  
              
            -   ``noatime`` Do not update the file access time when reading from a file          
            -   ``relatime``  Update inode access times relative to  modify  or  change  time. Access time is only updated if the previous access time was earlier than the current modify or change time.      
    -   ``ls -lu`` 列出 atime， ``ls -ltu`` 按 atime 排序（最近的排最上面）  
-   mtime  
      
    -   Modify Time，最后一次修改文件（内容）或者目录（内容）的时间      
    -   ``ls -l`` ``ls -lt`` 默认使用 mtime      
    -   ``cp`` 文件默认会改变 mtime + ctime      
    -   ``cp -a`` 只会改变 ctime，mtime 不变  
-   ctime  
      
    -   Change Time，最后一次改变文件（属性或权限）或者目录（属性或权限）的时间      
    -   ``ls -lc`` ``ls -ltc`` 使用 ctime    
    

```shell
stat filename

# sort by mtime
ls -lt

# sort by ctime
ls -ltc

ls -alR > /path/to/timestamp_modification.txt
ls -alRc > /path/to/timestamp_creation.txt
```



## 文件打洞

如何在一个大文件（例如 >500G）的开头，高效的插入一些内容？

如果是 Linuxe + ext4/XFS，可以考虑用 fallocate 函数，FALLOC_FL_INSERT_RANGE 标记位。

参考：[https://stackoverflow.com/questions/37882286/prepend-to-very-large-file-in-fixed-time-or-very-fast/37884191#37884191](https://stackoverflow.com/questions/37882286/prepend-to-very-large-file-in-fixed-time-or-very-fast/37884191#37884191)


```bash

# 子目录按占用空间大小排序
du -d 1 |sort -n

# 查看 CPU 总核心数
grep -c 'model name' /proc/cpuinfo

# 在 /bin 或者 /usr/sbin 中运行，搜索哪些程序会创建临时文件
strings * | grep tmp

# 查看资源限制的设置
ulimit -a

# 测试硬盘速度
# bs 默认为 512 字节，单位支持 k/m （macOS 上，Linux 不一样）
dd of=/Volumes/disk-1t/1.out if=/dev/zero bs=1m count=10000
dd if=/Volumes/disk-1t/1.out of=/dev/null bs=1m

# 监控磁盘 io 情况，-p 指定 某个（或某些）磁盘
iostat -x 1

# 显示当前时间配置信息
timedatectl show
# 将本地时间同步到 rtc 时钟，解决 Windows 10 的系统时间不对的问题（会修改配置文件/etc/adjtime）
timedatectl set-local-rtc 1

# lazy umount
umount -l

# 下载脚本并立即执行
curl --compressed -o- -L https://yarnpkg.com/install.sh | bash

# 文件恢复到某个 revision
git checkout c5f567 -- file1/to/restore file2/to/restore

# 文件恢复到某个 revision 之前的 1 个版本
git checkout c5f567~1 -- file1/to/restore file2/to/restore

pgrep process-name
pkill process-name
```



## loopback device

loopback device 可以将文件作为设备来挂载，类似于虚拟机的镜像文件。

```bash
# 创建一个稀疏文件（sparse file）
dd if=/dev/zero of=1.img bs=1 count=0 seek=512M

# 将 1.img 自动绑定到一个可用的 loop device
# -P, --partscan：扫描分区
sudo losetup -fP 1.img

# 列出当前的 loop devices
losetup

# 分区
sudo fdisk /dev/loop13
# 分区之后可以看到分区的设备号，例如这里为 /dev/loop13p1

# 将分区格式化为 ext4 文件系统
sudo mkfs.ext4 /dev/loop13p1

# 挂载 loop device
sudo mount /dev/loop13p1 /mount/point

# 删除 loopback device
sudo umount /mount/point
sudo losetup -d /dev/loop13p
```


稀疏文件（sparse file) 的特点:

```bash
$ dd if=/dev/zero of=sparse_file bs=1 count=0 seek=512M
0+0 records in
0+0 records out
0 bytes copied, 0.000138743 s, 0.0 kB/s

$ ls -lh sparse_file 
-rw-rw-r-- 1 user user 512M 2月   2 22:36 sparse_file

$ du -sh sparse_file 
0       sparse_file

# -s：查看实际分配的尺寸
$ ls -lhs sparse_file
0 -rw-r--r-- 1 user user 512M 2月   2 22:36 sparse_file
```

如果将 sparse file 作为镜像文件挂载，会随着在设备中文件的写入，镜像文件的尺寸才会逐渐增大。


## 常用 Console 快捷键


|快捷键|动作|
| :---: | :---: |
|ctrl+l|清屏|
|ctrl+d|退出shell|
|ctrl+u|清除光标之前|
|ctrl+k|清除光标之后|
|ctrl+w|清除光标之前的一个单词|
|ctrl+y|粘贴刚才ctrl+u/k/w的内容|
|ctrl+t|交换最后两个字符|
|esc+k|交换最后两个单词|
|alt+f/ctrl+b|向前/后移动一个单词|
|ctrl+f/ctrl+b|向前/后移动一个字符|
|alt+d/ctrl+d|删除光标后的一个单词/字符|
|ctrl-n/ctrl-p|上一个/下一个命令|
|ctrl-/|undo|
|ctrl-r|reverse-i-search|
|!|history substitution|
|!n|Refer to command line n.|
|!-n|Refer to the current command line minus n.|
|!!|Refer to the previous command.  This is a synonym for \`!-1'.|