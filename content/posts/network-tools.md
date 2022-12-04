
---
title: "Network Tools 网络工具箱"
date: 2022-10-07T11:49:00.000Z
lastmod: 2022-12-04T12:22:00.000Z
tags: ['tools', 'network', 'linux']
draft: false
---



## 网络端口

```shell
# 查看网络端口监听情况
netstat -tunlp

# 查看 TCP 监听端口
sudo lsof -iTCP -sTCP:LISTEN -P -n

# 查看 UDP 监听端口
sudo lsof -iUDP -P -n

# 查看端口 55555 监听情况（不论 TCP/UDP）
sudo lsof -i:55555

# 查看某个进程监听的 IP/Port
ss -nlput | grep ssh

-n
    no port to name resolution
-l
    only listening sockets
-p
    show processes listening
-u
    show udp sockets
-t
    show tcp sotckets
```


## CURL

```shell
# post data
curl -d postthis localhost:8888

# post with empty data
curl -X POST localhost:8888

# add header
curl -H "Authorization: 484995" localhost:8888
```

下载脚本并执行：

```bash
# 下载并自动安装 yarn 包管理工具
curl --compressed -o- -L <https://yarnpkg.com/install.sh> | bash

```

参数解释：  
  
-   ``-compressed``
请求压缩方式传输，节省流量、提高速度  
-   ``-o-``
-o 指定保存的文件路径，参数指定为 ``-`` 则强制输出到 stdout  
-   ``-L``
跟随重定向


批量下载

```sql
#curl -I https://js.a.kspkg.com/bs2/fes/kuaiying-anzhi--4.0.0.400006_d0c756.apk
#curl --header "Range: bytes=89155500-89155509" https://js.a.kspkg.com/bs2/fes/kuaiying-anzhi--4.0.0.400006_d0c756.apk

urls=(
https://js.a.kspkg.com/bs2/fes/kuaiying-oppo--4.0.0.400006_16656c.apk
https://js.a.kspkg.com/bs2/fes/kuaiying-anzhi--4.0.0.400006_d0c756.apk
https://js.a.kspkg.com/bs2/fes/kuaiying-kw1--4.0.0.400006_e81108.apk
)

for i in "${urls[@]}"; do
    curl --header "Range: bytes=89155500-89155509" $i > /dev/null || { echo 'failed' ; exit 1; }
done
```


## SSH

更多 ssh 用法见：[ssh 常见用法介绍]({{< relref "ssh-usage" >}}) 


### AUTOSSH

```shell
# 监控 ssh 连接，断开时自动重连
autossh -M 4444 -NfL 9443:localhost:9443 xjp
```


### 本地端口转发

```shell
# 本地端口转发
# 通过 ssh 登陆 server 作为跳板，把 target.server 的 80 端口映射到本机的 8888 端口
# localhost:8888 <-> server:22 <-> target.server:80
ssh -NfL 8888:target.server:80 server

# 通过 ssh 转发访问 squid 代理服务
ssh -CNf -L 7777:localhost:3128 -l username squid.server
```


### OpenSSL 用于 base64 编解码

```bash
openssl enc -base64 -in myfile -out myfile.b64
openssl enc -d -base64 -in myfile.b64 -out myfile.decrypt
```


### Get ssh server key fingerprint

```shell
# 获取本机的 ssh 服务的 key 指纹
ssh-keyscan localhost | ssh-keygen -lf -

# 或者：
ssh-keygen -lf <(ssh-keyscan localhost 2>/dev/null)
```


### sshfs

```shell
sshfs nas:/mnt nas
```

参考：[https://www.itsfullofstars.de/2022/03/mount-a-remote-directory-via-ssh-on-macos-sshfs/](https://www.itsfullofstars.de/2022/03/mount-a-remote-directory-via-ssh-on-macos-sshfs/)


### VNC throught ssh 通过 ssh 转发连接 Mac 远程桌面

```shell
# VNC 默认监听 5900 端口
ssh -NfL 5910:172.22.22.10:5900 $(gethomeip) -p 52222
```

Finder → 前往 → 连接服务器：``vnc://localhost:5910``


### ssh + polipo

polipo 是一个轻量级的 http 代理服务器，且可以将 SOCKS 代理转成 http 代理。

结合上述两点，ssh 搭配 polipo 可以实现一个简易的本地 http 代理服务。


## NC

```shell
# 扫描端口 22 是否已打开
# -z : sets nc to simply scan for listening daemons, without actually sending any data to them.
# -v : enables verbose mode
nc -zv 192.168.1.15 22

# 扫描多个端口
nc -zv 192.168.56.10 80 22 21
nc -zv 192.168.56.10 20-80

# 启动 8888 端口的 udp 监听
nc -ulp 8888

# 检查 udp 端口是否正常
nc -zvu localhost 8888
```


## NMAP

```shell
# 端口扫描
nmap -Pn 172.19.50.218
```
