---
title: Linux内核实验笔记
date: 2025-08-11 17:04:55
tags: 内核教程
categories:
  - 内核教程
from: https://linux-kernel-labs-zh.xyz/
keywords: Linux内核、C语言、操作系统
---

<h1><center>OS2 内核编程和驱动程序开发实验</center></h1>

> 基于布加勒斯特理工大学自动控制与计算机学院计算机科学与工程系的 "操作系统2" 课程

## 环境配置

### 软件配置

- 主机：在win10上使用oracle virtualbox创建的 ubuntu 20.04 作为上位机，用于编辑模块源代码、编译模块、将模块应用到操作系统上等
- 虚拟机：通过Docker配置虚拟机模拟OS内核，在ubuntu 20.04上直接使用，通过脚本运行完成内核模块代码测试

### 操作流程

> 若提示用户没有sudo权限，使用`su -`切换到 root 用户即可，密码是虚拟机设置镜像时自己设置的，一般就是开机密码

1. 在ubuntu 20.04中安装Docker

   ```bash
   # 1. 安装依赖
   sudo apt update
   sudo apt install -y \
       ca-certificates \
       curl \
       gnupg \
       lsb-release
   
   # 2. 添加 Docker 官方 GPG 密钥
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
       sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   
   # 3. 添加 Docker 软件源
   echo \
     "deb [arch=$(dpkg --print-architecture) \
     signed-by=/etc/apt/keyrings/docker.gpg] \
     https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   
   # 4. 更新 apt 并安装 Docker
   sudo apt update
   sudo apt install -y docker-ce docker-ce-cli containerd.io
   
   # 5. 启动并设置开机启动
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

2. 在ubuntu 20.04中安装必需软件

   ```bash
   sudo apt update
   sudo apt install -y flex bison build-essential gcc-multilib libncurses5-dev \
   qemu-system-x86 qemu-system-arm python3 minicom
   ```

3. 创建一个脚本文件并编辑如下内容，随后运行脚本

```bash
#!/bin/bash

if [[ $(id -u) != "0" ]]; then
   echo "Please run as root (or use sudo)"
   exit 1 
fi

cd "$(dirname "$0")" || exit 1

#=============================================================================
#================================= CONSTANTS =================================
#=============================================================================

RED='\033[0;31m'
NC='\033[0m'

DEFAULT_IMAGE_NAME="so2/so2-assignments"
DEFAULT_TAG='latest'
DEFAULT_REGISTRY='gitlab.cs.pub.ro:5050'
SO2_WORKSPACE="/linux/tools/labs"
SO2_VOLUME="SO2_DOCKER_VOLUME"
#=============================================================================
#=================================== UTILS ===================================
#=============================================================================

LOG_INFO() {
    echo -e "[$(date +%FT%T)] [INFO] $1"
}

LOG_FATAL() {
    echo -e "[$(date +%FT%T)] [${RED}FATAL${NC}] $1"
    exit 1
}

#=============================================================================
#=============================================================================
#=============================================================================

print_help() {
    echo "Usage:"
    echo "local.sh docker interactive [--privileged]"
    echo ""
    echo "      --privileged - run a privileged container. This allows the use of KVM (if available)"
    echo "      --allow-gui - run the docker such that it can open GUI apps"
    echo ""
}

