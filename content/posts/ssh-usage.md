+++
title = "ssh 用法简介"
date = 2022-03-30T00:00:00+08:00
tags = ["ssh", "tools"]
draft = false
+++

ssh 是平时十分常用的工具，这里记录一些常见用法。


## ssh 客户端配置文件 {#ssh-客户端配置文件}

通常我们在连接不同的服务器时，需要使用不同的参数。例如最常见的用户名不
同、端口不同、私钥不同等。这时可以配置 `~/.ssh/config` 文件，按照不同
的服务器，列出各自的参数，避免每次登陆时都需要重复输入这些参数。

常用配置示例：

```cfg
# ~/.ssh/config

Host *
	 Port 1234

	 # 是否加密
	 Compression yes

	 # 在给定秒数内，没收到服务器数据，则向服务器请求一个回包，可用于连接保活
	 ServerAliveInterval 60

	 # 禁止发送 TCP keepalive 消息，避免临时网络中断引起连接中断
	 TCPKeepAlive no

Host svr
	 HostName svr.example.com
	 User tom
	 Port 2222
	 IdentityFile ~/.ssh/id_rsa-svr

	 # 指定本机 IP 地址
	 BindAddress 192.168.10.235
```

更多配置可参考 `man ssh_config` 。


## ssh-agent {#ssh-agent}

如果私钥设置了密码，每次登陆都需要输入密码，比较麻烦。 `ssh-agent` 可
以解决这个问题。ssh-agent 允许在整个 session 中，只输入一次密码。


### 建立 session {#建立-session}

```sh
# 开新的 shell 并启用 ssh-agent, bash 可以换成 zsh
ssh-agent bash

# 在当前 shell 下启用  ssh-agent
eval `ssh-agent`

# 退出 ssh-agent
ssh-agent -k
```


### 添加私钥 {#添加私钥}

```sh
# 添加默认私钥，按照提示输入密码（passphrase）
ssh-add

# 添加指定的私钥
ssh-add ~/.ssh/id_rsa-svr

# 列出所有已添加的私钥
ssh-add -l

# 从内存中移除指定的私钥
ssh-add -d ~/.ssh/id_rsa-svr

# 从内存中删除所有已添加的私钥
ssh-add -D
```


## 上传公钥 {#上传公钥}

上传公钥有两种方式：

1.  利用 `ssh-copy-id` 命令

    ```sh
    # -i id_rsa 参数可以省略
    ssh-copy-id -i id_rsa user@svr
    ```
2.  利用 ssh 执行远程命令上传

    ```sh
    cat ~/.ssh/id_rsa.pub | ssh user@svr "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
    ```

当然，你也可以登陆服务器，然后将公钥手动粘贴到服务器端。


## 端口转发 {#端口转发}


### 动态端口转发 {#动态端口转发}

```sh
# -D: 指定本地要转发的端口，使用 SOCSOCKS4/SOCKS5 协议
# -N: 表示不执行远程命令，仅用于端口转发
ssh -D 1080 svr -N
```

利用上述建立的 SOCSOCKS4/SOCKS5 服务来获取数据：

```sh
curl -x socks5://localhost:1080 https://www.google.com/
```


### 本地端口转发 {#本地端口转发}

转发到其他机器：

```sh
# 建立从本机 11111 端口到 www.baidu.com:80 的加密隧道，通过 svr ssh 服务器做中转
# -f: 后台运行
ssh -NfL 11111:www.baidu.com:80 svr

# 访问本机的 11111 端口，相当于通过 svr 访问 www.baidu.com:80
# 这里需要设置 host header, 否则百度会拒绝访问
curl --header 'Host: www.baidu.com' localhost:11111
```

转发到 ssh 服务器本机的其他端口：

```sh
# svr 的 8118 端口上启动一个 http 代理服务器，例如 privoxy/squid/varnish 等
ssh -NfL 11111:localhost:8118 svr

# 本机的 11111 端口，可以作为代理端口来使用
curl -x http://localhost:11111 www.google.com
```


### 远程端口转发 {#远程端口转发}

**远程端口转发** 和 **本地端口转发** 刚好相反，本地转发是 **本地计算机通过
加密隧道访问远程主机**, 而远程转发是 **远程主机通过加密隧道访问本机计算
机** 。

下面通过两个案例来具体介绍其用法。


#### 案例一：将内网的某个服务映射到外网服务器上 {#案例一-将内网的某个服务映射到外网服务器上}

```sh
# 将 my.public.svr:80 映射到 192.168.1.10:8080
# 外网访问 my.public.svr:80 即可访问到内网的 192.168.1.10:8080
ssh -R 80:192.168.1.10:8080 my.public.svr

# 将 my.public.svr:8080 映射到本机的 80 端口
ssh -R 8080:localhost:80 my.public.svr
```


#### 案例二：将内网服务器通过 ssh 跳板机映射到本机 {#案例二-将内网服务器通过-ssh-跳板机映射到本机}

```sh
# 在 ssh 跳板机上执行，假设 192.168.1.10 为本机 IP 地址（前提是本机装有 sshd 服务）
# 之后在本机执行 ssh -p 2222 localhost 即可登陆 my.private.svr
ssh -R 2222:my.private.svr:22 192.168.1.10
```

