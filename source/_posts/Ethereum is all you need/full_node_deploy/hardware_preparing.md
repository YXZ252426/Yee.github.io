---
title: 以太坊全节点部署：硬件篇
date: 2025-11-30 17:05:38
tags:
categories:
   - Ethereum is all you need
   - 全节点部署
---
在真正了解全节点之前，我有各种疑惑：一个以太坊全节点会不会需要非常专业的设备才可以运行，因为像比特币就已经走向专业矿机（ASIC），存储以太坊的全部数据会不会需要非常大的硬盘（运行Reth目前只用了1TB），节点运行起来后会不会非常耗性能，非常吃内存（目前用了16G，但设置frp，peers数量增加后32GB的内存条已经不够），而在折腾完这些以后我对硬件设备也有了更深的了解。

先来看看我的设备情况吧：

## 🖥️ 主机硬件

| 项目 | 参数 |
| --- | --- |
| CPU | 13th Gen Intel(R) Core(TM) i5-13400F  |
| 内存 | Crucial/Ballistix 16GB DDR4 3200(CL16) ✖️ 2 |
| 硬盘 | ZhiTai TiPro 9000  |

那么，实际部署一个以太坊全节点有哪些设备要求呢
以下文字来自[以太坊官方文档](https://ethereum.org/zh/developers/docs/nodes-and-clients/run-a-node/#requirements)
> 硬件要求因客户端不同而异，但通常不是很高，因为节点只需保持同步即可。 不要将其与需要更多算力的挖矿混淆。 不过，功能更强大的硬件的确可以优化同步时间和性能。
在安装任何客户端之前，请确保计算机有足够的资源运行它。 你可以在下面找到最低要求和推荐要求。
硬件的瓶颈通常是磁盘空间。 同步以太坊区块链是一种高强度的输入/输出密集型操作，并且需要大量空间。 最好使用在同步完成后还有数百 GB 可用空间的固态硬盘 (SSD)。
推荐的规格要求:4 核以上高速 CPU,16GB 以上内存,2TB 以上高速固态硬盘,25 MB/秒以上带宽

甚至，你还可以在树莓派上跑一个以太坊全节点‼️🤯[详见](https://x.com/EthereumOnARM/status/1880258026676146618?s=20)

## 💾硬件选购
由上可见，对硬件的要求主要在于足够大的内存条和固态硬盘，那么，就让我们开始选购设备吧😎

可以参考这篇非常权威详细的[硬盘选购指南](https://gist.github.com/yorickdowne/f3a3e79a573bf35767cd002cc977b038)

我在这里总结一下你可能需要一块怎么样的固态

### ✅ **必须满足（硬性条件）**

这些是 *必须满足* 的基本要求：

-   **NAND：TLC（必须）**
    
    -   QLC 不行：写入慢，耐久差，容易卡死甚至同步失败。
        
-   **DRAM：必须带 DRAM**
    
    -   DRAMless 会让节点像穿铅鞋跑步：IOPS 掉得很惨。
        
-   **接口：NVMe（PCIe 3.0×4 或更高）**
    
    -   SATA SSD 的 IOPS 远远不够，只能用于轻节点或实验。
        
-   **容量：≥ 2TB，推荐 4TB**
    
    -   2TB：可撑到 2026 左右（前合并历史裁剪）。
        
    -   4TB：长期稳定、可支持 Fusaka 超级节点、留足余量。

这里对部分名词做一个简单的介绍

| NAND 类型 | 优点 | 缺点 | 典型用途 |
| --- | --- | --- | --- |
| **SLC** | 极致寿命 & 速度 | 成本高 | 企业/缓存 |
| **MLC** | 高速、耐久 | 稀有 | 企业/重度用户 |
| **TLC ✅ 推荐** | 速度好、寿命够 | 比 MLC 差 | 主流 NVMe SSD |
| **QLC ❌** | 成本最低 | 容易掉速、寿命最短 | 只适合仓库盘、大容量便宜盘 |

- NVMe：专为 SSD 设计的高速存储协议

- PCIe：NVMe SSD 所走的“高速通道”，通道越宽、代数越高越快。

- DRAM：SSD 自带的 DRAM 缓存，决定随机 IO 的下限；无 DRAM 的盘不适合跑全节点。

## 😄我的选择

最终，我入手了致态 TiPro 9000
![tipro9000](/images/Tipro9000.png)

至于我最终为何选择这款，在这里特别鸣谢lEo_Power同学🥰，他给我了非常好的建议
以下是一些参考资料
- [三星杀手:长江存储致态TiPro9000 2T评测报告 YMTC ZHITAI](https://www.bilibili.com/video/BV1WH6ZYLEfJ?buvid=XX2C47A2CC629CEE5C20DD157744E961A9FA2&from_spmid=main.space-contribution.0.0&is_story_h5=false&mid=JxyobQGGPcq1lV86A%2BrxaQ%3D%3D&plat_id=114&share_from=ugc&share_medium=android&share_plat=android&share_session_id=3b297722-6dc6-479e-953e-6510ee692025&share_source=WEIXIN&share_tag=s_i&spmid=united.player-video-detail.0.0&timestamp=1762534000&unique_k=yylhHnf&up_id=3537122193574340&vd_source=119627cb922bc235ae215a4e866d6e03)
- [暴力测试市面上原厂颗粒PCI-E4.0固态及避坑(上)](bilibili.com/video/BV1pA411d7sT?buvid=XX2C47A2CC629CEE5C20DD157744E961A9FA2&from_spmid=search.search-result.0.0&is_story_h5=false&mid=JxyobQGGPcq1lV86A%2BrxaQ%3D%3D&plat_id=114&share_from=ugc&share_medium=android&share_plat=android&share_session_id=6e14ebf0-6f36-4233-b32f-4f79da66f453&share_source=WEIXIN&share_tag=s_i&spmid=united.player-video-detail.0.0&timestamp=1762526549&unique_k=0mwwGBB&up_id=40366538)
- [爆淦3个月暴力测试市面上原厂颗粒PCI-E4.0固态及避坑(下)](https://www.bilibili.com/video/BV19a4y137p3?buvid=XX2C47A2CC629CEE5C20DD157744E961A9FA2&from_spmid=united.player-video-detail.relatedvideo.0&is_story_h5=false&mid=JxyobQGGPcq1lV86A%2BrxaQ%3D%3D&plat_id=114&share_from=ugc&share_medium=android&share_plat=android&share_session_id=ea33dc1a-ed4e-4e22-bef3-b7caa6edab58&share_source=WEIXIN&share_tag=s_i&spmid=united.player-video-detail.0.0&timestamp=1762526561&unique_k=jswzsHa&up_id=40366538&vd_source=119627cb922bc235ae215a4e866d6e03)

装机的过程没有记录下来有点可惜，不过有件趣事，在给主板松螺丝的时候手滑划了一下，闪了一下电火花，不过万幸最后通过设备自检成功开机😇。

## 🔧 硬盘装好后：识别、分区、格式化与挂载
装好硬盘以后还要一下步骤
> 识别 → 分区 → 格式化 → 挂载

### 1\. 识别硬盘

```bash
yxz@KX:~$ lsblk | grep nvme1n1
nvme1n1     259:4    0   1.9T  0 disk /home/yxz/mnt
```

---

## 2\. 创建分区（常用 GPT）

由于这块硬盘我只用于存储全节点数据，所以我并没有分区

```bash
sudo fdisk /dev/nvme0n1
```

常用操作：

-   `g` → 创建 GPT
    
-   `n` → 新建分区（一路默认即可）
    
-   `w` → 保存并退出
    

创建后会自动产生：

```bash
/dev/nvme0n1p1
```

---

## 3\. 格式化为 ext4

格式化：

```bash
sudo mkfs.ext4 /dev/nvme0n1p1
```
---

## 4\. 创建挂载点
把挂载的目录命名为mnt是一个好习惯

```bash
sudo mkdir -p /mnt
#挂载分区：
sudo mount /dev/nvmen1n1 /home/yxz/mnt
```

```bash
#查看挂载结果：
yxz@KX:~$ df -h | grep /dev/nvme1n1
/dev/nvme1n1    1.9T  1.2T  609G  66% /home/yxz/mnt
```

---
## 总结

我们现在已经有一台足以跑以太坊全节点的设备了，让我们进入下一步——节点配置

