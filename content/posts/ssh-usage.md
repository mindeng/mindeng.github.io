
---
title: "ssh 常见用法介绍"
date: 2022-11-15T08:27:00.000Z
lastmod: 2023-06-17T10:14:00.000Z
tags: ['ssh', 'linux', 'tools']
draft: false
---



ssh 是平时十分常用的工具，这里记录一些常见用法。


## 一些花样玩法


### 通过 ssh 共享 tmux session  
  
-   remote machine: ``tmux -S /tmp/shared-tmux-socket new-session``  
-   local machine: ``ssh -t remote-machine tmux -S /tmp/shared-tmux-socket attach``

通过上面简单两步，就可以通过 ssh 远程远程共享 tmux session 了。

跟普通的 ssh 登录上去再 tmux 还是有一定区别的，其中最大的区别，是本地、远程能同时看到这个 session，包括你在 tmux 里面敲的每个命令，以及终端的所有输出，都是实时同步的，有点「终端版本的桌面共享」的意思。感觉用来做远程教学挺方便的。


## ssh 客户端配置文件 

通常我们在连接不同的服务器时，需要使用不同的参数。例如最常见的用户名不
同、端口不同、私钥不同等。这时可以配置 ``~/.ssh/config`` 文件，按照不同
的服务器，列出各自的参数，避免每次登陆时都需要重复输入这些参数。

常用配置示例：

```bash
# ~/.ssh/config

Host *
	 Port 1234

	 # 是否压缩
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

更多配置可参考 ``man ssh_config`` 。

Tips:  
  
-   如果要移除一个过期了的服务器指纹（例如遇到 ``WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`` 的警告），可以使用该命令： ``ssh-keygen -R github.com`` 


## ssh-agent

如果私钥设置了密码，每次登陆都需要输入密码，比较麻烦。 ``ssh-agent`` 可
以解决这个问题。ssh-agent 允许在整个 session 中，只输入一次密码。


### 建立 session

```bash
# 开新的 shell 并启用 ssh-agent, bash 可以换成 zsh
ssh-agent bash

# 在当前 shell 下启用  ssh-agent
eval `ssh-agent`

# 退出 ssh-agent
ssh-agent -k

```


### 添加私钥

```bash
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


### 通过私钥导出公钥

有时候我们只有私钥文件，这时可以通过如下命令导出其对应的公钥。

```bash
ssh-keygen -y -f ~/.ssh/id_rsa
```


### eval `ssh-agent` 或 eval "$(ssh-agent)" 的一点解释

在运行 ssh-agent 的时候，一般使用  `eval `ssh-agent`` 或 ``eval "$(ssh-agent)"`` ，而不是直接运行 ssh-agent，什么原因呢？

当你直接运行 ``ssh-agent`` 时，它会启动一个新的代理进程并输出一些环境变量（打印到标准输出），例如 ``SSH_AGENT_PID`` 和 ``SSH_AUTH_SOCK``。这些环境变量用于告诉 SSH 客户端如何与代理进程进行通信。然而，直接运行 ``ssh-agent`` 并不会设置这些环境变量，它们只会被输出到终端。

```bash
$ ssh-agent
SSH_AUTH_SOCK=/tmp/ssh-xxxxxxxxxx/agent.12345; export SSH_AUTH_SOCK;
SSH_AGENT_PID=12345; export SSH_AGENT_PID;
echo Agent pid 12345;


```

相反，当你使用 ``eval "$(ssh-agent)"`` 时，``eval`` 命令会解析并执行 ``ssh-agent`` 输出的环境变量设置命令。这样，环境变量就会被正确设置，SSH 客户端就能与代理进程进行通信。

```bash
$ eval "$(ssh-agent)"
Agent pid 12345
```

总之，使用 ``eval "$(ssh-agent)"`` 能确保 SSH 客户端能够正确识别并使用代理进程，而直接运行 ``ssh-agent`` 只会输出环境变量，而不会将它们设置到当前 shell 环境中。为了确保 ``ssh-agent`` 能正常工作，建议使用 ``eval "$(ssh-agent)"`` 命令启动它。


### 如何自动运行 ssh-agent

可以将 ``eval "$(ssh-agent)"`` 添加到 ``~/.bash_profile`` 或 ``~/.profile`` 文件中（取决于你的系统配置）。这些文件仅在登录时执行一次，这样就可以避免重复启动 ``ssh-agent``。

