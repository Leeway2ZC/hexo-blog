---
title: VScode连接虚拟机
date: 2025-09-17 10:16:48
tags:
---

<h1><center> 通过Remote-SSH插件连接虚拟机 </center></h1>

主机环境：

- 操作系统：WIN10专业版22H2
- vscode版本：vscode2025 | 1.102.2
- 虚拟机软件：VirtualBox

虚拟机环境：

- 系统版本：ubuntu20.04
- 内核版本：5.15.0
- 架构：x86_64

> 如果不一样应该也没关系

## 1  虚拟机安装 SSH 服务

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

> 安装完成之后需要检查看是否启动ssh服务成功

```bash
ps -e | grep ssh
```

![image-20250917103002105](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917103002105.png)

> ssh-agent表示ssh-client启动，sshd表示ssh-server启动了，这两个是必须有的

## 2  配置VirtualBox

> 为了让vscode能连接上虚拟机，必须设置端口转发规则

![image-20250917103400917](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917103400917.png)

> 或者修改连接方式为桥接网卡，据说这样可以直接将虚拟机和主机置于同一个局域网，但我没有试过

![image-20250917103546423](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917103546423.png)

## 3  主机VScode安装Remote-SSH

![image-20250917103725725](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917103725725.png)

## 4  查看虚拟机ip地址

> 使用 ip a 查看

![image-20250917103932476](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917103932476.png)

> 127.0.0.1 就是虚拟机的ip地址了，vscode使用这个来连接到虚拟机

## 5  vscode连接到虚拟机

![image-20250917104211942](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917104211942.png)

> 输入enter确认后就是确定操作系统以及输入密码了

如果使用NAT模式，则输入 ssh -p 22 root@ip地址

## 6  设置免密登录

1. 设置虚拟机的ssh配置文件允许免密登录

```bash
# 使用vim /etc/ssh/sshd_config 设置下面两条语句
PasswordAuthentication yes
PubkeyAuthentication yes
```

![image-20250917111005551](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917111005551.png)![image-20250917111021391](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917111021391.png)

2. 在主机生成SSH密钥

   1. 使用 git bash 输入`ssh-keygen -t rsa -b 4096 -C "vscode-ssh"`然后一路回车
   2. 这时 `C:\Users\Administrator\.ssh` 下面会有两个文件

   ![image-20250917111344580](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917111344580.png)

   3. 说明密钥生成成功

3. 在虚拟机上生成SSH密钥

![image-20250917111524924](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917111524924.png)

4. 重启一下 ssh

```bash
sudo systemctl restart ssh
```

4. 在主机的 git bash 中使用 `ssh-copy-id` 上传公钥

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub -p 子系统端口号 name@ip
```

> 这里的name还有ip要和vscode的ssh config文件中的一致

![image-20250917111713683](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917111713683.png)

> 这样虚拟机中就有了主机的公钥信息了

5. 在vscode中设置本机的公钥信息

![image-20250917111818587](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917111818587.png)

6. 下次使用vscode远程连接就能直接登录了

## 7  可能出现的问题

### case 1：输入的密码是正确的密码但是重复提示输入密码

> 这种情况是因为虚拟机中的ssh配置文件没有设置允许密码登录

1. 打开ssh配置文件

   ```bash
   sudo vim /etc/ssh/sshd_config
   ```

2. 确保以下两行没有被注释（前面不要有 `#`）

   ```bash
   PermitRootLogin yes         # 如果你用 root 登录
   PasswordAuthentication yes  # 允许密码登录
   ```

3. 如果上面两行没有找到或者只找到一行，自己添加在里面也是可以的

![image-20250917104852593](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917104852593.png)![image-20250917104829435](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250917104829435.png)

4. 添加完成保存并关闭vim，然后重启 SSH

   ```bash
   sudo systemctl restart ssh
   ```





