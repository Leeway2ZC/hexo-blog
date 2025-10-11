---
title: IOR 测试
date: 2025-10-11 14:42:36
tags:
---

<h1><center>IOR 测试</center></h1>



## 一  本地文件系统测试

### 1 安装依赖

```bash
sudo apt update && sudo apt install -y git build-essential automake libopenmpi-dev openmpi-bin libaio-dev
```

### 2 克隆并编译 IOR

```bash
git clone https://github.com/hpc/ior.git
cd ior
./bootstrap
./configure
make
```

### 3 测试文件系统

```bash
# Test ext4 (replace /path/to/ext4/testfile with your ext4 mount path):
./src/ior -a POSIX -w -r -t 1m -b 1g -s 1 -o /path/to/ext4/testfile -e -C
```

### 4 测试结果

<img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010085447357.png" alt="image-20251010085447357" style="zoom:150%;" />



## 二  分布式文件系统测试

> 配置：至少两个节点，节点间能互相ping通
> 这里使用两台virtualBox模拟两个节点，通过桥接模式连接
> ior11作为客户端 ip 地址为 192.168.222.154，ubuntu20作为服务器 ip 地址为 192.168.222.233
> 文中node1和node2在此测试中分别为ubuntu20和ior11

### 1 部署 GlusterFS

1. 更新系统并添加 GlusterFS PPA（在所有节点上执行）

```bash
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:gluster/glusterfs-10 -y
sudo apt update
```

2. 安装 GlusterFS 服务器（在所有节点上执行）

```bash
sudo apt install glusterfs-server -y
```

3. 启动并启用 GlusterFS 服务（在所有节点上执行）

```bash
sudo systemctl start glusterd
sudo systemctl enable glusterd
sudo systemctl status glusterd  # 检查服务是否运行
```

![image-20251010153130847](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010153130847.png)

4. 配置主机名解析（在所有节点上执行）: 编辑 /etc/hosts 文件，添加所有节点的 IP 和主机名。例如

```bash
sudo vim /etc/hosts
```

​	添加内容如下:

```bash
# 替换node和地址为真实的主机名和ip地址
192.168.1.101 node1  # 使用hostname查看主机名
192.168.1.102 node2  # 使用ip a查看ip地址
```

![image-20251010153354392](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010153354392.png)

![image-20251010153421113](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010153421113.png)

> 这一步的目的是确保双方能ping通

5. 配置防火墙（如果启用，在所有节点上执行）: GlusterFS 使用端口 24007-24008、49152 等。

```bash
sudo ufw allow from 192.168.1.0/24 to any port 24007:24008 proto tcp
# 根据卷 brick 数量调整
sudo ufw allow from 192.168.1.0/24 to any port 49152:49160 proto tcp  
sudo ufw reload
```

> 这一步我没整，因为虚拟机没启用防火墙

6. 建立节点间的信任(从服务器节点上执行)

```bash
sudo gluster peer probe node2
sudo gluster peer status  # 检查是否连接成功
```

![image-20251010153709826](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010153709826.png)

> 只要能ping通就能连接

7. **创建存储砖**（brick，在所有节点上执行）: 为 GlusterFS 准备专用目录和分区（例如，使用 /data/brick1）。假设您有一个空分区 /dev/sdb：

![image-20251010153828717](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010153828717.png)

> 通过在virtualbox中添加虚拟磁盘可以为虚拟机增加一个空分区sdb

```bash
# 执行下面命令完成分区创建以及挂载
sudo mkfs.xfs /dev/sdb
sudo mkdir -p /data/brick1
sudo mount /dev/sdb /data/brick1
```

​	编辑 /etc/fstab 添加自动挂载:

```bash
/dev/sdb /data/brick1 xfs defaults 0 0
```

8. 创建分布式卷（从服务器节点执行）: 这里创建一个简单的分布式复制卷（replicated volume），副本数为 2

```bash
sudo gluster volume create test-volume replica 2 node1:/data/brick1 node2:/data/brick1 force
sudo gluster volume start test-volume
sudo gluster volume info test-volume  # 检查卷状态
```

9. 挂载 GlusterFS 卷（在客户端节点或测试节点上执行）: 安装客户端（如果不是服务器节点）

```bash
sudo apt install glusterfs-client -y
```

```bash
sudo mkdir /mnt/gluster
sudo mount -t glusterfs node1:/test-volume /mnt/gluster
```

> 此时就在客户端节点上挂载了服务器端的卷，客户端或服务器写这个卷两边都会同时修改

- 如果重启虚拟机，则需要重新挂载一一遍glusterfs

```bash
sudo mount -t glusterfs node1:/test-volume /mnt/gluster
```

​	或者设置自动挂载

```bash
sudo vim /etc/fstab
node1:/test-volume /mnt/gluster glusterfs defaults,_netdev,backupvolfile-server=node2 0 0
```