> 注意，请不要在 ``~/.bashrc`` 文件中添加 ``eval "$(ssh-agent)"``  原因如下：  

然后，你需要确保 ``~/.bashrc`` 或其他会话初始化脚本中包含以下代码，以便在新的 shell 会话中继承父会话的环境变量：

```bash
if [ -n "$SSH_AUTH_SOCK" ]; then
  export SSH_AUTH_SOCK
fi

if [ -n "$SSH_AGENT_PID" ]; then
  export SSH_AGENT_PID
fi
```

这样，在登录时启动的 ``ssh-agent`` 实例将在所有新的 shell 会话中可用，而不会启动额外的代理进程。


## 上传公钥

上传公钥有两种方式：  
  
1.  利用 ``ssh-copy-id`` 命令    
    
    ```shell
    # -i id_rsa 参数可以省略
    ssh-copy-id -i id_rsa user@svr
    
    ```  
1.  利用 ssh 执行远程命令上传    
    
    ```shell
    cat ~/.ssh/id_rsa.pub | ssh user@svr "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
    
    ```

当然，你也可以登陆服务器，然后将公钥手动粘贴到服务器端。


## 端口转发


### 动态端口转发 

```shell
# -D: 指定本地要转发的端口，使用 SOCSOCKS4/SOCKS5 协议
# -N: 表示不执行远程命令，仅用于端口转发
ssh -D 1080 svr -N

```

利用上述建立的 SOCSOCKS4/SOCKS5 服务来获取数据：

```shell
curl -x socks5://localhost:1080 <https://www.google.com/>

```


### 本地端口转发 

转发到其他机器：

```shell
# 建立从本机 11111 端口到 www.baidu.com:80 的加密隧道，通过 svr ssh 服务器做中转
# -f: 后台运行
ssh -NfL 11111:www.baidu.com:80 svr

# 访问本机的 11111 端口，相当于通过 svr 访问 www.baidu.com:80
# 这里需要设置 host header, 否则百度会拒绝访问
curl --header 'Host: www.baidu.com' localhost:11111

```

转发到 ssh 服务器本机的其他端口：

```shell
# svr 的 8118 端口上启动一个 http 代理服务器，例如 privoxy/squid/varnish 等
ssh -NfL 11111:localhost:8118 svr

# 本机的 11111 端口，可以作为代理端口来使用
curl -x http://localhost:11111 www.google.com

```


### 远程端口转发 

**远程端口转发** 和 **本地端口转发** 刚好相反，本地转发是 **本地计算机通过
加密隧道访问远程主机**, 而远程转发是 **远程主机通过加密隧道访问本机计算
机** 。

下面通过两个案例来具体介绍其用法。


### 案例一：将内网的某个服务映射到外网服务器上 

```shell
# 将 my.public.svr:80 映射到 192.168.1.10:8080
# 外网访问 my.public.svr:80 即可访问到内网的 192.168.1.10:8080
ssh -R 80:192.168.1.10:8080 my.public.svr

# 将 my.public.svr:8080 映射到本机的 80 端口
ssh -R 8080:localhost:80 my.public.svr

```


### 案例二：将内网服务器通过 ssh 跳板机映射到本机

```shell
# 在 ssh 跳板机上执行，假设 192.168.1.10 为本机 IP 地址（前提是本机装有 sshd 服务）
# 之后在本机执行 ssh -p 2222 localhost 即可登陆 my.private.svr
ssh -R 2222:my.private.svr:22 192.168.1.10

```


## 利用端口转发做 ssh 二次跳转   
  
1.  建立 localhost 到 host1 的隧道    
    
    ```shell
    ssh -L 9999:host2:1234 -N host1
    
    ```    
    
    注意，这里 host1 到 host2 的连接是非加密的。  
1.  建立 localhost 到 host1 的隧道，同时建立 host1 到 host2 的隧道    
    
    ```shell
    ssh -L 9999:localhost:9999 host1 ssh -L 9999:localhost:1234 -N host2
    
    ```    
    
    host1 到 host2 的连接也是加密的，但是 host1:9999 到 host2:1234 的隧
    道可以被 host1 上的任意用户使用。  