docker_interactive() {
    local full_image_name="${DEFAULT_REGISTRY}/${DEFAULT_IMAGE_NAME}:${DEFAULT_TAG}"
    local executable="/bin/bash"
    local registry=${DEFAULT_REGISTRY}

    while [[ $# -gt 0 ]]; do
        case $1 in
        --privileged)
            privileged=--privileged
            ;;
        --allow-gui)
            allow_gui=true
            ;;
        *)
            print_help
            exit 1
        esac
        shift
    done

    if [[ $(docker images -q $full_image_name 2> /dev/null) == "" ]]; then
        docker pull $full_image_name
    fi

    if ! docker volume inspect $SO2_VOLUME >/dev/null 2>&1; then
        echo "Volume $SO2_VOLUME does not exist."
        echo "Creating it"
	docker volume create $SO2_VOLUME
	local vol_mount=$(docker inspect $SO2_VOLUME | grep -i mountpoin | cut -d : -f2 | cut -d, -f1)
	chmod 777 -R $vol_mount
    fi

    echo "The /linux directory is made persistent within the $SO2_VOLUME:"
    docker inspect $SO2_VOLUME

    if $allow_gui; then
	
	# TODO: remove this after you change sdl to gtk in qemu-runqemu.sh
 	docker run $privileged --rm -it --cap-add=NET_ADMIN --device /dev/net/tun:/dev/net/tun \
        -v $SO2_VOLUME:/linux \
        --workdir "$SO2_WORKSPACE" \
        "$full_image_name" sed "s+\${QEMU_DISPLAY:-\"sdl\"+\${QEMU_DISPLAY:-\"gtk\"+g" -i /linux/tools/labs/qemu/run-qemu.sh

	# wsl
	if cat /proc/version | grep -i microsoft &> /dev/null ; then
		export DISPLAY="$(ip r show default | awk '{print $3}'):0.0"
	fi

	if [[ $DISPLAY == "" ]]; then
		echo "Error: Something unexpected happend. The environment var DISPLAY is not set. Consider setting it with"
		echo -e "\texport DISPLAY=<dispaly>"
		exit 1
	fi

	local xauth_var=$(echo $(xauth info | grep Auth | cut -d: -f2))
        docker run --privileged --rm -it \
        --net=host --env="DISPLAY" --volume="${xauth_var}:/root/.Xauthority:rw" \
        -v $SO2_VOLUME:/linux \
        --workdir "$SO2_WORKSPACE" \
        "$full_image_name" "$executable"
    else
        docker run $privileged --rm -it --cap-add=NET_ADMIN --device /dev/net/tun:/dev/net/tun \
        -v $SO2_VOLUME:/linux \
        --workdir "$SO2_WORKSPACE" \
        "$full_image_name" "$executable"
    fi

}

docker_main() {
    if [ "$1" = "interactive" ] ; then
        shift
        docker_interactive "$@"
    fi
}


if [ "$1" = "docker" ] ; then
    shift
    docker_main "$@"
elif [ "$1" = "-h" ] || [ "$1" = "--help" ]; then
    print_help
else
    print_help
fi
```

```bash
# 在终端运行脚本
sudo bash ./local.sh docker interactive --privileged
```

> 显示如下内容时说明配置成功

<img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250804165404202.png" alt="image-20250804165404202" style="zoom:67%;" />

4. 此时根目录下会出现一个目录/linux，这是docker虚拟挂载出来的，只有执行脚本时才会出现这个目录。/linux/tools/lab是这个docker容器的工作目录，是我们编译模块和启动虚拟机的地方。

5. 需要两个Docker内部的终端，通过tmux(终端复用器)可以得到分离的两个终端，输入指令如下

   ```bash
   $tmux
   ```

   使用ctrl+b然后shift+"可以得到水平分割的两个终端，如下所示

   <img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/docker_tmux_VM.png" style="zoom:67%;" />

   > 使用ctrl+b然后shift+$可以得到垂直分割的两个终端，但是不方便复制代码
   > 使用ctrl+b然后按下 [ 可以自由浏览终端界面，方便复制粘贴代码
   > 使用ctrl+b然后按下 d 可以推出tmux，但这会杀死所有tmux正在执行的进程

6. 配置好环境以后就可以进行试验了，将上面的窗格作为虚拟机OS2，下面的窗格作为主机Docker容器，在上面的窗格中执行以下命令即可生成骨架，开始实验

   ```bash
   $ LABS=<实验名称> make skels
   ```

7. 接下来要启动虚拟机，执行 `make console` 使用 `root` 用户名登陆

8. 我们的工作流程包括：在Docker内编写模块代码，修改Make|Kbuild文件，执行make build编译得到ko模块，然后在虚拟机中通过 `insmod` 命令将其插入到虚拟机，或者通过 `rmmod` 将其移除。

   > 每次构建模块无需重启虚拟机，停止虚拟机的操作是 ctrl+a，然后按下 q

## 一 内核模块

### 实验目标

- 创建简单的模块
- 描述内核模块编译的过程
- 展示如何在内核中使用模块
- 简单的内核调试方法

### 1 内核模块的使用|加载|卸载

> 使用 make console 启动虚拟机，并完成以下任务 (正确启动虚拟机以及上位机应该是下面这个界面)

1. 使用`ctrl+alt+t`打开一个终端，确保pwd下有文件`local.sh`，使用以下代码进入docker容器

```bash
sudo bash ./local.sh docker interactive --privileged
```

<img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/%E4%B8%8A%E4%BD%8D%E6%9C%BA.png"  />

2. 在docker容器中构建骨架，编写模块代码，编译模块，然后启动虚拟机安装测试模块

```bash
# 在docker容器中构建骨架，在里面编写代码
# 实验名称一般是skel目录下的目录路径如LABS="kernel_modules/6-cmd-mod kernel_modules/8-kprobes"
LABS=<实验名称> make skels
# 编写完成后，修改Make和Kbuild文件，使用 make build 编译得到 .ko 模块文件，就可以测试了

