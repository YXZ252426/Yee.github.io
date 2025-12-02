---
title: 以太坊全节点部署：节点配置篇
date: 2025-11-30 17:06:38
tags:
categories:
   - Ethereum is all you need
   - 全节点部署
---
这篇我们来讲讲一个全节点的相关配置项，在这里我会介绍我自己的配置以供参考，文章里可能涉及不少知识点，如果有问题可能需要自己补足相关知识，以下是涉及到的。
`Linux 基础` `Docker 与容器化`  `网络与节点通信` `Reth` `Lighthouse` `节点监控`
其中，大家可以着重看我的配置😊
# 全节点客户端的选择
为什么我会选择Reth+lighthouse的组合呢，非常简单，因为我是一个rustacean😎🦀，这里是reth的官方文档，几乎所有问题在[这里](https://reth.rs/run/ethereum)都可以找到答案。
# reth文档导读
Reth的官方文档提供了不少资源，在这里我列举一些你一定会用到的。

[设备要求](https://reth.rs/run/system-requirements)，在硬件篇也有提过
[run Reth with docker compose](https://reth.rs/installation/docker#using-docker-compose:~:text=your%20preferred%20tag.-,Using%20Docker%20Compose,-To%20run%20Reth)，这是最推荐的方式，因为共识层，执行层，节点监控都集成在同一个compose里
[初始化配置](https://reth.rs/run/ethereum)，这个也可以重点来看我提供的配置
[JSON-RPC服务](https://reth.rs/jsonrpc/intro)，让你的节点提供服务
[Reth node cli](https://reth.rs/cli/reth/node),虽然很长，但推荐看一遍，更加了解节点配置
[Reth架构](https://reth.rs/sdk/node-components/network)，最终你肯定会需要他
# docker compose 配置及命令
这是这篇文章最最重要的部分，在其中有无数我踩过的坑😭😭
```yaml
name: reth

services:
  reth:
    restart: unless-stopped
    image: ghcr.io/paradigmxyz/reth
    ports:
      - "9001:9001" # metrics
      - "30303:30303/tcp" # eth/66 peering
      - "30303:30303/udp"
      - "8545:8545" # rpc
      - "8551:8551" # engine
    environment:
      - NO_PROXY=localhost,127.0.0.1,reth,lighthouse,prometheus,grafana,metrics-exporter,172.22.0.0/16
    volumes:
      - /home/yxz/mnt/reth5/mainnet:/root/.local/share/reth/mainnet
      - /home/yxz/mnt/reth5/sepolia:/root/.local/share/reth/sepolia
      - /home/yxz/mnt/reth5/holesky:/root/.local/share/reth/holesky
      - /home/yxz/mnt/reth5/hoodi:/root/.local/share/reth/hoodi
      - /home/yxz/mnt/reth5/logs:/root/logs
      - ./jwttoken:/root/jwt/:ro
    # https://paradigmxyz.github.io/reth/run/troubleshooting.html#concurrent-database-access-error-using-containersdocker
    pid: host
    # For Sepolia, replace `--chain mainnet` with `--chain sepolia`
    # For Holesky, replace `--chain mainnet` with `--chain holesky`
    # For Hoodi, replace `--chain mainnet` with `--chain hoodi`
    command: >
      node
      --chain mainnet
      --full
      --metrics 0.0.0.0:9001
      --log.file.directory /root/logs
      --authrpc.addr 0.0.0.0
      --authrpc.port 8551
      --authrpc.jwtsecret /root/jwt/jwt.hex
      --max-inbound-peers 30
      --http --http.addr 0.0.0.0 --http.port 8545
      --http.api "eth,net,web3"

  prometheus:
    restart: unless-stopped
    image: prom/prometheus
    depends_on:
      - reth
    ports:
      - 9090:9090
    environment:
      - NO_PROXY=localhost,127.0.0.1,reth,lighthouse,prometheus,grafana,metrics-exporter,172.22.0.0/16
    volumes:
      - ./prometheus/:/etc/prometheus/
      - /home/yxz/mnt/reth5/prometheus:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus

  grafana:
    restart: unless-stopped
    image: grafana/grafana:latest
    depends_on:
      - reth
      - prometheus
    ports:
      - 3000:3000
    environment:
      - PROMETHEUS_URL=${PROMETHEUS_URL:-http://prometheus:9090}
      - NO_PROXY=localhost,127.0.0.1,reth,lighthouse,prometheus,grafana,metrics-exporter,172.22.0.0/16
    volumes:
      - /home/yxz/mnt/reth5/grafana:/var/lib/grafana
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
      - ./grafana/dashboards:/etc/grafana/provisioning_temp/dashboards
    # 1. Copy dashboards from temp directory to prevent modifying original host files
    # 2. Replace Prometheus datasource placeholder with the actual name
    # 3. Run Grafana
    entrypoint: >
      sh -c "cp -r /etc/grafana/provisioning_temp/dashboards/. /etc/grafana/provisioning/dashboards &&
             find /etc/grafana/provisioning/dashboards/ -name '*.json' -exec sed -i 's/$${DS_PROMETHEUS}/Prometheus/g' {} \+ &&
             /run.sh"

```
其中，我来挑选几个比较重要的~
## 卷挂载与权限设置
docker通过挂载卷（volumn）实现持久化存储，但是由于Reth的数据量极大（1TB），所以我们不能使用默认的容器内部路径，而是需要手动挂载到自己的固态硬盘，确保性能与空间都足够。
```yaml
    volumes:
      - /home/yxz/mnt/reth5/mainnet:/root/.local/share/reth/mainnet
      - /home/yxz/mnt/reth5/sepolia:/root/.local/share/reth/sepolia
      - /home/yxz/mnt/reth5/holesky:/root/.local/share/reth/holesky
      - /home/yxz/mnt/reth5/hoodi:/root/.local/share/reth/hoodi
      - /home/yxz/mnt/reth5/logs:/root/logs
```
下一步还需要文件夹权限设置（这时候如果直接跑你会发现permission denied😭）
### 知识补全：linux权限系统
Linux 文件系统的每个文件（包括目录）都带着一个三段式的权限字段：
```css
[ owner ][ group ][ others ]
```
其中，rwx表示可读可写可执行
接下来让我们看看各个容器内运行的用户
```bash
yxz@KX:~/reth-main$ docker exec -it reth-reth-1 bash
root@812343f0489d:/# id
uid=0(root) gid=0(root) groups=0(root)

yxz@KX:~/reth-main$ docker exec -it reth-grafana-1 bash
a359ad628d4a:/usr/share/grafana$ id
uid=472(grafana) gid=0(root) groups=0(root)

yxz@KX:~/reth-main$ docker exec -it reth-prometheus-1 sh
/prometheus $ id
uid=65534(nobody) gid=65534(nobody) groups=65534(nobody)
```
由此可见，reth是以root用户运行的，没有问题，而prometheus(数据采集工具)，和grafana（可视仪表盘）是需要权限设置的
这个时候，我们就需要设置文件夹的用户与权限
```bash
sudo chown -R 472:472 /home/yxz/mnt/reth5/grafana
sudo chmod -R 755 /home/yxz/mnt/reth5/grafana
```
## node Cli
首先，再次强烈建议看看官方的[Reth node cli](https://reth.rs/cli/reth/node)，在这里我简单解析一下指令
```bash
node
--chain mainnet                     # 指定网络：mainnet/sepolia/holesky。决定创世块与网络 ID。

--full                              # 运行全节点（Pruned Full Node），不保存全部历史 state，性能与占用空间更均衡。

--metrics 0.0.0.0:9001              # 打开 Prometheus metrics，用于 Grafana 监控。
                                    # 0.0.0.0 表示监听所有网卡，若公网部署务必配合防火墙。

--log.file.directory /root/logs     # 将日志写入独立目录，便于持久化与排查问题。

--authrpc.addr 0.0.0.0              # 执行层与共识层通信接口（EL ↔ CL）。
--authrpc.port 8551                 # Lighthouse 等共识客户端会通过此端口连接 Reth。
--authrpc.jwtsecret /root/jwt/jwt.hex
                                    # JWT 密钥文件，EL 与 CL 的唯一认证方式。
                                    # 参数错误将导致 Beacon Node 无法连接。

--max-inbound-peers 30              # 允许的入站 peers 数量。
                                    # 入站 peers 越多，区块传播越快，同步质量越高。
                                    # 可根据机器带宽适当调大（建议 30–80）。

--http                              # 开启 HTTP JSON-RPC 服务，用于本地开发与查询。
--http.addr 0.0.0.0                 # 监听所有地址（⚠ 不推荐在公网直接暴露）。
--http.port 8545                    # 常用 RPC 端口。
--http.api "eth,net,web3"           # 开放的 API 模块列表：eth、net、web3。
                                    # 仅提供基础查询，避免暴露危险接口。

# 网络穿透与 peer 数量优化相关（强烈建议使用）
--nat extip:<你的公网IP>            # 显式声明公网 IP，让其他节点更容易与您建立连接。
                                    # 建议公网部署、NAT/双层 NAT 环境下必开。

# (可选) 增强 peers 数量的其它方式：
# --max-outbound-peers 30            # 出站 peers 数量上限。通常与 inbound 保持均衡。
# --trusted-peers <peer1,peer2,...> # 固定可信节点，可提升连接质量。

```

## docker compose 
接下来讲讲你一定会用到的docker compose命令
首先先进入docker-compose.yml所在的目录（reth/etc）
### `logs --tail <N> -f`
查看日志（最常用）

```bash
docker compose logs --tail 200 -f reth
```
---
### `exec <service> <cmd>`

进入容器运行命令

```bash
docker compose exec reth-reth-1 bash
```

用途：检查权限、目录、内部配置。
---
### `stats`

实时查看容器资源占用

```bash
docker compose stats
```
---
### `up -d`
后台启动服务
```bash
docker compose up -d
```
---
### `down`
停止并**删除容器实例**（数据卷不删）
```bash
docker compose down
```

用途：大改配置、清理旧实例。（修改docker-compose.yml需要down并重新up）

---
### `restart`
重启容器，不重建
```bash
docker compose restart reth
```
---
### down 和 restart 的区别

-   **restart**：直接重新启动
    
-   **down**：删除容器，再 `up` 会重新创建实例（更干净）
---
# 节点监控
## linux指令
### `ss` —— 查看端口监听状态

```bash
ss -tulpn | grep 9000     # Lighthouse P2P
ss -tulpn | grep 30303    # Reth P2P
ss -tulpn | grep 8545     # JSON-RPC
ss -tulpn | grep 8551     # AuthRPC（EL <-> CL）
```
### `htop` —— CPU/内存实时监控

```bash
htop
```
可以看到，cpu和内存都被reth狠狠吃满了😭😭
![htop](/images/htop.png)
## grafana仪表盘
这两张图片分别展示了节点同步情况和内存使用情况
![grafana1](/images/grafana1.JPG)
![grafana2](/images/grafana2.png)

## 总结
以上就是这篇的全部内容，如果你照着做完了，剩下的事情就是等待同步和日常维护了，但是假如你没有公网ip，那你可能需要下一篇教程——frp😭，