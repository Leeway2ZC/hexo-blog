---
title: BeeGFS学习笔记
date: 2025-10-09 15:56:06
tags:
---

<h1><center>BeeGFS 学习笔记</center></h1>

> 通过实践的方式学习BeeGFS

## 一  在虚拟机中通过docker容器搭建beegfs集群

### 1 目标

在一台虚拟机中运行四个 Docker 容器

| 容器名           | 角色       | IP         | 作用                 |
| ---------------- | ---------- | ---------- | -------------------- |
| `beegfs-mgmtd`   | 管理节点   | 172.25.0.2 | 注册、调度、心跳     |
| `beegfs-meta`    | 元数据节点 | 172.25.0.3 | 管理目录结构和 inode |
| `beegfs-storage` | 存储节点   | 172.25.0.4 | 存放数据块           |
| `beegfs-client`  | 客户端节点 | 172.25.0.5 | 挂载并访问 BeeGFS    |

所有容器在同一个 Docker 自定义网络中，能够互相 `ping` 通

### 2 创建自定义网络

```bash
docker network create --subnet=172.25.0.0/16 beegfs-net
```

验证

```bash
docker network inspect beegfs-net | grep Subnet
# "Subnet": "172.25.0.0/16"
```

![image-20251009092114274](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251009092114274.png)

### 3 创建四个容器

#### 创建管理节点容器(mgmtd)

```bash
docker run -itd --name beegfs-mgmtd --hostname mgmtd \
  --network beegfs-net --ip 172.25.0.2 \
  --privileged ubuntu:20.04 bash
```

#### 创建元数据节点(meta)

```bash
docker run -itd --name beegfs-meta --hostname meta \
  --network beegfs-net --ip 172.25.0.3 \
  --privileged ubuntu:20.04 bash
```

#### 创建存储节点(storage)

```bash
docker run -itd --name beegfs-storage --hostname storage \
  --network beegfs-net --ip 172.25.0.4 \
  --privileged ubuntu:20.04 bash
```

#### 创建客户端节点(client)

```bash
docker run -itd --name beegfs-client --hostname client \
  --network beegfs-net --ip 172.25.0.5 \
  --privileged ubuntu:20.04 bash
```

### 4 测试网络连通性

```bash
# 管理节点
docker exec -it beegfs-mgmtd bash
# 先安装工具
apt update
apt install -y iputils-ping net-tools vim build-essential gdb wget curl
# ping一下
ping -c 2 172.25.0.3   # meta
ping -c 2 172.25.0.4   # storage
ping -c 2 172.25.0.5   # client

# 元数据节点
docker exec -it beegfs-meta bash
# 先安装工具
apt update
apt install -y iputils-ping net-tools vim build-essential gdb wget curl
# ping一下
ping -c 2 172.25.0.2   # meta
ping -c 2 172.25.0.4   # storage
ping -c 2 172.25.0.5   # client

# 存储节点
docker exec -it beegfs-storage bash
# 先安装工具
apt update
apt install -y iputils-ping net-tools vim build-essential gdb wget curl
# ping一下
ping -c 2 172.25.0.2   # meta
ping -c 2 172.25.0.3   # storage
ping -c 2 172.25.0.5   # client

# 客户端节点
docker exec -it beegfs-client bash
# 先安装工具
apt update
apt install -y iputils-ping net-tools vim build-essential gdb wget curl
# ping一下
ping -c 2 172.25.0.2   # meta
ping -c 2 172.25.0.3   # storage
ping -c 2 172.25.0.4   # client
```

### 5 逐个容器安装BeeGFS并启动节点

1️⃣ 管理节点 `mgmtd`

```bash
wget https://www.beegfs.io/release/beegfs_8.1/dists/focal/amd64/beegfs-mgmtd_8.1.0_amd64.deb

mkdir -p /data/mgmtd
echo "storeMgmtdDirectory=/data/mgmtd" > /etc/beegfs/beegfs-mgmtd.conf

/opt/beegfs/sbin/beegfs-mgmtd --init=true --log-target=stderr --log-level=info
/opt/beegfs/sbin/beegfs-mgmtd --log-target=stderr  --log-level=info  --auth-disable=true  --tls-disable=true
```

![image-20251009114047133](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251009114047133.png)

2️⃣ 元数据节点 `meta`

```bash
wget https://www.beegfs.io/release/beegfs_8.1/dists/focal/amd64/beegfs-meta_8.1.0_amd64.deb

dpkg -i beegfs-meta_8.1.0_amd64.deb

mkdir -p /data/meta
echo "storeMetaDirectory=/data/meta" > /etc/beegfs/beegfs-meta.conf
echo "connInterfaces=eth0" >> /etc/beegfs/beegfs-meta.conf
echo "sysMgmtdHost=<MGMT_NODE_IP>" >> /etc/beegfs/beegfs-meta.conf
```

3️⃣ 存储节点 `storage`

```bash
wget https://www.beegfs.io/release/beegfs_8.1/dists/focal/amd64/beegfs-storage_8.1.0_amd64.deb

dpkg -i beegfs-storage_8.1.0_amd64.deb

mkdir -p /data/storage
echo "storeStorageDirectory=/data/storage" > /etc/beegfs/beegfs-storage.conf
echo "connInterfaces=eth0" >> /etc/beegfs/beegfs-storage.conf
echo "sysMgmtdHost=<MGMT_NODE_IP>" >> /etc/beegfs/beegfs-storage.conf
```

4️⃣  客户节点 `client`

```bash
wget https://www.beegfs.io/release/beegfs_8.1/dists/focal/amd64/beegfs-client_8.1.0_all.deb

dpkg -i beegfs-client_8.1.0_all.deb

echo "sysMgmtdHost=172.25.0.2" > /etc/beegfs/beegfs-client.conf
echo "connInterfaces=eth0" >> /etc/beegfs/beegfs-client.conf
```



到这里宣告失败了，在docker容器中部署beegfs实在太难了