# 使用tmux分离出一个终端，使用make console启动虚拟机
make console #使用root用户名login，效果如下，此时主机名为qemu
```

![](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/xv6%E8%99%9A%E6%8B%9F%E6%9C%BA.png)

- 加载内核模块
  1. 在 `~/skels/kernel_modules` 目录下有很多模块目录，里面存放要完成的任务

![image-20250806091936889](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806091936889.png)

​          2. 在 `1-2-test-mod` 这个目录下，执行命令 `insmod hello_mod.ko` 完成模块加载

![image-20250806092135698](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806092135698.png)

- 列出内核模块并检查当前模块是否存在

  1. 使用指令 `lsmod`  查看模块是否加载成功

     <img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806092445847.png" alt="image-20250806092445847" style="zoom:67%;" />

- 卸载内核模块

  1. 使用指令 `rmmod hello_mod` (不需要后缀) 完成模块卸载

- 使用 **dmesg** 命令查看加载/卸载内核模块时显示的消息

  ![image-20250806092748489](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806092748489.png)

### 2 Printk

> 配置系统，使消息不直接显示在串行控制台上，只能使用 `dmesg` 命令来查看

- 使用命令 `echo "4 4 1 7" > /proc/sys/kernel/printk` 修改打印日志行为设置

  ![image-20250806093723057](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806093723057.png)

- 此时再加载模块就不会显示消息在串行控制台上了

  ![image-20250806094004426](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806094004426.png)

### 3 错误

> 生成名为 **3-error-mod** 的任务的框架。编译源代码并得到相应的内核模块。
>
> 为什么会出现编译错误? **提示:** 这个模块与前一个模块有什么不同？
>
> 修改该模块以解决这些错误的原因，然后编译和测试该模块。

- 根据TODO提示，缺少头文件 `<linux/module.h>`，添加后能编译成功

  ![image-20250806104845849](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806104845849.png)

  ![image-20250806104917752](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806104917752.png)

### 4 子模块

> 查看 `4-multi-mod/` 目录中的 C 源代码文件 `mod1.c` 和 `mod2.c`。模块 2 仅包含模块 1 使用的函数的定义。
>
> 修改 `Kbuild` 文件，从这两个 C 源文件创建 `multi_mod.ko` 模块。
>
> 编译、复制、启动虚拟机、加载和卸载内核模块。确保消息在控制台上正确显示。

1. 使用 `LABS="kernel_modules/4-multi-mod" make skels` 构建骨架

2. 在目录 `root@ubuntu20:/linux/tools/labs/skels/kernel_modules/4-multi-mod` 中修改Kbuild文件

   ```makefile
   # TODO: add rules to create a multi object module
   obj-m = multi-mod.o
   multi-mod-y = mod1.o mod2.o
   ```

3. 然后 `cd /linux/tools/labs` 进行编译 `make build`

4. 启动虚拟机，加载和卸载 `multi-mod.ko` 模块

   <img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806161240091.png" alt="image-20250806161240091" style="zoom:67%;" />

### 5 内核 oops

> 学习当内核模块代码有问题导致模块插入后内核发生了错误应该怎么处理

1. 使用 `LABS="kernel_modules/5-oops-mod" make skels`  构建骨架

2. 在 `root@ubuntu20:/linux/tools/labs/skels/kernel_modules/5-oops-mod` 中修改Kbuild文件，为Kbuild文件添加编译标记，使得之后在安装模块时，会出现编译过程信息，提示哪里出现了问题

   ```makefile
   # TODO: add flags to generate debug information
   ccflags-y += -g
   
   obj-m = oops_mod.o
   ```

3. `make build` 进行编译，然后在虚拟机中安装模块 `insmod oops_mod.ko` 会输出很长一段编译信息，其中最重要的是

   ```
   # 告诉我们错误的原因
   BUG: kernel NULL pointer dereference, address: 00000000
   # 告诉我们这是第一个 oops（#1）
   Oops: 0002 [#1] SMP
   # 造成错误的指令的地址，它解码了指令指针 (EIP) 的值，并指出错误出现在 my_oops_init 函数中，偏移为 d个字节
   EIP: my_oops_init+0xd/0x22 [oops_mod]
   ```

   ```
   oops 代码（0002）提供了有关错误类型的信息（参见 arch/x86/include/asm/trap_pf.h ）：
   
   第 0 位 == 0 表示找不到页面，1 表示保护故障
   第 1 位 == 0 表示读取，1 表示写入
   第 2 位 == 0 表示内核模式，1 表示用户模式
   ```

4. 有了 EIP 值就可以使用 address2line 来找到出错的代码出现的位置，在主机中使用

   ```bash
   addr2line -e oops_mod.ko +0xd
   /linux/tools/labs/skels/./kernel_modules/5-oops-mod/oops_mod.c:15
   ```

   可以知道是 oops_mod.c 的第 15 行出现了问题

5. 由于oops_mod.ko模块加载卡住了，所以无法正常卸载，因此要重启虚拟机才能完成卸载

   > 模块加载必须经过init函数以及注册exit函数

6. 重启虚拟机之后，删去第15行代码，重新编译以及插入模块即可完成模块的加载与卸载

   <img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250807083857615.png" alt="image-20250807083857615" style="zoom:67%;" />

### 6 模块参数

> 在不修改源代码 `cmd_mod.c` 的情况下，加载内核模块以显示消息 `Early bird gets tired`

- 通过命令行传递参数可以修改函数变量的值从而输出特定内容

  <img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250807090120386.png" alt="image-20250807090120386" style="zoom: 80%;" />

- 使用命令行传递参数需要源代码满足以下条件

  1. 变量必须是模块级别的全局变量，不能是函数内部变量，必须像这样：

     ```c
     static char *str = "default";
     ```

  2. 使用 `module_param()` 宏声明该变量为模块参数

     ```c
     /*
     str：变量名
     charp：变量类型（支持 int、charp、bool、ulong 等）
     0：权限标志位（sysfs 中的访问权限）
     */
     module_param(str, charp, 0);
     ```

  3. 模块必须使用标准 `init` / `exit` 入口函数机制，如

     ```c
     static int __init my_init(void) { ... }
     static void __exit my_exit(void) { ... }
     module_init(my_init);
     module_exit(my_exit);
     # 这样，内核在执行 insmod 时会先处理模块参数，再调用 init 函数。
     ```

  4. 模块参数变量声明前不能加 `const` 因为内核需要在运行时修改它

     ```c
     static const char *str = "hello";  // ❌ 无法作为 module_param
     ```

### 7 进程信息

> 检查名为 **7-list-proc** 的任务的框架。添加代码来显示当前进程的进程 ID（ `PID` ）和可执行文件名

1. 执行 `root@ubuntu20:/linux/tools/labs/skels/kernel_modules/7-list-proc# vim list_proc.c` 命令，修改 `list_proc.c` 文件源代码

2. 在注释TO DO处添加如下代码

   ```c
   /* TODO: add missing headers */
   #include <linux/sched/signal.h>
   /* TODO/2: print current process pid and its name */
   pr_info("Current process: pid = %d; comm = %s\n",
           current->pid, current->comm);
   /* TODO/3: print the pid and name of all processes */
   pr_info("\nProcess list:\n\n");
   for_each_process(p)
           pr_info("pid = %d; comm = %s\n", p->pid, p->comm);
   /* TODO/2: print current process pid and name */
   pr_info("Current process: pid = %d; comm = %s\n",
           current->pid, current->comm);
   ```

3. 编译执行得到如下输出

   <img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250807100210962.png" alt="image-20250807100210962" style="zoom:67%;" />

4. 这里得查很多资料才能知道这些代码是什么意思

### 8 KDB

> 使用KDB(Kernel Debugger)分析堆栈找出错误代码位置|使用KDB找到模块加载的地址|在一个新窗口中使用 GDB 并根据 KDB 提供的信息查看代码(没解决)

1. 在虚拟机中配置KDB使用hvc0串口 `echo hvc0 > /sys/module/kgdboc/parameters/kgdboc`

2. 使用 SysRq 命令启用 KDB (**Ctrl + O g**)，此时进入KDB调试命令行，输入Help可查看可用KDB命令，如果出现乱码例如文字显示不出来，很多乱码挤在界面右侧，是因为minicom的换行格式有问题，按下 ctrl + A 然后按下 U(或者L)，这样会将minicom从列显示模式切换到行显示模式，此时输出即可恢复正常

   >  kdb> "这里输入go可以继续执行内核跳出kdb调试，按回车是重新输入并执行上一个命令，按↑是显示上一个命令"

3. 加载 `hello_kdb` 模块。该模块在写入 `/proc/hello_kdb_bug` 文件时会模拟一个错误。使用以下命令模拟错误：`echo 1 > /proc/hello_kdb_bug`

4. 运行这个命令就会发生oops错误，然后会进入KDB调试命令行，使用 `[0]kdb> bt` 即可分析堆栈跟踪并确定导致错误的代码，bt输出的最下面是执行的起始处(堆栈跟踪要从后往前看)，有些行前面的 ? 是指KDB不确定这个地址偏移是否计算正确，bt输出中最重要的就是kgbd_panic、kgdb_breakpoint这两个点，这表明有函数执行之后发生了错误，可以看到kgbd_panic下面的函数是panic，panic下面的函数是dummy_func1并指明它是 hello_kdb.c文件中的函数，所以错误代码就是 hello_kdb.c 中的函数 dummy_func1 有问题

   ```
   notify_die+0x4d/0x90
   exc_int3+0x5c/0x140
   handle_exception+0x140/0x140
   EIP: kgdb_breakpoint+0xe/0x20                                                         
   Code: b4 26 00 00 00 00 8d b6 00 00 00 00 31 c0 c3 8d b4 26 00 00 00 00 8d b6 00 00 00 00 3e ff8
   EAX: 0000001e EBX: c40b9e00 ECX: 00000000 EDX: 00000000
   ESI: c180e898 EDI: c1badb40 EBP: c4519e2c ESP: c4519e20                               
   DS: 007b ES: 007b FS: 00d8 GS: 00e0 SS: 0068 EFLAGS: 00000002                         
   ? exc_general_protection+0x2c0/0x2c0                                                 
   ? kgdb_breakpoint+0xe/0x20                                                           
   ? kgdb_panic+0x4d/0x60                                                               
   panic+0xbc/0x266                                                                     
   ? dummy_func1+0x8/0x8 [hello_kdb]                                                     
   dummy_func18+0xd/0xd [hello_kdb]                                                     
   dummy_func17+0x8/0x8 [hello_kdb] 
   ```

5. 使用 `[0]kdb> lsmod` 可以看到模块的加载地址

   ![image-20250808170413340](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250808170413340.png)

### 9 PS模块

### 10 内存信息

### 11 动态调试

### 12 初始化期间的动态调试



## 二 内核 API













