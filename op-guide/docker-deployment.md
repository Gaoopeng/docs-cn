# TiDB Docker 部署方案

本篇将展示如何在多台主机上使用 Docker 部署一个 TiDB 集群。

阅读本章前，请先确保阅读 [TiDB 整体架构](../README.md#tidb-总览) 及 [部署建议](../op-guide/recommendation.md)

## 环境准备

### 安装 Docker
使用 Docker 可以非常快速搭建一套 TiDB 环境。
Docker 可以方便地在 Linux / Mac OS X / Windows 平台安装，安装过程可参考 [Docker 官网](https://www.docker.com/products/docker)

### 获取镜像
一个 TiDB 集群包含三种类型的服务组件，TiDB, TiKV 和 PD。

+ tidb-server 是 TiDB 的执行进程，负责客户端的接入，对用户的 SQL 进行解析、优化和执行，并转化为下层的 KV 操作，将执行结果进行聚合返回给客户端。TiDB 对外兼容 MySQL 协议，可以直接使用 MySQL Client 进行测试。
+ tikv-server，作为 TiDB 的分布式 KV 存储引擎，可以在网络互通的多机环境部署，使用 Raft 协议实现强一致性的数据复制，当多数派节点存活时 TiKV 保持可用状态。
+ pd-server，负责 TiKV 的 Region 路由信息的维护和存储，并协调处理数据的 rebalance 以及 merge 和 split 等操作。一个集群可部署多个 PD 实例。

获取 TiDB, TiKV, PD 的 Docker 镜像可以直接拉 Docker Hub 发布的 latest 镜像。

```
docker pull pingcap/tidb
docker pull pingcap/tikv
docker pull pingcap/pd
```

## 多节点部署

假设我们有六台机器:

|Name|Host IP|Services|
|----|-------|----|
|**host1**|192.168.1.101|PD1, TiDB|
|**host2**|192.168.1.102|PD2|
|**host3**|192.168.1.103|PD3|
|**host4**|192.168.1.104|TiKV1|
|**host5**|192.168.1.105|TiKV2|
|**host6**|192.168.1.106|TiKV3|


## 1. 在每台主机上启动一个 `busybox` 作为数据存储容器

```bash
export host1=192.168.1.101
export host2=192.168.1.102
export host3=192.168.1.103
export host4=192.168.1.104
export host5=192.168.1.105
export host6=192.168.1.106

docker run -d --name ti-storage \
  -v /tidata \
  busybox
```

## 2. 在 **host1**，**host2**，**host3** 机器上启动 PD

**host1:**
```bash
docker run -d --name pd1 \
  -p 2379:2379 \
  -p 2380:2380 \
  -v /etc/localtime:/etc/localtime:ro \
  --volumes-from ti-storage \
  pingcap/pd \
  --name="pd1" \
  --data-dir="/tidata/pd1" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://${host1}:2379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://${host1}:2380" \
  --initial-cluster="pd1=http://${host1}:2380,pd2=http://${host2}:2380,pd3=http://${host3}:2380" \
```

**host2:**
```bash
docker run -d --name pd2 \
  -p 2379:2379 \
  -p 2380:2380 \
  -v /etc/localtime:/etc/localtime:ro \
  --volumes-from ti-storage \
  pingcap/pd \
  --name="pd2" \
  --data-dir="/tidata/pd2" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://${host2}:2379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://${host2}:2380" \
  --initial-cluster="pd1=http://${host1}:2380,pd2=http://${host2}:2380,pd3=http://${host3}:2380" \
```

**host3:**
```bash
docker run -d --name pd3 \
  -p 2379:2379 \
  -p 2380:2380 \
  -v /etc/localtime:/etc/localtime:ro \
  --volumes-from ti-storage \
  pingcap/pd \
  --name="pd3" \
  --data-dir="/tidata/pd3" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://${host3}:2379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://${host3}:2380" \
  --initial-cluster="pd1=http://${host1}:2380,pd2=http://${host2}:2380,pd3=http://${host3}:2380" \
```

## 3. 在 **host4**，**host5**，**host6** 机器上启动 TiKV

**host4:**
```bash
docker run -d --name tikv1 \
  -p 20160:20160
  -v /etc/localtime:/etc/localtime:ro \
  --volumes-from ti-storage \
  pingcap/tikv \
  --addr="0.0.0.0:20160" \
  --advertise-addr="${host4}:20160" \
  --store="/tidata/tikv1" \
  --pd="${host1}:2379,${host2}:2379,${host3}:2379" 
```

**host5:**
```bash
docker run -d --name tikv2 \
  -p 20160:20160
  -v /etc/localtime:/etc/localtime:ro \
  --volumes-from ti-storage \
  pingcap/tikv \
  --addr="0.0.0.0:20160" \
  --advertise-addr="${host5}:20160" \
  --store="/tidata/tikv2" \
  --pd="${host1}:2379,${host2}:2379,${host3}:2379" 
```

**host6:**
```bash
docker run -d --name tikv3 \
  -p 20160:20160
  -v /etc/localtime:/etc/localtime:ro \
  --volumes-from ti-storage \
  pingcap/tikv \
  --addr="0.0.0.0:20160" \
  --advertise-addr="${host6}:20160" \
  --store="/tidata/tikv3" \
  --pd="${host1}:2379,${host2}:2379,${host3}:2379" 
```

## 4. 在 **host1** 上启动 TiDB

```bash
docker run -d --name tidb \
  -p 4000:4000 \
  -p 10080:10080 \
  -v /etc/localtime:/etc/localtime:ro \
  pingcap/tidb \
  --store=tikv \
  --path="${host1}:2379,${host2}:2379,${host3}:2379" \
  -L warn
```

## 5. 在 **host1** 上使用 MySQL 客户端连接 TiDB 进行测试

```bash
mysql -h ${host1} -P 4000 -u root -D test
```
