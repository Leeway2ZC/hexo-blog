---
title: VirtualBox 虚拟机使用代理教程
date: 2025-10-11 14:40:49
tags:
---

<h1><center>Oracle VirtualBox 虚拟机使用代理教程</center></h1>

##  环境：

- 主机 win10 专业版
- 虚拟机镜像 ubuntu 20.04 LTS
- 代理使用 HTTP/HTTPS/SOCKS5 协议

## 步骤：

1. 在代理软件中查看代理使用的端口号(一般是4位数)

2. 在主机中使用 cmd 输入命令 ipconfig 查看自己的主机 IP 地址

3. 找到 ipv4 的地址信息，知道了这个就可以在ubuntu中设置代理了

4. 在虚拟机中打开终端(ctrl+alt+T)，输入以下命令即可设置虚拟机走代理 

   ```bash
   gsettings set org.gnome.system.proxy mode 'manual'
   gsettings set org.gnome.system.proxy.http host 'ipv4地址'
   gsettings set org.gnome.system.proxy.http port 代理使用的端口号如7897
   gsettings set org.gnome.system.proxy.https host 'ipv4地址'
   gsettings set org.gnome.system.proxy.https port 代理使用的端口号如7897
   ```

5. 为命令行工具配置代理如 apt、curl、wget 等(可选)，在终端使用 gedit ~/.bashrc 打开bash配置文件，在文件末尾添加如下代码 

   ```bash
   export http_proxy="http://ipv4地址:端口号"
   export https_proxy="http://ipv4地址:端口号"
   export ftp_proxy="http://ipv4地址:端口号"
   export no_proxy="localhost,127.0.0.1,::1"
   ```

6. 在虚拟机终端执行 source ~/.bashrc 更新配置

7. 为 APT 工具设置代理(可选)，在终端输入sudo nano /etc/apt/apt.conf.d/95proxies，在末尾添加代码

   ```bash
   sudo nano /etc/apt/apt.conf.d/95proxies
   
   Acquire::http::Proxy "http://ipv4地址:端口号";
   Acquire::https::Proxy "http://ipv4地址:端口号";
   ```

8. 在代理软件中选择"允许局域网连接"，在主机cmd中输入以下代码验证是否生效

   ```bash
   curl cip.cc
   ```

9. 如果返回出口IP，也就是自己的IPv4地址就说明成功了，此后就可以在代理软件中选择允许局域网连接来控制虚拟机的代理了

![img](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/a48cfb879a73499b8af73907e2807df5.png)