- 在服务器上使用 `echo "Hello Gluster" > /mnt/gluster/testfile` 测试是否挂载成功

![image-20251010165013777](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010165013777.png)

![image-20251010165020162](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010165020162.png)

- 或者使用 `mount | grep /mnt/gluster` 查看是否挂载了 gluster 卷，如果为空则没有挂载

![image-20251010164633499](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010164633499.png)



## 2  安装 MPI 和 IOR

> IOR 是一个 I/O 基准测试工具，需要 MPI支持分布式测试，这里使用OpenMPI

1. 安装OpenMPI（在所有测试节点上执行）

```bash
sudo apt update
sudo apt install openmpi-bin openmpi-common libopenmpi-dev -y
mpirun --version
```

​	如果需要 SSH 支持分布式运行（多节点）

```bash
sudo apt install openssh-server -y
# 配置无密码 SSH（生成密钥并复制到其他节点）
ssh-keygen -t rsa
ssh-copy-id user@node2  # 替换 user 和 node2

ssh-keygen -t rsa
ssh-copy-id user@node1  # 替换 user 和 node2
```

> 配置无密码 SSH 时要注意，如果node1或2的ssh配置文件没有设置
> PermitRootLogin yes
> PasswordAuthentication yes
> 是无法通过密码 ssh 到 node1或2 上的
> 修改完ssh配置后，使用sudo sshd -t检查是否有语法错误，然后重启ssh服务
> sudo systemctl restart ssh 
> sudo systemctl status ssh   # 检查服务是否运行，无错误

2. 安装 IOR（在测试节点上执行，从源代码编译，因为 Ubuntu 仓库中没有直接包）: IOR 需要 MPI 支持，所以在配置时指定

```
sudo apt install git autoconf automake libtool make gcc -y  # 安装依赖
git clone https://github.com/hpc/ior.git
cd ior
./bootstrap
./configure --prefix=/usr/local --with-mpiio
make
sudo make install
```

> 如果提示 autoconf 版本不正确，则通过以下命令升级autoconf版本
> sudo apt update && sudo apt install -y wget tar make m4 perl
> wget https://ftp.gnu.org/gnu/autoconf/autoconf-2.71.tar.gz 
> tar -xzf autoconf-2.71.tar.gz
> cd autoconf-2.71 
> ./configure --prefix=/usr/local 
> make 
> sudo make install

## 3  使用 IOR 通过 MPI 测试 GlusterFS 性能

> IOR 支持多种 I/O 模式（如 POSIX、MPI-IO），可以测试读/写带宽、IOPS 等。使用 MPI 允许分布式测试（多进程/多节点）

1. 单机测试写入性能（MPI-IO 接口，2 进程，块大小 1MB，传输大小 1MB，重复 3 次）

```bash
mpirun -np 2 ior -a MPIIO -b 1m -t 1m -i 3 -o /mnt/gluster/testfile -w
```

- -np 2: 使用 2 个 MPI 进程（根据您的 CPU 核心调整）
- -a MPIIO: 使用 MPI-IO 接口（适合分布式文件系统）
- -b 1m: 块大小（block size）
- -t 1m: 传输大小（transfer size）。
- -i 3: 重复迭代 3 次。
- -o: 输出文件路径（在 Gluster 卷上）
- -w: 只测试写入。 输出将显示带宽（MB/s）、操作时间等

![image-20251010172727658](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251010172727658.png)

2. 测试读取性能

```bash
mpirun -np 2 ior -a MPIIO -b 1m -t 1m -i 3 -o /mnt/gluster/testfile -r
```

3. **多节点测试(重点部分)**

使用 -hostfile 指定主机： 创建 hostfile.txt：

```bash
node1 slots=2  // slots是可用cpu数量
node2 slots=2
```

进行测试

```bash
mpirun --hostfile hostfile.txt ior -a MPIIO -b 1m -t 1m -i 3 -o /mnt/gluster/testfile -w
```

![image-20251011081051277](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251011081051277.png)![image-20251011081112988](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251011081112988.png)

![image-20251011081230613](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20251011081230613.png)

## 4 总结

> 做完了前面的准备工作后，使用ior测试分布式文件系统gluster的性能就这么几步

1. 在所有节点上启动 GlusterFS 服务

```bash
sudo systemctl start glusterd
sudo systemctl enable glusterd
sudo systemctl status glusterd
```

2. 挂载 GlusterFS 卷

```
sudo mount -t glusterfs node1:/test-volume /mnt/gluster
```

3. 进行测试

```bash
# 准备hostfile.txt文件内容
node1 slots=2
node2 slots=2
```

```bash
# 运行测试
mpirun --hostfile hostfile.txt ior -a MPIIO -b 1m -t 1m -i 3 -o /mnt/gluster/testfile -w
```



## 三  离线环境下安装 IOR

