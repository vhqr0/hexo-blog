---
title: Go Tomato 点对点加密 VPN 及配套工具集
date: 2022-12-22 01:03:51
tags: Network
---

前言
====

最近又重写了 tomato，本节之后的内容是这个新轮子的 readme，下面写点感想。

[tomato_archive](https://github.com/vhqr0/tomato_archive) 保存了历代 tomato 的源码，基本上使用 c/c++/python 实现。tomato 项目最初的目的是写个类似于 trojan 的基于 TLS 的加密代理，后来功能不断丰富，要实现兼容 HTTP 代理、代理规则匹配等功能，并且添加端口转发（TCP/TLS 转换、UDP）和内网穿透（TCP、UDP）等工具。虽然这些功能已经基本实现，但是愈发感觉整个项目像个屎山而不断重构。

从实现原理上看，经过各种尝试后总结出 qtmd 加密和内网穿透，这些功能应该在网络层而非传输层实现。在传输层要考虑到加密/非加密转换、TCP/UDP 协议、是否要穿透内网等，排列组合一下就要实现多个工具，尤其是内网穿透要考虑到加密/非加密转换、认证和异常重启等问题，产生了大量重复工作和 BUG。而在网络层实现加密和内网穿透就变得十分简单，基于 tun 的 VPN 无连接无状态不容易出错，而且在传输层只需要实现支持规则转发的代理，如果需要暴露内网服务就再加上 TCP 或 UDP 转发，是最优雅的实现方式。

从实现工具上看，编程语言从c、c++ 到 python，编程方式从基本的套接字 API 和多线程、异步回调到协程。协程写起来更像进程、线程，流程清晰明确，而异步回调的一个流程则分布在状态机中，更加复杂，因此前一个版本的 tomato 已完全转向协程（asio、asyncio）。

然后是这一版本的开发历程。前一版本意识到两个问题：应该将加密和内网穿透在网络层实现（VPN），和应该使用日志库。

为什么不用 c++？事实上我一开始就选择了熟悉的 asio，日志库选择 glog。glog 还是不错的，我很快写完了配置解析，准备开始写服务的时候问题就出来了：clangd 补全协程代码需要 clang 支持协程 => clang 协程依赖 libc++ => libc++ 依赖特定版本的 libunwind => 与 glog 依赖默认版本的 libunwind 冲突。简而言之就是 debian 的 libc++ 与所有依赖默认版本的 libunwind 的包冲突，去网上搜索一下能看到很多人吐槽这个问题。总之，asio 是很优秀的网络库，但是仅有网络库是不够的：

1. 工具链对 c++20 协程的支持还不够成熟：例如 clang 和 gcc 的不一致性问题、libc++ 依赖性问题等。
2. 缺乏支持协程的同步机制：例如锁、队列，asio 协程仅可以异步等待 IO、超时和信号。
3. 缺乏流连接和数据报连接的统一抽象机制：例如一个流连接支持 send、recv 和 close，底层可以是 TCP 或 TLS。
4. 缺乏实现网络服务的基本库：例如日志库轮子一大堆，但要么太复杂（boost），要么有依赖问题（glog），而且 c++ 的 IO 流实在是丑陋不堪。

用 c++ 写一个简单的网络服务，大部分时间却花费在工具链、日志等与网络服务本身无关的地方。

为什么不用 python？asyncio 虽然有同步机制和流连接（TCP 和 TLS），但是没有完整封装异步 IO 调用，例如数据报套接字 API 和文件 IO。我在放弃用 c++ 实现后也尝试过用 python 实现，但是这次连配置解析都写不下去，深感 python 只适合写脚本。

最终，我现学了 go，然后只用了两天就实现了全部功能。go 的优点：

1. 简单易学，表达能力不弱于 python，而且 ls 更稳定，只是不能像 python 那样交互式开发。
2. 协程、同步机制和 IO 调用封装是完备的，并且实现了流连接、数据报连接的统一抽象机制。
3. 标准库简单易用，网络方面比 python 标准库更 battery included，尤其是密码库。

总而言之，go 非常适合开发网络服务，改用 go 后神清气爽，难怪同类型的 VPN 或代理都用 go 实现。

最后是这个项目的计划，目前能想到的就是基于 netfilter_queue 的 UDP 转 TCP 连接，用 QoS 更高的 TCP 连接代替 UDP 连接，可能只有实现了这个功能，tomato 才能取代常见的加密代理。

tomato
======

本项目包含三个独立的可执行文件：

1. `vpn` 是点对点加密 VPN 实现，用于建立隐秘的网络层链路。（Linux）
2. `ping` 是支持 IPv4/IPv6 的 Ping 实现，用于 `vpn` 的心跳检测。（Unix）
3. `proxy` 是支持规则的 HTTP 代理实现，配合 `vpn` 可取代常见的加密代理工具。

install
-------

1. 安装 `go` 和依赖库 `golang.org/x/sys` （`go get`）。
2. 在根目录执行 `make` ，三个可执行文件将生成到 `bin` 目录。
3. （Optional）`scripts/dlmerge.py` 将 [domain-list](https://github.com/v2fly/domain-list-community) 转化为 `proxy` 的规则文件。

vpn
===

`vpn` 是基于 Linux `tun/tap` 机制的类似于 `wireguard` 的点对点加密 VPN 实现。
VPN 两端节点是对等的，在各自的角度分别称为 local 和 peer，各自使用独立的密码加密网络层的报文。

PS. local 和 peer 是对等的，不能基于身份从一个密码中生成不同的密钥，因此 local 和 peer 应各自使用独立的密码。

basic
-----

以主机 A（192.168.8.130）连接主机 B（192.168.8.131）为例：

```sh
# 新建名为 tun0 的 tun 类型设备
ip tuntap add dev tun0 mode tun

# 开启 tun0
ip l set tun0 up

# （优化）配置 tun0 MTU
ip l set tun0 mtu 1400

# 配置 tun0 IPv4 地址
ip a add 10.0.1.1/30 dev tun0

# （优化）关闭 tun0 IPv6 协议栈
echo 1 > /proc/sys/net/ipv6/conf/tunl/disable_ipv6

# 运行 vpn
# la: local address
# lp: local password
# pa: peer address
# pp: peer password
# v:  verbose
vpn -la :5000 -lp pwd1 -pa 192.168.8.131:5000 -pp pwd2 -v
```

主机 B 也使用相同的方法连接主机 A，这样就建立了点对点加密网络 `10.0.1.1/30 <=> 10.0.1.2/30`。

PS. 通常将点对点网络的前缀长度配置为 /30，包含 4 个地址 0、1、2、3，分别表示网络本身、两端节点和广播地址。

PS. 报头长度：'((IPv4 . 20) (IPv6 . 40) (UDP . 8) (`vpn` . 44))，在使用时最好酌情配置 MTU，例如 `1500 - 40 - 8 - 44 = 1408`。

PS. IPv6 在点对点链路上相对 IPv4 没有任何优势：路由优化没有效果、报头更长、频繁发送无用报文等等，在使用时最好关闭 IPv6 协议栈。

gateway
-------

如果想要将主机 A 作为主机 B 的网关，两个主机都需要额外的配置。

主机 A 的额外配置：

```sh
# 开启 IPv4 路由转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 配置 NAT
iptables -t nat -A POSTROUTING -s 10.0.1.0/30 -o ens33 -j MASQUERADE
```

PS. `iptables` 的 `MASQUERADE` 即根据出口接口的地址自动配置 NAT。

主机 B 的额外配置：

```sh
# 添加主机 A 路由规则
ip r add 192.168.8.130 via x.x.x.x dev ens33

# 配置网关
ip r add default via 10.0.1.1/30 dev tun0

# 配置 DNS
vim /etc/resolv.conf
```

PS. 为确保发送给主机 A（192.168.8.130）的 UDP 报文可以正常转发，应在改变网关前添加正确的路由规则。

PS. 主机 B 应配置正确的 DNS 服务器地址，通常与主机 A 的 DNS 服务器地址相同。

dynamic update
--------------

VPN 两端节点不必都预先知道对端地址，其中一端可以等收到对端报文时动态更新地址，称为动态更新模式。

动态更新模式适用于无法固定 peer 的地址的场景，例如 NAT 地址转换或切换网络、设备。

继续前面的例子，如果主机 A 无法固定主机 B 的地址，`vpn` 的参数应做出如下改变：

```sh
vpn -la :5000 -lp pwd1 -d -pp pwd2 -v
```

注意到 `-pa 192.168.8.131:5000` 变为 `-d`。

更进一步，在一个具有公网地址的节点上通过动态更新模式建立两个连接并开启路由转发可以连接两个内网节点。

PS. 在动态更新模式下，如果未收到第一个报文或长时间（默认 60s）未收到报文，会造成连接丢失。

protocol
--------

TODO

ping
====

`ping` 和众所周知的同名命令功能一致：周期性（默认 30s）Ping 目标主机，并使用 `go` 的 `log` 输出结果。

在这里提供 `ping` 的目的是为使用动态更新模式的 `vpn` 创建的连接提供心跳检测。

继续前面的例子，如果主机 B 想要保证对端不会丢失连接：

```sh
ping -a 10.0.1.1
```

proxy
=====

`proxy` 是支持规则的 HTTP 代理实现，与 `vpn` 配合可以取代 `v2ray`、`trojan` 等加密代理工具。

与通过 `vpn` 将对端当作网关相比，我更推荐在连接两端部署 `proxy`，配置规则按需访问对端：

```sh
# 配置环境变量（Unix）
export ALL_PROXY=http://localhost:1080

# 运行 proxy
# a：local address
# f：forward address
# r：rule file
proxy -a :1080 -f 10.0.1.1:1080 -r rule.txt -v
```

这样，支持代理协议的应用，例如浏览器和包管理器等，将使用本地 `proxy`，后者根据规则决定如何处理请求，例如将请求转发给对端 `proxy`。

rule
----

`proxy` 对于一个请求有三个处理方式：

1. `block`：丢弃请求。
2. `direct`：在本地处理请求。
3. `forward`：将请求转发给对端，如果未设置对端则按照 `direct` 处理。

规则文件的格式如下：

```txt
# comment

block	ads.baidu.com
direct	baidu.com
forward	github.com
```

`proxy` 将忽略所有注释和空行，将所有规则按照 "direction	domain"（注意中间为 TAB） 解析。
如果一个域名出现多次，则以第一次为准，因此用户可以编辑文件头来覆盖后面的规则。

规则匹配将逐步匹配根域名直至匹配成功，例如匹配域名 www.baidu.com 时将依次匹配 www.baidu.com、baidu.com 和 com。
命令行参数 `-d DIRECTION` 决定默认的处理方式，默认为 `direct`。

PS. `proxy` 即使不用来做分流，也可以用来做广告过滤器。

PS. `script/dlmerge.py` 将 [domain-list](https://github.com/v2fly/domain-list-community) 转化为 `proxy` 的规则文件可能满足你的需求。

TODO
====

1. `udp2tcp`：基于 Linux 的 `netfilter_queue` 实现 UDP 连接转 TCP 连接，解决 UDP 和 TCP 的 QoS 不对等的问题。
