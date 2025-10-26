---
title: 湘潭大学浪潮AI服务器使用
date: 2025-10-26 18:57:22
tags:
---

# 湘潭大学浪潮AI服务器使用

> 首先要使用EasyConnect连接到学校内网

## 一  安装 torch 和 tensorflow

- 创建环境后，在vscode中安装ssh拓展，使用ssh链接登录

```bash
连接命令：ssh root@172.24.128.111 -p 30170
ssh密码：553139
```

- 登录后，在vscode的终端界面，先创建一个虚拟环境

```bash
conda create -n env_name python=3.9
```

- 然后激活环境

```bash
conda activate env_name
```

- 如果激活失败，可能是当前的bashrc配置没写conda的环境变量，使用以下指令

```bash
source ~/.bashrc
```

- 在自己的虚拟环境中使用如下命令安装pytorch

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

- 使用如下命令安装tensorflow

```bash
pip install tensorflow
```

- 使用以下代码检查是否安装成功

```python
# 验证 torch 安装
import torch
# 当前安装的 PyTorch 库的版本
print(torch.__version__)
# 检查 CUDA 是否可用，即你的系统有 NVIDIA 的 GPU
print(torch.cuda.is_available())
# 检查系统可用的GPU数量
print(torch.cuda.device_count())

# 验证 tensorflow 安装
import tensorflow as tf
# 当前安装的 tensorflow 库的版本
print(f"tensorflow version:  {tf.__version__}")
print(f"Keras version: {tf.keras.__version__}")
# 检查系统可用的GPU数量
print(len(tf.config.list_physical_devices('GPU')))
```