另见 [利用端口转发做 ssh 二次跳转](#利用端口转发做-ssh-二次跳转),  [利用代理命令做 ssh 二次跳转（double hop）](#利用代理命令做-ssh-二次跳转-double-hop) 。


## 利用端口转发做 ssh 二次跳转 {#利用端口转发做-ssh-二次跳转}

1.  建立 localhost 到 host1 的隧道

    ```sh
    ssh -L 9999:host2:1234 -N host1
    ```

    注意，这里 host1 到 host2 的连接是非加密的。
2.  建立 localhost 到 host1 的隧道，同时建立 host1 到 host2 的隧道

    ```sh
    ssh -L 9999:localhost:9999 host1 ssh -L 9999:localhost:1234 -N host2
    ```

    host1 到 host2 的连接也是加密的，但是 host1:9999 到 host2:1234 的隧
    道可以被 host1 上的任意用户使用。
3.  建立 localhost 到 host1 的隧道，再建立 localhost 到 host2 的隧道

    ```sh
    # 转发本机 9998 端口到 host2:22
    ssh -L 9998:host2:22 -N host1
    # 通过 9998 端口连接 host2 的 sshd, 并转发本机端口 9999 到 host2:1234
    ssh -L 9999:localhost:1234 -N -p 9998 localhost
    ```

    如果 1234 服务只能通过 host2 本机访问，则可以通过这种方式来间接访问
    该服务。

另见 [利用代理命令做 ssh 二次跳转（double hop）](#利用代理命令做-ssh-二次跳转-double-hop) 和 [将内网服务器通过 ssh
跳板机映射到本机](#案例二-将内网服务器通过-ssh-跳板机映射到本机) 。

参考：

1.  [An SSH tunnel via multiple hops - Super User](https://superuser.com/a/97007/869637)
2.  [Emacs Tramp ssh double hop - Stack Overflow](https://stackoverflow.com/questions/5847814/emacs-tramp-ssh-double-hop)


## 利用代理命令做 ssh 二次跳转（double hop） {#利用代理命令做-ssh-二次跳转-double-hop}

```cfg
Host endpoint2
	 User myusername
	 HostName mysite.com
	 Port 3000
	 ProxyCommand ssh endpoint1 nc -w300 %h %p

Host endpoint1
	 User somename
	 HostName otherdomainorip.com
	 Port 6893
```

执行 `ssh endpoint2` 会自动通过跳板机 endpoint1 来登陆 endpoint2 。

另见 [利用端口转发做 ssh 二次跳转](#利用端口转发做-ssh-二次跳转) 。


## ssh 证书登陆 {#ssh-证书登陆}


### 证书签发 <span class="tag"><span class="certificate">certificate</span><span class="ca">ca</span></span> {#证书签发}

要使用证书登陆，首先需要为服务器、用户端分别签发证书。

CA 密钥：

-   CA 要签发证书，首先需要一对密钥，可以通过 ssh-keygen 来生成密钥对
-   考虑到安全性和灵活性，在签发服务器证书和用户证书时，可以分别使用两对
    不同的密钥，这里分别称为 ca-server 密钥对和 ca-user 密钥对

签发证书：

-   签发 ssh 服务器证书：

    使用 ca-server 私钥 + 服务器公钥 + 其他信息（域名，identity, 有效期
    等），生成服务器证书。
-   签发 ssh 用户证书：

    使用 ca-user 私钥 + 用户公钥 + 其他信息（identity, 有效期等），生成
    用户证书。

-   证书可以通过 identity 来撤销（revoke），或者通过指定 public key 来
    revoke


### 证书配置 {#证书配置}

1.  配置服务器证书：通过 sshd_config 的 HostCertificate 字段来进行配
    置；
2.  配置用户证书：拷贝到用户的 ~/.ssh/ 目录即可；
3.  服务器信任用户证书：

    通过 sshd_config 的 TrustedUserCAKeys 字段，配置服务器信任的
    ca-user 公钥。即服务器会信任由 TrustedUserCAKeys 指定的文件里的公
    钥所签发的用户证书。
4.  客户端信用服务器证书：

    在 ~/.ssh/known_hosts 中增加一行：

    ```cfg
    @cert-authority * ssh-rsa <ca-server-public-key>
    ```

    &lt;ca-server-public-key&gt; 替换成 ca-server 公钥即可。


### ssh 证书登陆流程 {#ssh-证书登陆流程}

大致的 ssh 证书登陆流程如下所示：

1.  用户登陆 ssh 服务器，ssh 客户端自动将用户证书发给服务器
2.  服务器校验用户证书是否可信和有效，验证通过后，将服务器证书发给用户侧
3.  用户侧校验服务器证书是否可信和有效，验证通过后，走正常登陆流程


## ssh 服务端配置文件 {#ssh-服务端配置文件}

常用配置示例：

```cfg
# /etc/ssh/sshd_config

Port 2222

# 禁止密码登陆
PasswordAuthentication no
```