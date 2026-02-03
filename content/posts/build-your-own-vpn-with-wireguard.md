+++
title = "利用 WireGuard 快速构建属于你自己的虚拟专用网络"
lastmod = 2026-02-03T11:14:47+08:00
tags = ["wireguard", "network", "tunnel", "vpn", "homelab"]
draft = false
+++

## 引言 {#引言}

> 本文详细分析了 WireGuard 的数据收发、数据中继的基本原理，并介绍了一种利用
> WireGuard 快速构建一个属于自己的虚拟专用网络的方法，可用于跨广域网安全通信，以及内网穿透。常见应用场景有：
>
> -   从公网安全访问家庭内网。例如：访问家庭摄像头，访问智能门锁，访问 NAS 服务器等。
> -   从公网安全访问公司内网，满足远程办公需求。
>
> “快速”二字主要体现在本文介绍的方法可以较大程度简化配置和管理，并允许线性扩展网络规模，且不倚赖第三方服务（例如 tailscale）。


## 为什么选择 WireGuard {#为什么选择-wireguard}

安全
: WireGuard 的安全性经过[形式化验证](https://zh-wireguard.com/formal-verification/)。

开源
: WireGuard 完全开源，有 Go 和 C 语言两种实现。

跨平台
: 多平台支持，包括 Linux, Windows, macOS, Android, iOS 等。且内核模块已进入 Linux 内核树。

性能
: 快速，现代化，精简且实用。


## 方案介绍 {#方案介绍}

本文介绍的方法是利用 WireGuard 搭建一个逻辑上的「星型」网络结构，如下图所示：

```uniline
          ╭─────────╮                          ╭───────────╮
          │ MacBook <╌╌╌╌╌╌╌╌╌╌╌╮   ╭╌╌╌╌╌╌╌╌╌╌> Cellphone │
          ╰─10.9.9.1╯           ┆   ┆          ╰──╴10.9.9.2╯
                            ╭┬──v───v─┬╮☁️
                            ││ router ││      ╔═════════════════ office ══════╗
                            ╰┴10.9.0.1┴╯      ║                   ╭─────────╮ ║
                                ^   ^         ║ ╭┬──────────┬╮  ╭─┤ jenkins │ ║
                     ╭╌╌╌╌╌╌╌╌╌╌╯   ╰╌╌╌╌╌╌╌╌╌>─┤│ offic-gw │├──╯ ╰─────────╯ ║
                     ┆                        ║ ╰┴╴10.9.0.3╶┴╯       ...      ║
                     ┆                        ║                               ║
                     ┆                        ╚═════════════╸172.19.50.0/24 ══╝
╔════════════════════v═══════ homelab ═══╗
║              ╭┬────┴────┬╮             ║
║              ││ home-gw ││             ║                  <╌╌╌╌>
║              ╰┴╴10.9.0.2┴╯             ║           ╭─╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌─╮
║      ╭─────────────┼──────────────╮    ║           ┆ WireGuard Tunnel ┆
║ ╭────┴───╮  ╭──────┴─────╮     ╭──┴──╮ ║           ╰─╌╌10.9.0.0/16╶╌╌─╯
║ │ camera │  │ smart-lock │ ... │ NAS │ ║
║ ╰────────╯  ╰────────────╯     ╰─────╯ ║
╚══════════════════════ 192.168.1.0/24 ══╝

```


### 设备说明 {#设备说明}

router
: 云端的一台服务器，有公网 IP，充当 tunnel 网络的路由器，提供路由中转服务。

home-gw
: 放在家里的一台服务器，充当家庭网络 (homelab) 的网关，通过该网关可以安全地穿透到家庭网络，访问家里的各种设备，例如：摄像头，智能门锁，NAS 服务器等。

office-gw
: 放在公司办公室的一台服务器，充当办公室网络 (office) 的网关,通过该网关可以安全地访问到公司内网的各项服务，满足远程办公需求。

MacBook
: 作为移动办公设备，位置不固定，可能在办公室、在家里，也可能处在咖啡厅、图书馆等外部网络。

Cellphone
: 手机，位置不固定，和 MacBook 类似。

其中， home-gw 和 office-gw 所扮演的角色，其实就是 tailscale 中的 [subnet
routers](https://tailscale.com/kb/1019/subnets) 。


### 网段划分 {#网段划分}

`10.9.0.0/16`
: WireGuard tunnel 网段，是我们为 WireGuard 节点分配的网段（也可以改成你喜欢的任何局域网网段，只要不与现有的 LAN 网段冲突即可）。

    其内部划分出如下网段：

    -   **`10.9.0.0/24`:** 分配给 router 和 gateway。
    -   **`10.9.9.0/24`:** 分配给移动设备，包括 MacBook, Cellphone 等。

    这样划分主要是为了便于区分不同类型的设备，同时也起到一个隔离保护的作用。

`192.168.1.0/24`
: 家庭局域网私有网段，这里称为 homelab。

`172.19.50.0/24`
: 公司局域网私有网段，这里称为 office。


### 连通性说明 {#连通性说明}

-   MacBook 和 Cellphone 可以访问所有网络，包括：
    -   office：满足移动办公需求。
    -   homelab: 方便随时访问家庭网络中的各项设备。
-   office 和 homelab 之间默认无法互通，起到隐私安全方面的保护作用。


### 方案优缺点 {#方案优缺点}

方案优点：

-   简化 WireGuard 的配置和管理

    只有二级节点（指直接与 router 相连的节点）需要与 router 互换公钥，二级节点之间无需互相了解，二级节点后面的所有设备也无需任何配置。这样可以最大程度简化
    WireGuard 的配置和管理工作，同时允许以线性复杂度扩展接入 tunnel 的设备数量。
-   扩展性强

    上面提到二级节点后面的设备无需任何配置，就能通过 WireGuard 网络访问。这点其实很重要，因为像摄像头、智能门锁等设备是无法进行这类配置的。该方案基本上允许我们无限扩展 WG 网络的可访问范围。
-   安全、透明地访问内网资源

    MacBook/Cellphone 可以安全、透明地访问 homelab/office 的内网资源，被访问资源无需其他额外设置。

方案缺点：

-   设备之间的通信都需要经由中心的 router 服务器转发，可能存在单点故障和性能瓶颈。

作为个人用途而言，这个缺点一般是可以接受的。原因如下：

-   私人网络一般数据量都比较小，不太可能出现性能瓶颈。
-   除非接入 WireGuard 的每台设备都有外网 IP，否则转发不可避免。
-   现在的云服务器还是相当稳定的，出故障的概率极低。要真碰上了，最差也就只是访问不了内网（跟没配置这个 tunnel 时一样），也不会有啥其他影响。


## WireGuard 原理介绍 {#wireguard-原理介绍}

在方案实施之前，有必要先了解一下 WireGuard 的路由寻址、加解密通信原理，这对理解我们的方案有很大帮助。


### 最简 WireGuard Tunnel 配置 {#最简-wireguard-tunnel-配置}

为了方便讲解，我们先从一个最简单的 WireGuard Tunnel 配置开始。

该 tunnel 仅包含 A, B 两个节点：

```uniline
Public IP:  1.1.1.A                 1.1.1.B
            ╭──────╮                ╭──────╮
            │node A├───╴Internet ───┤node B│
            ╰──────╯                ╰──────╯
WG IP:      10.0.0.A                10.0.0.B
```

上图中，"Public IP" 指公网 IP，"WG IP" 指我们为 WireGuard 指定的虚拟局域网 IP。

经过上述最简单的 WG 组网，我们可以通过 `10.0.0.A` 和 `10.0.0.B` 这两个 IP 地址，实现节点 A 和节点 B 之间的安全加密通信。

两个节点的配置分别见下方（其中 A 和 B 为占位符，实际使用时，请自行修改）。

Node A 的配置清单（Endpoint 中节点 B 的公网 IP 请按照实际情况进行修改）：

```cfg
[Interface]
Address = 10.0.0.A
PrivateKey = <private key>

# Node B
[Peer]
PublicKey = <B`s public key>
AllowedIPs = 10.0.0.B/32
Endpoint = 1.1.1.B:51820
```

Node B 的配置清单：

```cfg
[Interface]
Address = 10.0.0.B
PrivateKey = <private key>

# Node A
[Peer]
PublicKey = <A`s public key>
AllowedIPs = 10.0.0.A/32
# A 的 Endpoint 会在协议握手之后自动获取
```


### WireGuard 数据发送 &amp; 接收流程（示意图） {#wireguard-数据发送-and-接收流程-示意图}

按照上述最简配置，以 `ping` 命令为例，我画了一张 WireGuard 数据发送 &amp; 接收流程示意图：

```uniline
       Node A                                                  Node B

╭───────╴user space╶────╮
│   ping 10.0.0.B       │
╰─────────┼─────────────╯
          1
╭─────────v───kernel────╮                      ╭───────────────kernel─────────────╮
│ ╭───────────────────╮ │                      │ ╭──────────────────────────────╮ │
│ │   TCP/IP stack    │ │                      │ │         TCP/IP stack      ╭────>──╮
│ ╰──┬────^───────┬───╯ │                      │ ╰──^──────────┬───^────┬────^──╯ │  │
│    │    │       │     │                      │    │          │   │    │    │    │  │
│  ICMP  UDP     UDP    │                      │   UDP        UDP ICMP Reply UDP  │  │
│    2    3       4     │                      │    5          6   7    8    9    │  10
╰─┬─╴v────┴─┬─┬───v────┬╯                      ╰─┬──┴──────┬─┬─v───┴────v────┴───┬╯  │
  │   wg0   │ │  eth0  │──────╮               ╭──>  eth0   │ │        wg0        │   │
  ╰10.0.0.A─╯ ╰1.1.1.A─╯      │               │  ╰─1.1.1.B─╯ ╰──────10.0.0.B─────╯   │
                  ^         ICMP Req          │                                   via eth0
                  │      (Encrypte in UDP)    │                                      │
                  │           │               ╵                                      │
                  │           │     ╭─────UDP datagram─────╮             ╭─────UDP datagram─────╮
                  │           ╰────>│  src: 1.1.1.A:xxxxx  │             │  src: 1.1.1.B:51820  │
                  │                 │  dst: 1.1.1.B:51820  │             │  dst: 1.1.1.A:xxxxx  │
                  │                 ╰─┬─────WG Header────┬─╯             ╰─┬─────WG Header────┬─╯
                  │                  ╭╯  Received Index  ╰╮               ╭╯  Received Index  ╰╮
                  │                  │   Counter          │               │   Counter          │
                  │                  ├──────Encrypted─────┤               ├──────Encrypted─────┤
                  │                  │  ╭─────ICMP────╮   │               │  ╭─╴ICMP Reply─╮   │
                  │                  │  │src: 10.0.0.A│   │               │  │src: 10.0.0.B│   │
                  │                  │  │dst: 10.0.0.B│   │               │  │dst: 10.0.0.A│   │
                  │                  ╰──┴─────────────┴───╯               ╰──┴─────────────┴───╯
                  │                                                                 │
                  ╰───────────────────────ICMP Reply to A───────────────────────────╯
                                         (Encrypted in UDP)
```

> 💡由于 `ping` 的响应由 kernel 的 TCP/IP stack 自动完成，因此上图中的 Node B 在
> user space 并没有对应的用户空间处理程序。
>
> 如果是普通的 TCP/UDP 应用，则会将数据丢给用户空间对应的处理程序，形成一个 Node
> A app &harr; Node B app 的完整闭环。

下面，我们就按照上图标注的 1~7 个步骤，依次分析 WireGuard 的数据发送 &amp; 接收过程。


#### WireGuard 数据发送流程 {#wireguard-数据发送流程}

> 💡WireGuard 虽然是通过 UDP 来传输数据的，但其建立的网络隧道确是工作在 Layer 3
> 上的（通过创建虚拟网络接口来实现）。

在 Node A 上执行命令 `ping 10.0.0.B` 时，会发生下面几件事情（大致对应 [WireGuard
数据发送 &amp; 接收流程示意图](#wireguard-数据发送-and-接收流程-示意图) 中的步骤 1~4）：

1.  应用层产生报文
    1.  在 node A 上执行 `ping 10.0.0.B` 命令
    2.  应用层交给内核协议栈生成一个 ICMP 报文
        ```cfg
        [Inner IP Header]               # 源=10.0.0.A，目的=10.0.0.B
        [ICMP Echo Request]
        ```
2.  路由查找
    1.  内核查路由表，发现 packet 的目标地址 10.0.0.B 应该由 WireGuard 接口 wg0
        处理。

        > 💡在配置文件中的 AllowedIPs 中声明的地址，都会自动加入到对应 WireGuard 接口的路由表项中。可以通过 `ip route` 命令可以很轻松的验证这一点。
    2.  内核将该 packet 交给 wg0 处理。
3.  WireGuard 接口处理（出站方向）
    1.  确定目标 Peer: WireGuard 内核模块查找 AllowedIPs，找到 10.0.0.B 属于Peer
        B。
    2.  加密：
        -   取出会话密钥 (session key) —— 该密钥由 WG 协议握手协商得来

        -   使用 ChaCha20-Poly1305 将整个原始 IP packet 加密
    3.  加上 WG Data Header, 包含如下信息：

        -   Receiver Index (32bit, 标识接受方的会话）
        -   Counter (64bit, 防重放计数）

        将 WG Data Header 和加密后的数据放到一个 UDP 包中：
        ```cfg
        [Outer IP Header]   # 源 = A 公网地址，目的 = B 公网地址（Endpoint 指定）
        [UDP Header]        # 源端口=随机/固定，目的端口=51820 (Endpoint 指定)
        [WG Data Header]    # Receiver Index, Counter
        [Encrypted Payload] # 原始 IP 报文 (10.0.0.A → 10.0.0.B)
        ```

4.  UDP 发送

    内核把这个 UDP packet 交给物理网卡，发送到 Internet &rarr; node B 。

从上述过程中可以看到， **在发送数据的过程中，AllowedIPs 起到类似路由表 (routing
table) 的作用** 。


#### WireGuard 数据接收流程 {#wireguard-数据接收流程}

下面是 node B 接收数据的过程（大致对应 [WireGuard 数据发送 &amp; 接收流程示意图](#wireguard-数据发送-and-接收流程-示意图) 中的步骤 5~7）：

5.  到达网卡
    1.  外层 UDP packet 到达 node B 的 WireGuard 监听端口（这里是 51820）。
    2.  内核网络栈把该 packet 交给 wg0 接口对应的 WireGuard 内核模块。
6.  WireGuard 接口处理（入站方向）
    1.  定位会话
        1.  WireGuard 内核模块读取 Receiver Index
        2.  根据这个 index 查找本地存储的会话信息 (session key, peer)。
        3.  如果找不到 &rarr; 丢弃。
    2.  防重放校验
        1.  使用 Counter 和本地的滑动窗口机制检查：
            -   这个包是不是比上一次的序号小？
            -   是否落在防重放窗口内？
        2.  未通过检查 &rarr; 丢弃
    3.  解密
        1.  使用找到的 session key 解密并校验完整性

        2.  解密失败 &rarr; 丢弃
    4.  校验原始 packet 的源 IP 地址

    5.  解密后得到的就是原始的 IP packet, 例如：
        ```cfg
        [Inner IP Header]   # 源=10.0.0.A，目的=10.0.0.B
        [ICMP Echo Request]
        ```
    6.  检查原始 IP packet 的源地址是否在该 Peer 的 AllowedIPs 配置范围内：
        -   ✅ 如果匹配 &rarr; 把原始报文交给 wg0 虚拟网卡 &rarr; 内核继续处理。

        -   ❌ 如果不匹配 &rarr; 直接丢弃。
7.  报文处理
    1.  内核查找路由表，发现目的 IP `10.0.0.B` 是 B 自己 (wg0 配置的隧道内 IP)

        > 💡如果发现目的 IP 不是 B 自己呢？读者可以思考一下。本节先留点悬念，后文章节 [WireGuard 数据中继原理](#wireguard-数据中继原理) 中会详细讲解。

    2.  内核将该 packet 交给本地协议栈（ICMP handler, TCP socket 等）。

从上述过程中可以看到， **在接收数据的过程中，AllowedIPs 起到类似 ACL (Access
Control List) 的作用** 。

Node B 的内核协议栈在收到 ping 后，向 Node A 发送响应 (ICMP Echo Rely) 的过程，其实就是上述步骤的逆向过程（对应图中的 8~10）：

8.  协议栈发现是发给本机的 ICMP Echo Request:
    1.  生成一个 ICMP Echo Reply。
    2.  内核查路由表，发现 Reply packet 应该交给 wg0 处理。
9.  WireGuard 查找 AllowedIPs, 找到 peer A 可以接收这个 ICMP Echo Reply，于是用
    peer A 的 session key 加密 packet, 并封装到 UDP 包中（该 UDP 包的目的地址为
    peer A 的公网 IP 地址 `10.0.0.A` ），交给内核协议栈。
10. 内核查路由表，然后通过物理网卡 `eth0` 将该 UDP 包发送给 node A。

    node A 收到 UDP 包后，按照同样的流程，将其交给监听该端口的程序 (WireGuard)
    &rarr; 找到 session key &rarr; 解密 &rarr; 交给内核协议栈 &rarr; 最终交给上层的 `ping` 应用 &rarr; 打印 ICMP Echo Reply 消息。


### WireGuard 数据中继原理 {#wireguard-数据中继原理}

前文提到，当 WireGuard 解出内层的原始 IP 报文，并交给内核处理，此时如果内核发现目的 IP 不是本机，会如何处理？

很简单，Linux 内核只有两种处理方式：

系统已打开 `ip_forward`
: 查找路由表，尝试转发。

系统未打开 `ip_forward`
: 直接丢弃。

WireGuard 的数据中继功能就是以此为基础的。

因此，本章有个前提，就是用作中继节点的 WireGuard 服务器已经打开了 `ip_forward` 。具体打开方式，可参考后文 [打开 IP 转发](#打开-ip-转发) 中的介绍。


#### WireGuard 数据中继（配置清单） {#wireguard-数据中继-配置清单}

现在咱们在 [最简 WireGuard Tunnel 配置](#最简-wireguard-tunnel-配置) 基础上，增加了一个 node C：

```uniline
Public IP:  1.1.1.A     1.1.1.B     1.1.1.C
            ╭──────╮    ╭──────╮    ╭──────╮
            │node A├────┤node B├────┤node C│
            ╰──────╯    ╰──────╯    ╰──────╯
WG IP:      10.0.0.A    10.0.0.B    10.0.0.C
```

这次需要在 node A 上，运行 `ping 10.0.0.C` 。

要满足这个需求，需要修改一下 WireGuard 配置文件。

<!--list-separator-->

-  Node A 的配置清单

    A 需要向 C 发消息，所以在 Peer B 处的 `AllowedIPs` 应该填 C 的 IP，而不是 B 的 IP。
    B 只起到一个路由转发的作用。

    ```cfg
    # A's wg0.conf
    [Interface]
    Address = 10.0.0.A/32
    PrivateKey = <private key>

    # B
    [Peer]
    PublicKey = <B`s public key>
    AllowedIPs = 10.0.0.C/32        # A \leftrightarrow C
    Endpoint = 1.1.1.B:51820        # 假设 B 的公网 IP 为 1.1.1.B
    ```

<!--list-separator-->

-  Node B 的配置清单

    B 作为中继节点，需要同时配置 peer A 和 peer C。

    ```cfg
    # B's wg0.conf
    [Interface]
    Address = 10.0.0.B/32
    PrivateKey = <private key>

    # node A
    [Peer]
    PublicKey = A`s public key
    AllowedIPs = 10.0.0.A/32

    # node C
    [Peer]
    PublicKey = C`s public key
    AllowedIPs = 10.0.0.C/32
    ```

<!--list-separator-->

-  Node C 的配置清单

    同样的，C 需要允许 A 向我发消息，所以在 Peer B 处的 `AllowedIPs` 应该填 A 的 IP，而不是 B 的 IP。B 只起到一个路由转发的作用。

    ```cfg
    # C's wg0.conf
    [Interface]
    Address = 10.0.0.C/32
    PrivateKey = <private key>

    # B
    [Peer]
    PublicKey = <B`s public key>
    AllowedIPs = 10.0.0.A/32        # A \leftrightarrow C
    Endpoint = 1.1.1.B:51820        # 假设 B 的公网 IP 为 1.1.1.B
    ```


#### WireGuard 数据中继（示意图） {#wireguard-数据中继-示意图}

还是以 `ping` 命令为例，下面是我画的 WireGuard 数据中继的流程示意图：

```uniline
       Node A                                         Node B                                              Node C
                                                  (ip_forward=1)
╭───────╴user space╶────╮                           Relay Node                                         Target Node
│   ping 10.0.0.B       │
╰─────────┼─────────────╯
          1
╭─────────v───kernel────╮             ╭───────────────kernel─────────────╮                ╭───────────────kernel─────────────╮
│ ╭───────────────────╮ │             │ ╭──────────────────────────────╮ │                │ ╭──────────────────────────────╮ │
│ │   TCP/IP stack    │ │             │ │         TCP/IP stack      ╭────>──╮             │ │         TCP/IP stack      ╭────>──╮
│ ╰──┬────^───────┬───╯ │             │ ╰──^──────────┬───^────┬────^──╯ │  │             │ ╰──^──────────┬───^────┬────^──╯ │  │
│    │    │       │     │             │    │          │   │    │    │    │  │             │    │          │   │    │    │    │  │
│  ICMP  UDP     UDP    │             │   UDP        UDP ICMP ICMP UDP   │  │             │   UDP        UDP ICMP Reply UDP  │  │
│    2    3       4     │             │    5          6   7    8    9    │  10            │    11         12  13   14   15   │  16
╰─┬─╴v────┴─┬─┬───v────┬╯             ╰─┬──┴──────┬─┬─v───┴────v────┴───┬╯  │             ╰─┬──┴──────┬─┬─v───┴────v────┴───┬╯  │
  │   wg0   │ │  eth0  │────────╮       │  eth0   │ │        wg0        │   │      ╭────────>  eth0   │ │        wg0        │   │
  ╰10.0.0.A─╯ ╰1.1.1.A─╯        │       ╰─1.1.1.B─╯ ╰──────10.0.0.B─────╯   │      │        ╰─1.1.1.C─╯ ╰──────10.0.0.C─────╯   │
                  ^       ICMP Echo Req      ^                           via eth0  │                                         via eth0
                  │     (Encrypted in UDP)   │                              │      │                                            │
                  │             │            │                              │      │                                            │
                  ╵             │    ╭─────UDP datagram─────╮      ╭─────UDP datagram─────╮     ╭─────UDP datagram─────╮        │
     Got ICMP Echo Reply!       ╰───>│  src: 1.1.1.A:xxxxx  │      │  src: 1.1.1.B:51820  │     │  src: 1.1.1.C:51820  │<───────╯
     (Encrypted in UDP)              │  dst: 1.1.1.B:51820  │      │  dst: 1.1.1.C:xxxxx  │     │  dst: 1.1.1.B:xxxxx  │
                  ╷                  ╰─┬─────WG Header────┬─╯      ╰─┬─────WG Header────┬─╯     ╰─┬─────WG Header────┬─╯
                  │                   ╭╯  Received Index  ╰╮        ╭╯  Received Index  ╰╮       ╭╯  Received Index  ╰╮
                  │                   │   Counter          │        │   Counter          │       │   Counter          │
       ╭─────UDP datagram─────╮       ├──────Encrypted─────┤        ├──────Encrypted─────┤       ├──────Encrypted─────┤
       │  src: 1.1.1.B:51820  │       │  ╭─────ICMP────╮   │        │  ╭─────ICMP────╮   │       │  ╭─ICMP Reply──╮   │
       │  dst: 1.1.1.A:xxxxx  │       │  │src: 10.0.0.A│   │        │  │src: 10.0.0.A│   │       │  │src: 10.0.0.C│   │
       ╰─┬─────WG Header────┬─╯       │  │dst: 10.0.0.C│   │        │  │dst: 10.0.0.C│   │       │  │dst: 10.0.0.A│   │
        ╭╯  Received Index  ╰╮        ╰──┴─────────────┴───╯        ╰──┴─────────────┴───╯       ╰──┴─────────────┴───╯
        │   Counter          │                                   ╭──────────╮                              │
        ├──────Encrypted─────┤                    ╭───────wg0────┴─╮ Node B │                              │
        │  ╭──ICMP Reply─╮   │<──────via eth0────╴│Encrypt into UDP│        <──────────────────────────────╯
        │  │src: 10.0.0.C│   │                 ╭──┴──ICMP Reply─╮  ├────────╯
        │  │dst: 10.0.0.A│   │                 │src: 10.0.0.C   │──╯
        ╰──┴─────────────┴───╯                 │dst: 10.0.0.A   │
                                               ╰────────────────╯



```

和前面的 [WireGuard 数据发送 &amp; 接收示意图](#wireguard-数据发送-and-接收流程-示意图) 相比较，可以看到，前面的 7 个步骤是完全一致的。区别从第 8 步开始：

8.  进入数据转发流程

    内核收到 WireGuard 解密后的 ICMP 包之后，发现这个包不是发给本机的，于是进入转发流程（前提是打开了 ip_forward）。

    首先仍然是内核查找路由表：

    -   发现目的 IP `10.0.0.C` 应该由 wg0 处理
    -   继续交给 wg0 负责发送该 ICMP 包
9.  WireGuard 接口处理（出站方向）

    到这一步之后，就和 [WireGuard 数据发送流程](#wireguard-数据发送流程) 中的出站数据处理一致了：

    1.  确定目标 peer C

    2.  重新用 peer C 的 session key 加密

    3.  封装 WG Data Header &rarr; 封装 UDP 包，UDP 的目标地址为 `1.1.1.C` 。
10. UDP 包发送至 node C。

> 💡请注意，WireGuard 在转发内层 IP 报文时，内层报文会 **原封不动** 地传递，包括：
>
> -   不会修改源地址
> -   不会修改目标地址
> -   不做 NAT
>
> WireGuard 唯一会做的就是：
>
> 接收方向
> : 确认解密后的源 IP 属于该 peer 的 AllowedIPs, 否则丢弃 （类似 ACL
>     的作用）。
>
> 发送方向
> : 根据目的 IP 查找匹配的 AllowedIPs, 把数据交给对应的 peer, 加密后发出去（类似 routing table 的作用）。
>
> 除此之外，它不会对内层 IP 报文做任何修改。
>
> 因此，node C 收到的内层报文，仍然是原始的 10.0.0.A &rarr; 10.0.0.C 的 ping 报文。

而 node C 之后的步骤，就是一个逆向返回的过程，这里就不再赘述。


#### WireGuard 内网网关（配置清单） {#wireguard-内网网关-配置清单}

有了对 WireGuard 数据中继原理的理解，就可以轻松理解 WireGuard 内网网关
(home-gw, office-gw) 是如何工作的了。

```uniline
            ╭────LAN: 192.168.1.0/24──────╮
            │   Gateway                   │
            │ 192.168.1.B     192.168.1.C │
╭1.1.1.A─╮  │  ╭1.1.1.B╶╮      ╭────────╮ │
│ node A ├──┼──┤ node B ├──────┤ node C │ │
╰────────╯  │  ╰────────╯      ╰────────╯ │
 10.0.0.A   │   10.0.0.B                  │
            ╰─────────────────────────────╯
```

请注意，这次的网络拓扑结构和之前的有所不同：

-   Node B 和 node C 处在同一个 LAN 中。
-   Node C 不是 WireGuard 节点，并没有加入 WG tunnel。

我们还是先给出配置清单。

<!--list-separator-->

-  Node A 的配置清单

    A 需要向 C 发消息，所以在 Peer B 处的 `AllowedIPs` 应该填 C 的 IP，而不是 B 的 IP。
    B 只起到一个路由转发的作用。

    ```cfg
    # A's wg0.conf
    [Interface]
    Address = 10.0.0.A/32
    PrivateKey = <private key>

    # B
    [Peer]
    PublicKey = <B`s public key>
    AllowedIPs = 192.168.1.C        # A \leftrightarrow C
    Endpoint = 1.1.1.B:51820        # 假设 B 的公网 IP 为 1.1.1.B
    ```

<!--list-separator-->

-  Node B 的配置清单

    B 作为 gateway，仅需配置 peer A （peer B 不加入 WG tunnel)。

    ```cfg
    # B's wg0.conf
    [Interface]
    Address = 10.0.0.B/32
    PrivateKey = <private key>
    # eth0 为物理网卡地址，如有不同请自行修改。
    PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

    # node A
    [Peer]
    PublicKey = A`s public key
    AllowedIPs = 10.0.0.A/32
    ```

    注意，相比之前的配置，Node B 增加了 `PostUp` 和 `PostDown` 配置。这两项配置分别会在
    wg0 接口启用、停用之后执行。也就是说：

    -   wg0 接口启动后： `iptables` 增加一条转发规则，允许将来自 wg0 的包转发给 eth0，并执行 NAT 操作。
    -   wg0 接口停止后： `iptables` 删除上述规则。

    如果没有这两条配置，Node B 就不能将数据转发给 LAN 网络。


#### WireGuard 内网网关（示意图） {#wireguard-内网网关-示意图}

还是以 `ping` 命令为例，但这次换成：

```shell
ping 192.168.1.C
```

这要求将 `ICMP` 包穿透到内网中的非 WireGuard 设备上。

下面是我画的内网穿透示意图。

```uniline
                                 ╭╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌LAN: 192.168.1.0/24╶╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╮
       Node A                    ┆              Node B                                                        Node C            ┆
                                 ┆          (ip_forward=1)                                                                      ┆
╭───────╴user space╶────╮        ┆        LAN IP: 192.168.1.B                                           LAN IP: 192.168.1.C     ┆
│   ping 192.168.1.C    │        ┆           Gateway Node                                                  Target Node          ┆
╰─────────┼─────────────╯        ┆                                                                                              ┆
          1                      ┆                                                                                              ┆
╭─────────v───kernel────╮        ┆    ╭──────────kernel─────────╮                                    ╭────────kernel╶───────╮   ┆
│ ╭───────────────────╮ │        ╰╌╌╌╴│ ╭─────────────────────╮ │                                    │ ╭──────────────────╮ │╶╌╌╯
│ │   TCP/IP stack    │ │             │ │  TCP/IP stack   ╭─────>─────╮                              │ │  TCP/IP stack    ╶─>──╮
│ ╰──┬────^───────┬───╯ │             │ ╰──^──────────┬───^───╯ │     │                              │ ╰──^───────────────╯ │  │
│    │    │       │     │             │    │          │   │     │     │                              │    │                 │  │
│  ICMP  UDP     UDP    │             │   UDP        UDP ICMP   │     │                              │  ICMP                │  │
│    2    3       4     │             │    5          6   7     │     8                              │    9                 │  10
╰─┬─╴v────┴─┬─┬───v────┬╯             ╰─┬──┴──────┬─┬─v───┴────┬╯     │                              ╰─┬──┴────── ───┬──────╯  │
  │   wg0   │ │  eth0  │────────╮       │  eth0   │ │   wg0    │  ╭────SNAT──────╮   ╭─via eth0────────>    eth0     │         │
  ╰10.0.0.A─╯ ╰1.1.1.A─╯        │       ╰─1.1.1.B─╯ ╰─10.0.0.B─╯  │╭─╴10.0.0.A   │   │                 ╰─192.168.1.C─╯         │
                  ^       ICMP Echo Req      ^                    │╰─>192.168.1.B│   │                                     via eth0
                  │     (Encrypted in UDP)   │                    ╰───┬──────────╯   │                                         │
                  │             │            │                        │              │                                         │
                  ╵             │    ╭─────UDP datagram─────╮       ╭──╴ICMP packet╶──╮                                        │
        Got ICMP Echo Reply     ╰───╴│  src: 1.1.1.A:xxxxx  │       │src: 192.168.1.B │                                        │
        (Encrypted in UDP)           │  dst: 1.1.1.B:51820  │       │dst: 192.168.1.C │                                        │
                  ╷                  ╰─┬─────WG Header────┬─╯       ╰─────────────────╯                                        │
                  │                   ╭╯  Received Index  ╰╮                                 ╭──╴ICMP Reply────╮               │
                  │                   │   Counter          │                                 │src: 192.168.1.C │<──────────────╯
     ╭──────UDP datagram─────╮        ├──────Encrypted─────┤                                 │dst: 192.168.1.B │
     │   src: 1.1.1.B:51820  │        │ ╭─────ICMP───────╮ │                                 ╰─────────────────╯
     │   dst: 1.1.1.A:xxxxx  │        │ │src: 10.0.0.A   │ │                                           │
     ╰──┬─────WG Header────┬─╯        │ │dst: 192.168.1.C│ │                                           │
      ╭─╯  Received Index  ╰╮         ╰─┴────────────────┴─╯                    ╭─────────╮            │
      │    Counter          │                                   ╭──────DNAT─────┴╮ Node B │            │
      ├───────Encrypted─────┤                   ╭───────wg0─────┴╮ ╭─╴192.168.1.B│        <─────11─────╯
      │  ╭─────ICMP───────╮ │<────via eth0─────╴│Encrypt into UDP│ ╰─>10.0.0.A   ├────────╯
      │  │src: 192.168.1.C│ │                ╭──┴──ICMP Reply─╮  ├───────────────╯
      │  │dst: 10.0.0.A   │ │                │src: 192.168.1.C│──╯
      ╰──┴────────────────┴─╯                │dst: 10.0.0.A   │
                                             ╰────────────────╯



```

和 [WireGuard 数据中继（示意图）](#wireguard-数据中继-示意图) 做对比，可以看到前面 7 个步骤都是一致的，从步骤
8 开始有区别：

8.  进入数据转发流程

    内核收到 WireGuard 解密后的 ICMP 包之后，发现这个包不是发给本机的，于是进入转发流程（前提是打开了 ip_forward）。

    首先仍然是内核查找路由表：

    -   发现目的 IP `192.168.1.C` 应该由 eth0 发送
    -   发送前需要做 `SNAT`, 将源地址从 `10.0.0.A` 转换为本机的 LAN 地址 `192.168.1.B`
    -   再通过 eth0 接口发送给 node C

9.  Node C 收到 ICMP 包后，内核协议栈按照正常流程生成 ICMP Echo Reply。

10. ICMP Echo Reply 经由 eth0 发往 node B。

11. Node B 收到 Reply, 需要先做 DNAT 将目的地址转换回 `10.0.0.A`, 后续步骤就和
    WireGuard 数据中继的流程一致了。


## 方案实施 {#方案实施}

有了上述理论基础，要实现文章开头的 WireGuard 星型拓扑结构就很简单了。

WireGuard 在不同平台上的安装都比较简单，网上也很多相关介绍，这里略去不提，直接给出各个节点的配置说明。

WireGuard 的配置文件如果没有特殊说明，路径统一为 `/etc/wireguard/wg0.conf` 。
macOS 和 iOS/Android 比较特殊，有 UI 界面可以直接配置（Windows 没用过，想必应该也有界面可以配置）。


### 打开 IP 转发 {#打开-ip-转发}

需要为如下节点打开 IP 转发功能：

-   router
-   home-gw
-   office-gw

打开步骤如下：

-   修改 `/etc/syslog.conf` 文件，增加如下内容：
    ```cfg
    net.ipv4.ip_forward = 1
    ```
-   运行命令：
    ```shell
    sysctl -p
    ```

> 💡为何要为 router, home-gw 和 office-gw 打开 IP 转发？
>
> 如果不打开 IP 转发，服务器在发现目标地址不属于自己的 packet 时，会直接丢弃。
>
> 而这几个节点分别作为“星型”网络的中心节点，homelab 内网的网关节点，以及 office
> 内网的网关节点，承担着为所有的下级节点中继数据的功能，所以必须打开这个功能。


### router 配置 {#router-配置}

router wg0.conf 配置：

```cfg
[Interface]
Address = 10.9.0.1/32
PrivateKey = <private key>

# home-gw
[Peer]
PublicKey = xxx                 # home-gw's public key
# 在发送报文时，请将目标地址符合 AllowedIPs 配置的报文，发给此 Peer
# 在收到报文时，允许接收源地址符合 AllowedIPs 配置的报文
AllowedIPs = 10.9.0.2/32, 192.168.1.0/24 # 允许将报文转发给 homelab 网络

# office-gw
[Peer]
PublicKey = xxx                          # office-gw's public key
AllowedIPs = 10.9.0.3/32, 172.19.50.0/24 # 允许将报文转发给 office 网络

# MacBook
[Peer]
PublicKey = xxx                 # MacBook's public key
AllowedIPs = 10.9.9.1/32

# Cellphone
[Peer]
PublicKey = xxx                 # Cellphone's public key
AllowedIPs = 10.9.9.2/32
```


### home-gw 配置 {#home-gw-配置}

home-gw wg0.conf 配置：

```cfg
[Interface]
Address = 10.9.0.2/32
PrivateKey = <private key>
# 将目标地址不是本机的数据包通过物理网卡 eth0 转发出去，同时做一个地址伪装 (SNAT)
# 如果你的物理网卡不是 eth0，请修改成实际物理网卡接口
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# router
[Peer]
PublicKey = xxx                 # router's public key
AllowedIPs = 10.9.9.0/24        # 和移动端设备（MacBook/Cellphone）互通
```


### office-gw 配置 {#office-gw-配置}

office-gw wg0.conf 配置：

```cfg
[Interface]
Address = 10.9.0.3/32
PrivateKey = <private key>
# 将目标地址不是本机的数据包通过物理网卡 eth0 转发出去，同时做一个地址伪装 (SNAT)
# 如果你的物理网卡不是 eth0，请修改成实际物理网卡接口
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# router
[Peer]
PublicKey = xxx             # router's public key
AllowedIPs = 10.9.9.0/24    # 和移动端设备（MacBook/Cellphone）互通
```


### MacBook 配置 {#macbook-配置}

MacBook WireGuard 配置：

```cfg
[Interface]
Address = 10.9.9.1/32
PrivateKey = <private key>

# router
[Peer]
PublicKey = xxx                                          # router's public key
AllowedIPs = 10.9.0.0/16, 192.168.1.0/24, 172.19.50.0/24 # 和所有设备互通
```


### Cellphone 配置 {#cellphone-配置}

Cellphone WireGuard 配置和 MacBook 配置类似：

```cfg
[Interface]
Address = 10.9.9.2/32
PrivateKey = <private key>

# router
[Peer]
PublicKey = xxx                                          # router's public key
AllowedIPs = 10.9.0.0/16, 192.168.1.0/24, 172.19.50.0/24 # 和所有设备互通
```


## 后记：网关自动转发的替代方案 {#后记-网关自动转发的替代方案}

如果你不想让 home-gw/office-gw 无脑转发所有通向内网的流量，也可以按需手动转发。

```artist
                                        +--------------------------------------+
                                        |            192.168.1.0/24            |
                                        |                                      |
+---------+          +--------+         +----------+        +--------------+   |
| MacBook |--------->| router |-------->| home-gw  |------->| home-server  |   |
|         |          |        |         | 10.9.0.2 |        | 192.168.1.10 |   |
+---------+          +--------+         +----------+        +--------------+   |
                                        |                                      |
                                        +--------------------------------------+
```

如上图所示，假设现在 homelab LAN 中有一台服务器 home-server, 目标是 MacBook 可以直接访问 home-server （例如，可以 ssh 登录 home-server）。

注意，这里有几个限制条件：

-   home-server 本身并未接入 WireGuard。
-   home-gw 未开启 iptables 转发（参考 [home-gw 配置](#home-gw-配置)）。

    如果开启了 iptables 转发，就是走的无脑转发的方案，不需要其他操作，MacBook 直接就能访问 home-server。
-   home-gw 上面安装有 sshd server, 且 MacBook 有 ssh 登录权限。

下面我们来实现按需手动转发的方案。

1.  在 MacBook 上 ssh 登录 home-gw
2.  在第一步建立的 ssh 会话中，利用 ncat 打开通向 192.168.1.10:22 的转发隧道：
    ```shell
    ncat -v --sh-exec 'ncat -i 5m -v 192.168.1.10 22' -l 2222 --keep-open
    ```
    其实也可以改成直接利用 ssh 来进行端口转发，但是 ssh 转发不太稳定，很容易就断了。 ncat 转发实测十分的稳定。
3.  现在可以在 MacBook 上通过 ssh 登录 home-server 了：
    ```shell
    ssh -p 2222 10.9.0.2
    ```
    在第二步中，我们将 home-gw 的 2222 端口映射到了 home-server 的 22 端口上，因此，上面的命令实际上会连接到 home-server 的 22 端口上。