1.  建立 localhost 到 host1 的隧道，再建立 localhost 到 host2 的隧道    
    
    ```shell
    # 转发本机 9998 端口到 host2:22
    ssh -L 9998:host2:22 -N host1
    # 通过 9998 端口连接 host2 的 sshd, 并转发本机端口 9999 到 host2:1234
    ssh -L 9999:localhost:1234 -N -p 9998 localhost
    
    ```    
    
    如果 1234 服务只能通过 host2 本机访问，则可以通过这种方式来间接访问
    该服务。


参考：  
  
1.  [An SSH tunnel via multiple hops - Super User](https://superuser.com/a/97007/869637)  
1.  [Emacs Tramp ssh double hop - Stack Overflow](https://stackoverflow.com/questions/5847814/emacs-tramp-ssh-double-hop)


## 利用代理命令做 ssh 二次跳转 - double hop

```shell
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

执行 ``ssh endpoint2`` 会自动通过跳板机 endpoint1 来登陆 endpoint2 。

另见 [ssh 常见用法介绍]({{< relref "ssh-usage#%E5%88%A9%E7%94%A8%E7%AB%AF%E5%8F%A3%E8%BD%AC%E5%8F%91%E5%81%9A-ssh-%E4%BA%8C%E6%AC%A1%E8%B7%B3%E8%BD%AC" >}}) 。


## ssh 证书登陆


### 证书签发

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


### 证书配置  
  
1.  配置服务器证书：通过 sshd_config 的 HostCertificate 字段来进行配
置；  
1.  配置用户证书：拷贝到用户的 ~/.ssh/ 目录即可；  
1.  服务器信任用户证书：    
    
    通过 sshd_config 的 TrustedUserCAKeys 字段，配置服务器信任的
    ca-user 公钥。即服务器会信任由 TrustedUserCAKeys 指定的文件里的公
    钥所签发的用户证书。  
1.  客户端信用服务器证书：    
    
    在 ~/.ssh/known_hosts 中增加一行：    
    
    ```shell
    @cert-authority * ssh-rsa <ca-server-public-key>
    
    ```    
    
    <ca-server-public-key> 替换成 ca-server 公钥即可。


### ssh 证书登陆流程

大致的 ssh 证书登陆流程如下所示：  
  
1.  用户登陆 ssh 服务器，ssh 客户端自动将用户证书发给服务器  
1.  服务器校验用户证书是否可信和有效，验证通过后，将服务器证书发给用户侧  
1.  用户侧校验服务器证书是否可信和有效，验证通过后，走正常登陆流程


## ssh 服务端配置文件

常用配置示例：

```shell
# /etc/ssh/sshd_config

Port 2222

# 禁止密码登陆
PasswordAuthentication no

```


## 使用 ed25519 key

生成 ed25519 key

```shell
# 生成 key，-a 指定 KDF round 数量，默认 16，越高 passphrase 越难破解
ssh-keygen -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C "min@mincodes.com"

# add key to ssh-agent
ssh-add ~/.ssh/id_ed25519

# add all keys to ssh-agent
ssh-add
```


## ssh key 用于签名


### git 签名

[https://calebhearth.com/sign-git-with-ssh](https://calebhearth.com/sign-git-with-ssh)


### 文件签名  
  
-   签名    
    
    ```shell
    # -n 参数指定 namespace，表达签名的目的
    # 改命令会生成 README.org.sig 签名文件
    ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n file README.org
    ```  
-   校验    
    
    先准备一份 allowed_signers 文件，内容是 id 和 public key 的映射：    
    
    ```shell
    min ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDxUFhyoSm3ZRR4Pm7rc/U8OdOGUWggrsUFzabTxI2lC
    ```    
    
    执行命令：    
    
    ```shell
    # 校验签名
    # -I 指定要验证谁的签名
    ssh-keygen -Y verify -f /tmp/allowed_signers -I min -n file -s README.org.sig < README.org
    ```    
        
    
    参考：[https://www.agwa.name/blog/post/ssh_signatures](https://www.agwa.name/blog/post/ssh_signatures)    
    
    
    ## ssh over http proxy    
    
    通常我们会用到 http over ssh，但其实也可以反过来，ssh over http，即通过 Web Proxy 来使用 ssh。    
    
    修改 ``.ssh/config`` 配置文件：    
    
    ```plain text
    Host ssh.server
        ProxyCommand connect -H web-proxy.server %h %p
    
    ```    
    
    这在有些受限环境下非常有用（例如有些公司网络只能通过http代理访问外网）。