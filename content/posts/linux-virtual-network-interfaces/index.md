---
title: "Linux 中的虚拟网络接口"
date: 2021-08-14T11:13:03+08:00
draft: false

resources:
- name: "featured-image"
  src: "linux-network.jpg"

tags: ["Linux", "网络", "虚拟化", "容器"]
categories: ["技术"]
---

>本文用到的字符画工具：[vscode-asciiflow2](https://github.com/zenghongtu/vscode-asciiflow2)

>注意: 本文中使用 `ip` 命令创建或修改的任何网络配置，都是未持久化的，主机重启即消失。

Linux 具有强大的虚拟网络能力，这也是 openstack 网络、docker 容器网络以及 kubernetes 网络等虚拟网络的基础。

这里介绍 Linux 常用的虚拟网络接口类型：TUN/TAP、bridge、veth、ipvlan/macvlan、vlan 以及 vxlan/geneve.

## 一、tun/tap 虚拟网络接口

tun/tap 是操作系统内核中的虚拟网络设备，他们为用户层程序提供数据的接收与传输。

普通的物理网络接口如 eth0，它的两端分别是内核协议栈和外面的物理网络。

而对于 TUN/TAP 虚拟接口如 tun0，它的一端一定是连接的用户层程序，另一端则视配置方式的不同而变化，可以直连内核协议栈，也可以是某个 bridge（后面会介绍）。
Linux 通过内核模块 TUN 提供 tun/tap 功能，该模块提供了一个设备接口 `/dev/net/tun` 供用户层程序读写，用户层程序通过 `/dev/net/tun` 读写主机内核协议栈的数据。

```
> modinfo tun
filename:       /lib/modules/5.13.6-1-default/kernel/drivers/net/tun.ko.xz
alias:          devname:net/tun
alias:          char-major-10-200
license:        GPL
author:         (C) 1999-2004 Max Krasnyansky <maxk@qualcomm.com>
description:    Universal TUN/TAP device driver
...

> ls /dev/net/tun
/dev/net/tun
```

一个 TUN 设备的示例图如下：

```
             
+----------------------------------------------------------------------+
|                                                                      |
|  +--------------------+      +--------------------+                  |
|  | User Application A |      | User Application B +<-----+           |
|  +------------+-------+      +-------+------------+      |           |
|               | 1                    | 5                 |           |
|...............+......................+...................|...........|
|               ↓                      ↓                   |           |
|         +----------+           +----------+              |           |
|         | socket A |           | socket B |              |           |
|         +-------+--+           +--+-------+              |           |
|                 | 2               | 6                    |           |
|.................+.................+......................|...........|
|                 ↓                 ↓                      |           |
|             +------------------------+          +--------+-------+   |
|             | Network Protocol Stack |          |  /dev/net/tun  |   |
|             +--+-------------------+-+          +--------+-------+   |
|                | 7                 | 3                   ^           |
|................+...................+.....................|...........|
|                ↓                   ↓                     |           |
|        +----------------+    +----------------+        4 |           |
|        |      eth0      |    |      tun0      |          |           |
|        +-------+--------+    +-----+----------+          |           |
|    10.32.0.11  |                   |   192.168.3.11      |           |
|                | 8                 +---------------------+           |
|                |                                                     |
+----------------+-----------------------------------------------------+
                 ↓
         Physical Network
```

因为 TUN/TAP 设备的一端是内核协议栈，显然流入 tun0 的数据包是先经过本地的路由规则匹配的。

路由匹配成功，数据包被发送到 tun0 后，tun0 发现另一端是通过 `/dev/net/tun` 连接到应用程序 B，就会将数据丢给应用程序 B。

应用程序对数据包进行处理后，可能会构造新的数据包，通过物理网卡发送出去。比如常见的 VPN 程序就是把原来的数据包封装/加密一遍，再发送给 VPN 服务器。

### C 语言编程测试 TUN 设备

为了使用 tun/tap 设备，用户层程序需要通过系统调用打开 `/dev/net/tun` 获得一个读写该设备的文件描述符(FD)，并且调用 `ioctl()` 向内核注册一个 TUN 或 TAP 类型的虚拟网卡(实例化一个 tun/tap 设备)，其名称可能是 `tun0/tap0` 等。

此后，用户程序可以通过该 TUN/TAP 虚拟网卡与主机内核协议栈（或者其他网络设备）交互。当用户层程序关闭后，其注册的 TUN/TAP 虚拟网卡以及自动生成的路由表相关条目都会被内核释放。

可以把用户层程序看做是网络上另一台主机，他们通过 tun/tap 虚拟网卡相连。

一个简单的 C 程序示例如下，它每次收到数据后，都只单纯地打印一下收到的字节数：

```c
#include <linux/if.h>
#include <linux/if_tun.h>

#include <sys/ioctl.h>

#include <fcntl.h>
#include <string.h>

#include <unistd.h>
#include<stdlib.h>
#include<stdio.h>

int tun_alloc(int flags)
{

    struct ifreq ifr;
    int fd, err;
    char *clonedev = "/dev/net/tun";

    // 打开 tun 文件，获得 fd
    if ((fd = open(clonedev, O_RDWR)) < 0) {
        return fd;
    }

    memset(&ifr, 0, sizeof(ifr));
    ifr.ifr_flags = flags;

    // 向内核注册一个 TUN 网卡，并与前面拿到的 fd 关联起来
    // 程序关闭时，注册的 tun 网卡及自动生成的相关路由策略，会被自动释放
    if ((err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0) {
        close(fd);
        return err;
    }

    printf("Open tun/tap device: %s for reading...\n", ifr.ifr_name);

    return fd;
}

int main()
{

    int tun_fd, nread;
    char buffer[1500];

    /* Flags: IFF_TUN   - TUN device (no Ethernet headers)
     *        IFF_TAP   - TAP device
     *        IFF_NO_PI - Do not provide packet information
     */
    tun_fd = tun_alloc(IFF_TUN | IFF_NO_PI);

    if (tun_fd < 0) {
        perror("Allocating interface");
        exit(1);
    }

    while (1) {
        nread = read(tun_fd, buffer, sizeof(buffer));
        if (nread < 0) {
            perror("Reading from interface");
            close(tun_fd);
            exit(1);
        }

        printf("Read %d bytes from tun/tap device\n", nread);
    }
    return 0;
}
```

接下来开启三个终端窗口来测试上述程序，分别运行上面的 tun 程序、tcpdump 和 iproute2 指令。

首先通过编译运行上述 c 程序，程序会阻塞住，等待数据到达：
```
# 编译，请忽略部分 warning
> gcc mytun.c -o mytun

# 创建并监听 tun 设备需要 root 权限
> sudo mytun 
Open tun/tap device: tun0 for reading...
```

现在使用 iproute2 查看下链路层设备：

```
# 能发现最后面有列出名为 tun0 的接口，但是状态为 down
❯ ip addr ls
......
3: wlp4s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether c0:3c:59:36:a4:16 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.228/24 brd 192.168.31.255 scope global dynamic noprefixroute wlp4s0
       valid_lft 41010sec preferred_lft 41010sec
    inet6 fe80::4ab0:130f:423b:5d37/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
7: tun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 500
    link/none 

# 为 tun0 设置 ip 地址，注意不要和其他接口在同一网段，会导致路由冲突
> sudo ip addr add 172.21.22.23/24 dev tun0
# 启动 tun0 这个接口，这一步会自动向路由表中添加将 172.21.22.23/24 路由到 tun0 的策略
> sudo ip link set tun0 up
#确认上一步添加的路由策略是否存在
❯ ip route ls
default via 192.168.31.1 dev wlp4s0 proto dhcp metric 600 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
172.21.22.0/24 dev tun0 proto kernel scope link src 172.21.22.23 
192.168.31.0/24 dev wlp4s0 proto kernel scope link src 192.168.31.228 metric 600 

# 此时再查看接口，发现 tun0 状态为 unknown
> ip addr ls
......
8: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 172.21.22.23/24 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::3d52:49b5:1cf3:38fd/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever

# 使用 tcpdump 尝试抓下 tun0 的数据，会阻塞在这里，等待数据到达
> tcpdump -i tun0
```

现在再启动第三个窗口发点数据给 tun0，持续观察前面 `tcpdump` 和 `mytun` 的日志:
```
# 直接 ping tun0 的地址，貌似有问题，数据没进 mytun 程序，而且还有响应
❯ ping -c 4 172.21.22.23
PING 172.21.22.23 (172.21.22.23) 56(84) bytes of data.
64 bytes from 172.21.22.23: icmp_seq=1 ttl=64 time=0.167 ms
64 bytes from 172.21.22.23: icmp_seq=2 ttl=64 time=0.180 ms
64 bytes from 172.21.22.23: icmp_seq=3 ttl=64 time=0.126 ms
64 bytes from 172.21.22.23: icmp_seq=4 ttl=64 time=0.141 ms

--- 172.21.22.23 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3060ms
rtt min/avg/max/mdev = 0.126/0.153/0.180/0.021 ms

# 但是 ping 该网段下的其他地址，流量就会被转发给 mytun 程序，因为 mytun 啥数据也没回，自然丢包率 100%
# tcpdump 和 mytun 都会打印出相关日志
❯ ping -c 4 172.21.22.26
PING 172.21.22.26 (172.21.22.26) 56(84) bytes of data.

--- 172.21.22.26 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3055ms
```

下面给出 mytun 的输出：

```
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device
```

以及 tcpdump 的输出：

```
00:22:03.622684 IP (tos 0x0, ttl 64, id 37341, offset 0, flags [DF], proto ICMP (1), length 84)
    172.21.22.23 > 172.21.22.26: ICMP echo request, id 11, seq 1, length 64
00:22:04.633394 IP (tos 0x0, ttl 64, id 37522, offset 0, flags [DF], proto ICMP (1), length 84)
    172.21.22.23 > 172.21.22.26: ICMP echo request, id 11, seq 2, length 64
00:22:05.653356 IP (tos 0x0, ttl 64, id 37637, offset 0, flags [DF], proto ICMP (1), length 84)
    172.21.22.23 > 172.21.22.26: ICMP echo request, id 11, seq 3, length 64
00:22:06.677341 IP (tos 0x0, ttl 64, id 37667, offset 0, flags [DF], proto ICMP (1), length 84)
    172.21.22.23 > 172.21.22.26: ICMP echo request, id 11, seq 4, length 64
```

更复杂的 tun 程序，可以参考

- [simpletun](https://github.com/gregnietsky/simpletun)
- [marywangran/simpletun](https://github.com/marywangran/simpletun)
- [tun go 语言版](https://github.com/marywangran/gotun-tunnel/blob/main/tun/tun.go)

### TUN 与 TAP 的区别

TUN 和 TAP 的区别在于工作的网络层次不同，用户程序通过 TUN 设备只能读写网络层的 IP 数据包，而 TAP 设备则支持读写链路层的数据包（通常是以太网数据包，带有 Ethernet headers）。

TUN 与 TAP 的关系，就类似于 socket 和 raw socket.

TUN/TAP 应用最多的场景是 VPN 代理，比如:

1. [clash](https://github.com/ryan4yin/clash): 一个支持各种规则的隧道，也支持 TUN 模式
2. [tun2socks](https://github.com/xjasonlyu/tun2socks/wiki): 一个全局透明代理，和 VPN 的工作模式一样，它通过创建虚拟网卡+修改路由表，在第三层网络层代理系统流量。


## 二、veth

veth 接口总是成对出现，一对 veth 接口就类似一根网线，从一端进来的数据会从另一端出去。

同时 veth 又是一个虚拟网络接口，因此它和 TUN/TAP 或者其他物理网络接口一样，也都能配置 mac/ip 地址（但是并不是一定得配 mac/ip 地址）。

其主要作用就是连接不同的网络，比如在容器网络中，用于将容器的 namespace 与 root namespace 的网桥 br0 相连。
容器网络中，容器侧的 veth 自身设置了 ip/mac 地址并被重命名为 eth0，作为容器的网络接口使用，而主机侧的 veth 则直接连接在 docker0/br0 上面。

使用 veth 实现容器网络，需要结合下一小节介绍的 bridge，在下一小节将给出容器网络结构图。

## 三、bridge

Linux Bridge 是工作在链路层的网络交换机，由 Linux 内核模块 `brige` 提供，它负责在所有连接到它的接口之间转发链路层数据包。

添加到 Bridge 上的设备被设置为只接受二层数据帧并且转发所有收到的数据包到 Bridge 中。
在 Bridge 中会进行一个类似物理交换机的查MAC端口映射表、转发、更新MAC端口映射表这样的处理逻辑，从而数据包可以被转发到另一个接口/丢弃/广播/发往上层协议栈，由此 Bridge 实现了数据转发的功能。

如果使用 tcpdump 在 Bridge 接口上抓包，可以抓到网桥上所有接口进出的包，因为这些数据包都要通过网桥进行转发。

与物理交换机不同的是，Bridge 本身可以设置 IP 地址，可以认为当使用 `brctl addbr br0` 新建一个 br0 网桥时，系统自动创建了一个同名的隐藏 `br0` 网络接口。`br0` 一旦设置 IP 地址，就意味着这个隐藏的 br0 接口可以作为路由接口设备，参与 IP 层的路由选择(可以使用 `route -n` 查看最后一列 `Iface`)。因此只有当 `br0` 设置 `IP` 地址时，Bridge 才有可能将数据包发往上层协议栈。

但被添加到 Bridge 上的网卡是不能配置 IP 地址的，他们工作在数据链路层，对路由系统不可见。


它常被用于在虚拟机、主机上不同的 namepsaces 之间转发数据。

### 虚拟机场景（桥接模式）

以 qemu-kvm 为例，在虚拟机的桥接模式下，qemu-kvm 会为每个虚拟机创建一个 tun/tap 虚拟网卡并连接到 br0 网桥。
虚拟机内部的网络接口 `eth0` 是 qemu-kvm 软件模拟的，实际上虚拟机内网络数据的收发都会被 qemu-kvm 转换成对 `/dev/net/tun` 的读写。

以发送数据为例，整个流程如下：

- 虚拟机发出去的数据包先到达 qemu-kvm 程序
- 数据被用户层程序 qemu-kvm 写入到 `/dev/net/tun`，到达 tap 设备
- tap 设备把数据传送到 br0 网桥
- br0 把数据交给 eth0 发送出去

整个流程跑完，数据包都不需要经过宿主机的协议栈，效率高。

```
+------------------------------------------------+-----------------------------------+-----------------------------------+
|                       Host                     |           VirtualMachine1         |           VirtualMachine2         |
|                                                |                                   |                                   |
|    +--------------------------------------+    |    +-------------------------+    |    +-------------------------+    |
|    |         Network Protocol Stack       |    |    |  Network Protocol Stack |    |    |  Network Protocol Stack |    |
|    +--------------------------------------+    |    +-------------------------+    |    +-------------------------+    |
|                       ↑                        |                ↑                  |                 ↑                 |
|.......................|........................|................|..................|.................|.................|
|                       ↓                        |                ↓                  |                 ↓                 |
|                  +--------+                    |            +-------+              |             +-------+             |
|                  | .3.101 |                    |            | .3.102|              |             | .3.103|             |
|     +------+     +--------+     +-------+      |            +-------+              |             +-------+             |
|     | eth0 |<--->|   br0  |<--->|tun/tap|      |            | eth0  |              |             | eth0  |             |
|     +------+     +--------+     +-------+      |            +-------+              |             +-------+             |
|         ↑             ↑             ↑      +--------+           ↑                  |                 ↑                 |
|         |             |             +------|qemu-kvm|-----------+                  |                 |                 |
|         |             ↓                    +--------+                              |                 |                 |
|         |         +-------+                    |                                   |                 |                 |
|         |         |tun/tap|                    |                                   |                 |                 |
|         |         +-------+                    |                                   |                 |                 |
|         |             ↑                        |            +--------+             |                 |                 |
|         |             +-------------------------------------|qemu-kvm|-------------|-----------------+                 |
|         |                                      |            +--------+             |                                   |
|         |                                      |                                   |                                   |
+---------|--------------------------------------+-----------------------------------+-----------------------------------+
          ↓
    Physical Network  (192.168.3.0/24)
```

### 跨 namespace 通信场景（容器网络，NAT 模式）

>docker/podman 提供的 bridge 网络模式，就是使用 veth+bridge+iptalbes 实现的。我会在下一篇文章详细介绍「容器网络」。

由于容器运行在自己单独的 network namespace 里面，所以和虚拟机一样，它们也都有自己单独的协议栈。

容器网络的结构和虚拟机差不多，但是它改用了 NAT 网络，并把 tun/tap 换成了 veth，导致 docker0 过来的数据，要先经过宿主机协议栈，然后才进入 veth 接口。

多了一层 NAT，以及多走了一层宿主机协议栈，都会导致性能下降。

示意图如下：

```
+-----------------------------------------------+-----------------------------------+-----------------------------------+
|                      Host                     |           Container 1             |           Container 2             |
|                                               |                                   |                                   |
|   +---------------------------------------+   |    +-------------------------+    |    +-------------------------+    |
|   |       Network Protocol Stack          |   |    |  Network Protocol Stack |    |    |  Network Protocol Stack |    |
|   +----+-------------+--------------------+   |    +-----------+-------------+    |    +------------+------------+    |
|        ^             ^                        |                ^                  |                 ^                 |
|........|.............|........................|................|..................|.................|.................|
|        v             v  ↓                     |                v                  |                 v                 |
|   +----+----+  +-----+------+                 |          +-----+-------+          |           +-----+-------+         |
|   | .31.101 |  | 172.17.0.1 |      +------+   |          | 172.17.0.2  |          |           |  172.17.0.3 |         |
|   +---------+  +-------------<---->+ veth |   |          +-------------+          |           +-------------+         |
|   |  eth0   |  |   docker0  |      +--+---+   |          | eth0(veth)  |          |           | eth0(veth)  |         |
|   +----+----+  +-----+------+         ^       |          +-----+-------+          |           +-----+-------+         |
|        ^             ^                |       |                ^                  |                 ^                 |
|        |             |                +------------------------+                  |                 |                 |
|        |             v                        |                                   |                 |                 |
|        |          +--+---+                    |                                   |                 |                 |
|        |          | veth |                    |                                   |                 |                 |
|        |          +--+---+                    |                                   |                 |                 |
|        |             ^                        |                                   |                 |                 |
|        |             +------------------------------------------------------------------------------+                 |
|        |                                      |                                   |                                   |
|        |                                      |                                   |                                   |
+-----------------------------------------------+-----------------------------------+-----------------------------------+
         v
    Physical Network  (192.168.31.0/24)
```

每创建一个新容器，都会在容器的 namespace 里新建一个 veth 接口并命令为 eth0，同时在主 namespace 创建一个 veth，将容器的 eth0 与 docker0 连接。

可以在容器中通过 iproute2 查看到， eth0 的接口类型为 `veth`：

```shell
❯ docker run -it --rm debian:buster bash
root@5facbe4ddc1e:/# ip --details addr ls
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0 minmtu 0 maxmtu 0 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
20: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 0 minmtu 68 maxmtu 65535 
    veth numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

同时在宿主机中能看到对应的 veth 设备是绑定到了 docker0 网桥的：

```shell
❯ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242fce99ef5       no              vethea4171a
```



## 四、macvlan

>目前 docker/podman 都支持使用 `--driver macvlan` 来使用 `macvlan` 创建容器网络。

>[Use macvlan networks - Docker Docs](https://docs.docker.com/network/macvlan/)

>参考文档：[linux 网络虚拟化： macvlan](https://cizixs.com/2017/02/14/network-virtualization-macvlan/)

macvlan 是比较新的 Linux 特性，需要内核版本 >= 3.9，它被用于在主机的网络接口（父接口）上配置多个虚拟子接口，这些子接口都拥有各自独立的 mac 地址，也可以配上 ip 地址进行通讯。

macvlan 下的虚拟机或者容器网络和主机在同一个网段中，共享同一个广播域。macvlan 和 bridge 比较相似，但因为它省去了 bridge 的存在，所以配置和调试起来比较简单，而且效率也相对高。除此之外，macvlan 自身也完美支持 VLAN。

如果希望容器或者虚拟机放在主机相同的网络中，享受已经存在网络栈的各种优势，可以考虑 macvlan。


## 五、ipvlan

>目前 docker 已支持使用 `--driver ipvlan` 来使用 `ipvlan` 创建容器网络，[podman 正计划支持](https://github.com/containers/podman/issues/10478).
>[Use ipvlan networks - Docker Docs](https://docs.docker.com/network/ipvlan/)
>[linux 网络虚拟化： ipvlan](https://cizixs.com/2017/02/17/network-virtualization-ipvlan/)

ipvlan 和 macvlan 的功能很类似，也是用于在主机的网络接口（父接口）上配置出多个虚拟的子接口。但不同的是，ipvlan 的各子接口没有独立的 mac 地址，它们和主机的父接口共享 mac 地址。

>因为 mac 地址共享，所以如果使用 DHCP，就要注意不能使用 mac 地址做 DHCP，需要额外配置唯一的 clientID.

如果你遇到以下的情况，请考虑使用 ipvlan：

- 父接口对 mac 地址数目有限制，或者在 mac 地址过多的情况下会造成严重的性能损失
- 工作在 802.11(wireless)无线网络中（macvlan 无法和无线网络共同工作）
- 希望搭建比较复杂的网络拓扑（不是简单的二层网络和 VLAN），比如要和 BGP 网络一起工作


## 六、vlan

vlan 即虚拟局域网，是一个链路层的广播域隔离技术，可以用于切分局域网，解决广播泛滥和安全性问题。被隔离的广播域之间需要上升到第三层才能完成通讯。

常用的企业路由器如 ER-X 基本都可以设置 vlan，Linux 也直接支持了 vlan.

以太网数据包有一个专门的字段提供给 vlan 使用，vlan 数据包会在该位置记录它的 VLAN ID，交换机通过该 ID 来区分不同的 VLAN，只将该以太网报文广播到该 ID 对应的 VLAN 中。


## 七、vxlan/geneve

>[rfc8926 - Geneve: Generic Network Virtualization Encapsulation](https://datatracker.ietf.org/doc/html/rfc8926)
>[rfc7348 - Virtual eXtensible Local Area Network (VXLAN)](https://datatracker.ietf.org/doc/html/rfc7348)

>[linux 上实现 vxlan 网络](https://cizixs.com/2017/09/28/linux-vxlan/)

在介绍 vxlan 前，先说明下两个名词的含义：

- **underlay 网络**：即物理网络
- **overlay 网络**：指在现有的物理网络之上构建的虚拟网络。其实就是一种隧道技术，将原生态的二层数据帧报文进行封装后通过隧道进行传输。

vxlan 与 geneve 都是 overlay 网络协议，它俩都是使用 UDP 包来封装链路层的以太网帧。

vxlan 在 2014 年标准化，而 geneve 在 2020 年底才通过草案阶段，目前尚未形成最终标准。但是目前 linux/cilium 都已经支持了 geneve.

geneve 相对 vxlan 最大的变化，是它更灵活——它的 header 长度是可变的。

目前所有 overlay 的跨主机容器网络方案，几乎都是基于 vxlan 实现的（例外：cilium 也支持 geneve）。

>我们在学习单机的容器网络时，不需要接触到 vxlan，但是在学习跨主机容器网络方案如 flannel/calico/cilium 时，那 vxlan(overlay) 及 BGP(underlay) 就不可避免地要接触了。

先介绍下 vxlan 的数据包结构：

![](/images/linux-virtual-interfaces/vxlan-frame.png "VXLAN 栈帧结构")

在创建 vxlan 的 vtep 虚拟设备时，我们需要手动设置图中的如下属性：

- VXLAN 目标端口：即接收方 vtep 使用的端口，这里 IANA 定义的端口是 4789，但是只有 calico 的 vxlan 模式默认使用该端口 calico，而 cilium/flannel 的默认端口都是 Linux 默认的 8472.
- VNID: 每个 VXLAN 网络接口都会被分配一个独立的 VNID

一个点对点的 vxlan 网络架构图如下:

![](/images/linux-virtual-interfaces/vxlan-architecture.gif "VXLAN 点对点网络架构")

可以看到每台虚拟机 VM 都会被分配一个唯一的 VNID，然后两台物理机之间通过 VTEP 虚拟网络设备建立了 VXLAN 隧道，所有 VXLAN 网络中的虚拟机，都通过 VTEP 来互相通信。

有了上面这些知识，我们就可以通过如下命令在两台 Linux 机器间建立一个**点对点的 VXLAN 隧道**：

```shell
# 在主机 A 上创建 VTEP 设备 vxlan0
# 与另一个 vtep 接口 B（192.168.8.101）建立隧道
# 将 vxlan0 自身的 IP 地址设为 192.168.8.100
# 使用的 VXLAN 目标端口为 4789(IANA 标准)
ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    remote 192.168.8.101 \
    local 192.168.8.100 \
    dev enp0s8
# 为我们的 VXLAN 网络设置虚拟网段，vxlan0 就是默认网关
ip addr add 10.20.1.2/24 dev vxlan0
# 启用我们的 vxlan0 设备，这会自动生成路由规则
ip link set vxlan0 up

# 现在在主机 B 上运行如下命令，同样创建一个 VTEP 设备 vxlan0，remote 和 local 的 ip 与前面用的命令刚好相反。
# 注意 VNID 和 dstport 必须和前面完全一致
ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    remote 192.168.8.100 \
    local 192.168.8.101 \
    dev enp0s8
# 为我们的 VXLAN 网络设置虚拟网段，vxlan0 就是默认网关
ip addr add 10.20.1.3/24 dev vxlan0
ip link set vxlan0 up

# 到这里，两台机器就完成连接，可以通信了。可以在主机 B 上 ping 10.20.1.2 试试，应该能收到主机 A 的回应。
ping 10.20.1.2
```

点对点的 vxlan 隧道实际用处不大，如果集群中的每个节点都互相建 vxlan 隧道，代价太高了。

一种更好的方式，是使用 **「组播模式」的 vxlan 隧道**，这种模式下一个 vtep 可以一次与组内的所有 vtep 建立隧道。
示例命令如下（这里略过了如何设置组播地址 `239.1.1.1` 的信息）：

```shell
ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    group 239.1.1.1 \
    dev enp0s8 
ip addr add 10.20.1.2/24 dev vxlan0
ip link set vxlan0 up
```

可以看到，只需要简单地把 local_ip/remote_ip 替换成一个组播地址就行。组播功能会将收到的数据包发送给组里的所有 vtep 接口，但是只有 VNID 能对上的 vtep 会处理该报文，其他 vtep 会直接丢弃数据。

接下来，为了能让所有的虚拟机/容器，都通过 vtep 通信，我们再添加一个 bridge 网络，充当 vtep 与容器间的交换机。架构如下：

![](/images/linux-virtual-interfaces/linux-vxlan-with-bridge.jpg "VXLAN 多播网络架构")


使用 ip 命令创建网桥、网络名字空间、veth pairs 组成上图中的容器网络：

```shell
# 创建 br0 并将 vxlan0 绑定上去
ip link add br0 type bridge
ip link set vxlan0 master bridge
ip link set vxlan0 up
ip link set br0 up

# 模拟将容器加入到网桥中的操作
ip netns add container1

## 创建 veth pair，并把一端加到网桥上
ip link add veth0 type veth peer name veth1
ip link set dev veth0 master br0
ip link set dev veth0 up

## 配置容器内部的网络和 IP
ip link set dev veth1 netns container1
ip netns exec container1 ip link set lo up

ip netns exec container1 ip link set veth1 name eth0
ip netns exec container1 ip addr add 10.20.1.11/24 dev eth0
ip netns exec container1 ip link set eth0 up
```

然后在另一台机器上做同样的操作，并创建新容器，两个容器就能通过 vxlan 通信啦~

### 比组播更高效的 vxlan 实现

组播最大的问题在于，因为它不知道数据的目的地，所以每个 vtep 都发了一份。如果每次发数据时，如果能够精确到对应的 vtep，就能节约大量资源。

另一个问题是 ARP 查询也会被组播，要知道 vxlan 本身就是个 overlay 网络，ARP 的成本也很高。

上述问题都可以通过一个中心化的注册中心（如 etcd）来解决，所有容器、网络的注册与变更，都写入到这个注册中心，然后由程序自动维护 vtep 之间的隧道、fdb 表及 ARP 表.

## 八、虚拟网络接口的速率

Loopback 和本章讲到的其他虚拟网络接口一样，都是一种软件模拟的网络设备。
他们的速率是不是也像物理链路一样，存在链路层（比如以太网）的带宽限制呢？

比如目前很多老旧的网络设备，都是只支持到百兆以太网，这就决定了它的带宽上限。
即使是较新的设备，目前基本也都只支持到千兆，也就是 1GbE 以太网标准，那本文提到的虚拟网络接口单纯在本机内部通信，是否也存在这样的制约呢？是否也只能跑到 1GbE?

使用 ethtool 检查：

```
# docker 容器的 veth 接口速率
> ethtool vethe899841 | grep Speed
        Speed: 10000Mb/s

# 网桥看起来没有固定的速率
> ethtool docker0 | grep Speed
        Speed: Unknown!

# tun0 设备的默认速率貌似是 10Mb/s ?
> ethtool tun0 | grep Speed
        Speed: 10Mb/s

# 此外 ethtool 无法检查 lo 以及 wifi 的速率
```

### 网络性能实测

接下来实际测试一下，先给出机器参数：

```
❯ cat /etc/os-release 
NAME="openSUSE Tumbleweed"
# VERSION="20210810"
...

❯ uname -a
Linux legion-book 5.13.8-1-default #1 SMP Thu Aug 5 08:56:22 UTC 2021 (967c6a8) x86_64 x86_64 x86_64 GNU/Linux


❯ lscpu
Architecture:                    x86_64
CPU(s):                          16
Model name:                      AMD Ryzen 7 5800H with Radeon Graphics
...

# 内存，单位 MB
❯ free -m
               total        used        free      shared  buff/cache   available
Mem:           27929        4482       17324         249        6122       22797
Swap:           2048           0        2048
```

使用 iperf3 测试：

```shell
# 启动服务端
iperf3 -s

-------------
# 新窗口启动客户端，通过 loopback 接口访问 iperf3-server，大概 49Gb/s
❯ iperf3 -c 127.0.0.1
Connecting to host 127.0.0.1, port 5201
[  5] local 127.0.0.1 port 48656 connected to 127.0.0.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  4.46 GBytes  38.3 Gbits/sec    0   1.62 MBytes       
[  5]   1.00-2.00   sec  4.61 GBytes  39.6 Gbits/sec    0   1.62 MBytes       
[  5]   2.00-3.00   sec  5.69 GBytes  48.9 Gbits/sec    0   1.62 MBytes       
[  5]   3.00-4.00   sec  6.11 GBytes  52.5 Gbits/sec    0   1.62 MBytes       
[  5]   4.00-5.00   sec  6.04 GBytes  51.9 Gbits/sec    0   1.62 MBytes       
[  5]   5.00-6.00   sec  6.05 GBytes  52.0 Gbits/sec    0   1.62 MBytes       
[  5]   6.00-7.00   sec  6.01 GBytes  51.6 Gbits/sec    0   1.62 MBytes       
[  5]   7.00-8.00   sec  6.05 GBytes  52.0 Gbits/sec    0   1.62 MBytes       
[  5]   8.00-9.00   sec  6.34 GBytes  54.5 Gbits/sec    0   1.62 MBytes       
[  5]   9.00-10.00  sec  5.91 GBytes  50.8 Gbits/sec    0   1.62 MBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  57.3 GBytes  49.2 Gbits/sec    0             sender
[  5]   0.00-10.00  sec  57.3 GBytes  49.2 Gbits/sec                  receiver

# 客户端通过 wlp4s0 wifi 网卡(192.168.31.228)访问 iperf3-server，实际还是走的本机，但是速度要比 loopback 快一点，可能是默认设置的问题
❯ iperf3 -c 192.168.31.228
Connecting to host 192.168.31.228, port 5201
[  5] local 192.168.31.228 port 43430 connected to 192.168.31.228 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  5.12 GBytes  43.9 Gbits/sec    0   1.25 MBytes       
[  5]   1.00-2.00   sec  5.29 GBytes  45.5 Gbits/sec    0   1.25 MBytes       
[  5]   2.00-3.00   sec  5.92 GBytes  50.9 Gbits/sec    0   1.25 MBytes       
[  5]   3.00-4.00   sec  6.00 GBytes  51.5 Gbits/sec    0   1.25 MBytes       
[  5]   4.00-5.00   sec  5.98 GBytes  51.4 Gbits/sec    0   1.25 MBytes       
[  5]   5.00-6.00   sec  6.05 GBytes  52.0 Gbits/sec    0   1.25 MBytes       
[  5]   6.00-7.00   sec  6.16 GBytes  52.9 Gbits/sec    0   1.25 MBytes       
[  5]   7.00-8.00   sec  6.08 GBytes  52.2 Gbits/sec    0   1.25 MBytes       
[  5]   8.00-9.00   sec  6.00 GBytes  51.6 Gbits/sec    0   1.25 MBytes       
[  5]   9.00-10.00  sec  6.01 GBytes  51.6 Gbits/sec    0   1.25 MBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  58.6 GBytes  50.3 Gbits/sec    0             sender
[  5]   0.00-10.00  sec  58.6 GBytes  50.3 Gbits/sec                  receiver

# 从容器中访问宿主机的 iperf3-server，速度几乎没区别
❯ docker run  -it --rm --name=iperf3-server networkstatic/iperf3 -c 192.168.31.228
Connecting to host 192.168.31.228, port 5201
[  5] local 172.17.0.2 port 43436 connected to 192.168.31.228 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  4.49 GBytes  38.5 Gbits/sec    0    403 KBytes       
[  5]   1.00-2.00   sec  5.31 GBytes  45.6 Gbits/sec    0    544 KBytes       
[  5]   2.00-3.00   sec  6.14 GBytes  52.8 Gbits/sec    0    544 KBytes       
[  5]   3.00-4.00   sec  5.85 GBytes  50.3 Gbits/sec    0    544 KBytes       
[  5]   4.00-5.00   sec  6.14 GBytes  52.7 Gbits/sec    0    544 KBytes       
[  5]   5.00-6.00   sec  5.99 GBytes  51.5 Gbits/sec    0    544 KBytes       
[  5]   6.00-7.00   sec  5.86 GBytes  50.4 Gbits/sec    0    544 KBytes       
[  5]   7.00-8.00   sec  6.05 GBytes  52.0 Gbits/sec    0    544 KBytes       
[  5]   8.00-9.00   sec  5.99 GBytes  51.5 Gbits/sec    0    544 KBytes       
[  5]   9.00-10.00  sec  6.12 GBytes  52.5 Gbits/sec    0    544 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  58.0 GBytes  49.8 Gbits/sec    0             sender
[  5]   0.00-10.00  sec  58.0 GBytes  49.8 Gbits/sec                  receiver
```

把 iperf3-server 跑在容器里再测一遍：

```shell
# 在容器中启动 iperf3-server，并映射到宿主机端口 6201
> docker run  -it --rm --name=iperf3-server -p 6201:5201 networkstatic/iperf3 -s
> docker inspect --format "{{ .NetworkSettings.IPAddress }}" iperf3-server
172.17.0.2
-----------------------------
# 测试容器之间互访的速度，ip 为 iperf3-server 的容器 ip，速度要慢一些。
# 毕竟过了 veth -> veth -> docker0 -> veth -> veth 五层虚拟网络接口
❯ docker run  -it --rm networkstatic/iperf3 -c 172.17.0.2
Connecting to host 172.17.0.2, port 5201
[  5] local 172.17.0.3 port 40776 connected to 172.17.0.2 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  4.74 GBytes  40.7 Gbits/sec    0    600 KBytes       
[  5]   1.00-2.00   sec  4.48 GBytes  38.5 Gbits/sec    0    600 KBytes       
[  5]   2.00-3.00   sec  5.38 GBytes  46.2 Gbits/sec    0    600 KBytes       
[  5]   3.00-4.00   sec  5.39 GBytes  46.3 Gbits/sec    0    600 KBytes       
[  5]   4.00-5.00   sec  5.42 GBytes  46.6 Gbits/sec    0    600 KBytes       
[  5]   5.00-6.00   sec  5.39 GBytes  46.3 Gbits/sec    0    600 KBytes       
[  5]   6.00-7.00   sec  5.38 GBytes  46.2 Gbits/sec    0    635 KBytes       
[  5]   7.00-8.00   sec  5.37 GBytes  46.1 Gbits/sec    0    667 KBytes       
[  5]   8.00-9.00   sec  6.01 GBytes  51.7 Gbits/sec    0    735 KBytes       
[  5]   9.00-10.00  sec  5.74 GBytes  49.3 Gbits/sec    0    735 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  53.3 GBytes  45.8 Gbits/sec    0             sender
[  5]   0.00-10.00  sec  53.3 GBytes  45.8 Gbits/sec                  receiver

# 本机直接访问容器 ip，走的是 docker0 网桥，居然还挺快
❯ iperf3 -c 172.17.0.2
Connecting to host 172.17.0.2, port 5201
[  5] local 172.17.0.1 port 56486 connected to 172.17.0.2 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  5.01 GBytes  43.0 Gbits/sec    0    632 KBytes       
[  5]   1.00-2.00   sec  5.19 GBytes  44.6 Gbits/sec    0    703 KBytes       
[  5]   2.00-3.00   sec  6.46 GBytes  55.5 Gbits/sec    0    789 KBytes       
[  5]   3.00-4.00   sec  6.80 GBytes  58.4 Gbits/sec    0    789 KBytes       
[  5]   4.00-5.00   sec  6.82 GBytes  58.6 Gbits/sec    0    913 KBytes       
[  5]   5.00-6.00   sec  6.79 GBytes  58.3 Gbits/sec    0   1007 KBytes       
[  5]   6.00-7.00   sec  6.63 GBytes  56.9 Gbits/sec    0   1.04 MBytes       
[  5]   7.00-8.00   sec  6.75 GBytes  58.0 Gbits/sec    0   1.04 MBytes       
[  5]   8.00-9.00   sec  6.19 GBytes  53.2 Gbits/sec    0   1.04 MBytes       
[  5]   9.00-10.00  sec  6.55 GBytes  56.3 Gbits/sec    0   1.04 MBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  63.2 GBytes  54.3 Gbits/sec    0             sender
[  5]   0.00-10.00  sec  63.2 GBytes  54.3 Gbits/sec                  receiver

# 如果走本机 loopback 地址 + 容器端口映射，速度就慢了好多
# 或许是因为用 iptables 做端口映射导致的？
❯ iperf3 -c 127.0.0.1 -p 6201
Connecting to host 127.0.0.1, port 6201
[  5] local 127.0.0.1 port 48862 connected to 127.0.0.1 port 6201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  2.71 GBytes  23.3 Gbits/sec    0   1.37 MBytes       
[  5]   1.00-2.00   sec  3.64 GBytes  31.3 Gbits/sec    0   1.37 MBytes       
[  5]   2.00-3.00   sec  4.08 GBytes  35.0 Gbits/sec    0   1.37 MBytes       
[  5]   3.00-4.00   sec  3.49 GBytes  30.0 Gbits/sec    0   1.37 MBytes       
[  5]   4.00-5.00   sec  5.50 GBytes  47.2 Gbits/sec    2   1.37 MBytes       
[  5]   5.00-6.00   sec  4.06 GBytes  34.9 Gbits/sec    0   1.37 MBytes       
[  5]   6.00-7.00   sec  4.12 GBytes  35.4 Gbits/sec    0   1.37 MBytes       
[  5]   7.00-8.00   sec  3.99 GBytes  34.3 Gbits/sec    0   1.37 MBytes       
[  5]   8.00-9.00   sec  3.49 GBytes  30.0 Gbits/sec    0   1.37 MBytes       
[  5]   9.00-10.00  sec  5.51 GBytes  47.3 Gbits/sec    0   1.37 MBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  40.6 GBytes  34.9 Gbits/sec    2             sender
[  5]   0.00-10.00  sec  40.6 GBytes  34.9 Gbits/sec                  receiver

# 可走 wlp4s0 + 容器端口映射，速度也不慢啊
❯ iperf3 -c 192.168.31.228 -p 6201
Connecting to host 192.168.31.228, port 6201
[  5] local 192.168.31.228 port 54582 connected to 192.168.31.228 port 6201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  4.34 GBytes  37.3 Gbits/sec    0    795 KBytes       
[  5]   1.00-2.00   sec  4.78 GBytes  41.0 Gbits/sec    0    834 KBytes       
[  5]   2.00-3.00   sec  6.26 GBytes  53.7 Gbits/sec    0    834 KBytes       
[  5]   3.00-4.00   sec  6.30 GBytes  54.1 Gbits/sec    0    875 KBytes       
[  5]   4.00-5.00   sec  6.26 GBytes  53.8 Gbits/sec    0    875 KBytes       
[  5]   5.00-6.00   sec  5.75 GBytes  49.4 Gbits/sec    0    875 KBytes       
[  5]   6.00-7.00   sec  5.49 GBytes  47.2 Gbits/sec    0    966 KBytes       
[  5]   7.00-8.00   sec  5.72 GBytes  49.1 Gbits/sec    2    966 KBytes       
[  5]   8.00-9.00   sec  4.81 GBytes  41.3 Gbits/sec    2    966 KBytes       
[  5]   9.00-10.00  sec  5.98 GBytes  51.4 Gbits/sec    0    966 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  55.7 GBytes  47.8 Gbits/sec    4             sender
[  5]   0.00-10.00  sec  55.7 GBytes  47.8 Gbits/sec                  receiver
```

总的来看，loopback、bridge、veth 这几个接口基本上是没被限速的，veth 有查到上限为 10000Mb/s（10Gb/s） 感觉也是个假数字，
实际上测出来的数据基本在 35Gb/s 到 55Gb/s 之间，视情况浮动。

性能的变化和虚拟网络设备的链路和类型有关，或许和默认配置的区别也有关系。

另外 TUN 设备这里没有测，`ethtool tun0` 查到的值是比较离谱的 10Mb/s，但是感觉不太可能这么慢，有时间可以再测一波看看。

## 参考

- [Linux虚拟网络设备之tun/tap](https://segmentfault.com/a/1190000009249039)
- [Linux虚拟网络设备之veth](https://segmentfault.com/a/1190000009251098)
- [云计算底层技术-虚拟网络设备(Bridge,VLAN)](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-bridge-and-vlan/)
- [云计算底层技术-虚拟网络设备(tun/tap,veth)](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-tuntap-veth/)
- [Universal TUN/TAP device driver - Kernel Docs](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)
- [Tun/Tap interface tutorial](https://backreference.org/2010/03/26/tuntap-interface-tutorial/)
- [Linux Loopback performance with TCP_NODELAY enabled](https://stackoverflow.com/questions/5832308/linux-loopback-performance-with-tcp-nodelay-enabled)



