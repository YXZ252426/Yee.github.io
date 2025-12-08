---
title: 以太坊全节点部署：内网穿透篇
date: 2025-11-30 17:07:38
tags:
categories:
   - Ethereum is all you need
   - 全节点部署
---

让我们来到全节点部署的最后一章！
首先，我来描述一下问题
> 当时，我的全节点已经完成部署，但是，在同步的时候我发现执行层的节点（peers）数量只有个位数，与之对应的，共识层节点数量高达上百个，而Reth官方部署的全节点peers数量达到了一百个😨😨

这让我感到非常困惑，而且这也极大程度上影响了我的节点同步速率。

而最终的矛头则指向了一个——公网ip

让我们先来补足一些基础知识

## 以太坊节点的p2p通信
以太坊执行层使用的是 [devp2p 协议](https://ethereum.org/zh/developers/docs/networking-layer/#devp2p),作为一个p2p网络，其节点之间需要建立双向连接，其中Outbound（主动连出）是没有问题的，问题是Inbound（别人主动连你）需要公网可达 IP + 端口
![gossio](/images/gossip.png)
## 校园网络环境
校园网通常使用的是一种多层 NAT 架构，其特征是没有真正的公网 IPv4，外部无法直接访问内网设备，这就导致执行层节点几乎只能主动连出，无法稳定接收inbound P2P连接

## 总结
综上所属，我们的问题已经明了：以太坊执行层高度依赖可达的 P2P 网络，校园网环境下没有公网 IP，因此执行层无法接收 inbound 连接，peers 数量受限，同步效率下降。
而我们的解决方案也已经明了
> 给执行层“造”一个公网入口

## 解决方案--frp
什么是frp？

以下文字源于[Frp官网](https://gofrp.org/zh-cn/)
> frp 采用 C/S 模式，将服务端部署在具有公网 IP 的机器上，客户端部署在内网或防火墙内的机器上，通过访问暴露在服务器上的端口，反向代理到处于内网的服务。

他的基本架构是这样的
![frp](/images/frp.png)
而这是我们要实现的架构
- outbound: 内网节点主动连接到以太坊的节点
- inbound: 以太坊节点连接 AWS，AWS 通过 FRP 转发到内网
![frparch](/images/frp_arch.png)
## aws服务器配置和frp配置
理论可行，开始实操！！😎
首先，让我们注册云服务厂商的账号（这里我选择了亚马逊，考虑到以太坊节点大部分在海外，这样可能可以减少潜在的问题）。
来到[EC2](https://us-east-2.console.aws.amazon.com/ec2)并且创建实例并设置相应配置，这里是我的实例配置
- Amazon Machine Image (AMI): Ubuntu Server 22.04 LTS (HVM), SSD Volume Type
- Instance type: t3.small(2v CPU, 2GB Memory)
同时，不要忘记入站规则的设置，保证端口可以被外界访问到（这里还有一些配置是为了开放frp服务，一并在这里放出）
![aws_inbound](/images/aws_inbound.png)
### frp配置
关于这部分frp官网已经非常详细，就不多赘述
详见此处[frpdocs](https://gofrp.org/zh-cn/docs/setup/),同时建议配置systemd服务。
在这里就给出我自己的frpc配置文件
```toml
serverAddr = " "
serverPort = 7000

#[[proxies]]
#name = "reth-tcp"
#type = "tcp"
#localIP = "127.0.0.1"
#localPort = 30303
#remotePort = 30303

#[[proxies]]
#name = "reth-udp"
#type = "udp"
#localIP = "127.0.0.1"
#localPort = 30303
#remotePort = 30303
```

同时别忘了在docker compose中的node命令中配置
```bash
--nat extip:你的AWS公网IP
--port 30303
```
这样其他节点在 peer discovery 时，能正确拿到你的外部地址。
## 总结
经过这么一番折腾，我们就成功启动了frp服务，让我们的内网节点成为了可以p2p通信的真正意义上的以太坊节点，过一会，你就可以看到节点数量以火箭般上涨🚀🚀，而我的节点数量最终到达了60个
![peers](/images/grafana1.JPG)


