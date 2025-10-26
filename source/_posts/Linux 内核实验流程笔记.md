---
title: Linux内核实验流程笔记
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

9. 重新开始课程（删除卷并重建）

   ```
   docker volume rm SO2_DOCKER_VOLUME
   sudo bash ./local.sh docker interactive --privileged
   ```



## 一  内核模块

实验目标

- 创建简单的模块
- 描述内核模块编译的过程
- 展示如何在内核中使用模块
- 简单的内核调试方法

### 1.内核模块的使用|加载|卸载

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

### 2.Printk

> 配置系统，使消息不直接显示在串行控制台上，只能使用 `dmesg` 命令来查看

- 使用命令 `echo "4 4 1 7" > /proc/sys/kernel/printk` 修改打印日志行为设置

  ![image-20250806093723057](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806093723057.png)

- 此时再加载模块就不会显示消息在串行控制台上了

  ![image-20250806094004426](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806094004426.png)

### 3.错误

> 生成名为 **3-error-mod** 的任务的框架。编译源代码并得到相应的内核模块。
>
> 为什么会出现编译错误? **提示:** 这个模块与前一个模块有什么不同？
>
> 修改该模块以解决这些错误的原因，然后编译和测试该模块。

- 根据TODO提示，缺少头文件 `<linux/module.h>`，添加后能编译成功

  ![image-20250806104845849](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806104845849.png)

  ![image-20250806104917752](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250806104917752.png)

### 4.子模块

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

### 5.内核 oops

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

### 6.模块参数

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

### 7.进程信息

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

### 8.KDB

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



## 二  内核 API

实验目标：

- 熟悉基本的Linux内核API

  > 内核是一个独立运行的实体，不能调用用户空间的任何库，所以不能使用printf、malloc、free等常见的用户控件函数

- 描述内存分配机制

- 描述锁定机制

### 1.Linux 内核中的内存分配

- `GFP_KERNEL` ——使用此值可能导致当前进程被挂起。因此，它不能在中断上下文中使用。
- `GFP_ATOMIC` ——使用此值确保 `kmalloc()` 函数不会挂起当前进程。它可以随时使用。

```c
static char *mem;
static int mem_init(void)
{
	size_t i;
	// 第一个参数是字节大小，这里是4096个字节，第二个参数是分配标志
    // 表示这是普通内核上下文分配，允许睡眠、可以进行内存回收
	mem = kmalloc(4096 * sizeof(*mem), GFP_KERNEL);
	if (mem == NULL)
		goto err_mem;
	// 打印mem~mem+4096内存地址区间的所有值为字母的元素
	pr_info("chars: ");
	for (i = 0; i < 4096; i++) {
		if (isalpha(mem[i]))
			printk("%c ", mem[i]);
	}
	pr_info("\n");
	
	return 0;

err_mem:
	return -1;
}
static void mem_exit(void)
{
	kfree(mem);
}
```

> 重点在于使用kmalloc分配内存给指针，从而能使用指针指向内存空间进行引用、操作，其实kmalloc的用法和malloc差不多

### 2.在原子上下文中睡眠

```c
static int sched_spin_init(void)
{	
    // 定义一个自旋锁变量
	spinlock_t lock;
	// 初始化自旋锁变量
	spin_lock_init(&lock);
	// 执行锁定，此时CPU进入中断上下文进行原语操作，即此时代码运行在由自旋锁保护的临界区域当前进程不能挂起或睡眠
	spin_lock(&lock);
	// 强制使当前进程进入睡眠，因此执行insmod时会报错
	set_current_state(TASK_INTERRUPTIBLE);
	schedule_timeout(5 * HZ);
	// 释放锁定，使用自旋锁时一定注意，在spin_lock和unlock这两个函数之间不能有挂机或睡眠操作代码
	spin_unlock(&lock);

	return 0;
}
```

> 学习重点在于学会使用自旋锁变量以及使用时的注意要点

### 3.使用内核内存

```c
struct task_info {
	pid_t pid;
	unsigned long timestamp;
};

static struct task_info *ti1, *ti2, *ti3, *ti4;

static struct task_info *task_info_alloc(int pid)
{
	struct task_info *ti;

	/* TODO 1/5: allocated and initialize a task_info struct */
	// 没有使用sizeof(struct task_info)，可能是为了更好的复用，这里参考1内存分配的操作即可
    ti = kmalloc(sizeof(*ti), GFP_KERNEL);
	if (ti == NULL)
		return NULL;
	ti->pid = pid;
    // jiffies 是一个全局可见变量，表示当前的时间
	ti->timestamp = jiffies;

	return ti;
}

static int memory_init(void)
{
	/* TODO 2/1: call task_info_alloc for current pid */
    // current是一个可以直接使用的宏，等价于struct task_struct结构体，使用current->pid查找当前进程的PID值
	ti1 = task_info_alloc(current->pid);

	/* TODO 2/1: call task_info_alloc for parent PID */
	// 使用current->parent查找当前进程的父进程，这里纯属背板操作，无需了解为什么
    ti2 = task_info_alloc(current->parent->pid);

	/* TODO 2/1: call task_info alloc for next process PID */
    // 使用next_task(current)宏找到当前进程的下一个进程
	ti3 = task_info_alloc(next_task(current)->pid);

	/* TODO 2/1: call task_info_alloc for next process of the next process */
	ti4 = task_info_alloc(next_task(next_task(current))->pid);

	return 0;
}

static void memory_exit(void)
{

	/* TODO 3/4: print ti* field values */
	printk("pid: %d, timestamp: %lu\n", ti1->pid, ti1->timestamp);
	printk("pid: %d, timestamp: %lu\n", ti2->pid, ti2->timestamp);
	printk("pid: %d, timestamp: %lu\n", ti3->pid, ti3->timestamp);
	printk("pid: %d, timestamp: %lu\n", ti4->pid, ti4->timestamp);

	/* TODO 4/4: free ti* structures */
    // 释放内存
	kfree(ti1);
	kfree(ti2);
	kfree(ti3);
	kfree(ti4);
}
```

> 重点在于学习如何给分配了内存的变量赋值以及如何使用current宏、next_task宏找到进程PID值

### 4.使用内核列表

```C
struct task_info {
	pid_t pid;
	unsigned long timestamp;
	struct list_head list;
};

static struct list_head head;

static struct task_info *task_info_alloc(int pid)
{
	struct task_info *ti;

	ti = kmalloc(sizeof(*ti), GFP_KERNEL);
	if (ti == NULL)
		return NULL;
	ti->pid = pid;
	ti->timestamp = jiffies;

	return ti;
}

static void task_info_add_to_list(int pid)
{
	struct task_info *ti;

	/* TODO 1/2: Allocate task_info and add it to list */
    // 分配内存给ti变量,ti是一个task_info结构体指针，描述任务信息(这里指描述当前进程的PID和时间)
	ti = task_info_alloc(pid);
	// 将ti添加到链表，注意list_add函数的用法，传入的是ti->list的地址和head的地址(没有使用结构体指针)
    list_add(&ti->list, &head);
}

static void task_info_add_for_current(void)
{
	/* Add current, parent, next and next of next to the list */
	task_info_add_to_list(current->pid);
	task_info_add_to_list(current->parent->pid);
	task_info_add_to_list(next_task(current)->pid);
	task_info_add_to_list(next_task(next_task(current))->pid);
}

static void task_info_print_list(const char *msg)
{
	struct list_head *p;
	struct task_info *ti;

	pr_info("%s: [ ", msg);
	list_for_each(p, &head) {
		ti = list_entry(p, struct task_info, list);
		pr_info("(%d, %lu) ", ti->pid, ti->timestamp);
	}
	pr_info("]\n");
}

static void task_info_purge_list(void)
{
	struct list_head *p, *q;
	struct task_info *ti;

	/* TODO 2/5: Iterate over the list and delete all elements */
    // 这里要注意，list_for_each_safe是一个宏而不是函数，不要加分号
	list_for_each_safe(p, q, &head) {
		// list_entry是找到当前链表节点相对应的原来的结构体指针变量，即映射回去
        ti = list_entry(p, struct task_info, list);
		list_del(p);
		kfree(ti);
	}
}
```

  知识点

- `list_entry(ptr, type, member)()` 返回列表中包含元素 `ptr` 的类型为 `type` 的结构，该结构中具有名为 `member` 的成员。
- `list_for_each(pos, head)` 使用 `pos` 作为游标来迭代列表。
- `list_for_each_safe(pos, n, head)` 使用 `pos` 作为游标，`n` 作为临时游标来迭代列表。此宏用于从列表中删除项目。
- `list_del(struct list_head *entry)()` 删除属于列表的 `entry` 地址处的项目。
- `list_add(struct list_head *new, struct list_head *head)()` 将 `new` 指针所引用的元素添加到 `head` 指针所引用的元素之后。
- 使用 `static struct list_head head;` 来声明一个链表头，在使用head前进行 `INIT_LIST_HEAD(&head);` 
- `INIT_LIST_HEAD(struct list_head *list)()` 用于在进行动态分配时，通过设置链表字段 `next` 和 `prev`，来初始化链表的标记。

### 5.使用内核列表进行进程处理

```C
// kernel_api\5-list-full\list-full.c
static struct task_info *task_info_find_pid(int pid)
{
	struct list_head *p;
	struct task_info *ti;

	/* TODO 1/5: Look for pid and return task_info or NULL if not found */
	// 找到成员pid值等于参数pid值的链表节点ti
    list_for_each(p, &head) {
		ti = list_entry(p, struct task_info, list);
		if (ti->pid == pid)
			return ti;
	}

	return NULL;
}

static void list_full_exit(void)
{
	struct task_info *ti;

	/* TODO 2/2: Ensure that at least one task is not deleted */
	// 这里要学会使用原子操作函数atomic_set，原子操作是一种不会被打断必定执行的操作，必须使用原子变量atomic_t
    ti = list_entry(head.prev, struct task_info, list);
	atomic_set(&ti->count, 10);

	task_info_remove_expired();
	task_info_print_list("after removing expired");
	task_info_purge_list();
}
```

> 一些常用的原子操作函数

![](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/Snipaste_2025-08-13_15-12-15.png)

### 6.同步列表工作

> 代码相关答案可以看/templates文件夹下的代码

  使用DEFINE_RWLOCK(lock)定义一个读写自旋锁

```c
/* TODO 1: you can use either a spinlock or rwlock, define it here */
DEFINE_RWLOCK(lock);
```

  读写自旋锁中的代码涉及到的共享资源会被锁定

```c
write_lock(&lock);
/* 临界区（critical region） */
ti = task_info_find_pid(pid);
if (ti != NULL) {
    ti->timestamp = jiffies;
    atomic_inc(&ti->count);
    /* TODO: Guess why this comment was added  here */
    /* 临界区（critical region） */
    write_unlock(&lock);
    return;
}
/* TODO 1: critical section ends here */
/* 临界区（critical region） */
write_unlock(&lock);

read_lock(&lock);
list_for_each(p, &head) {
    ti = list_entry(p, struct task_info, list);
    pr_info("(%d, %lu) ", ti->pid, ti->timestamp);
}
/* TODO 1: Critical section ends here */
read_unlock(&lock);
pr_info("]\n");
```

  简单来说，就是write_lock和write_unlock之间的代码片段当cpu在执行时会进入临界区，此时如果是write_lock，那就是只有当前的进程能写，其他的进程包括CPU都不能写，如果是read_lock，那就是所有进程都不能写，但是可以一起读

> 如果代码只涉及共享资源的访问就使用read_lock，如果设计对共享资源的修改就使用write_lock，在这个例子中，由于要修改ti的时间戳和计数器，所以使用了write_lock读自旋锁，而只需要打印ti的pid和时间戳，所以使用read_lock

  关于 `EXPORT_SYMBOL(name);` ，其作用是导出模块代码中的函数或者变量给其它模块使用，当模块代码中使用了 `EXPORT_SYMBOL(name);` 那么加载此模块后，其他模块也能使用 name 代表的函数或者变量，但有几点要求

1. 函数或变量不能是静态的，即不能使用 static 关键字
2. 必须在函数定义或变量赋值后使用

### 7.在我们的列表模块中测试模块调用

  这一节没什么好讲的，就是一个模块依赖关系，如果模块代码使用了其他模块导出的内核符号name，则这个模块依赖于其他模块，被依赖的模块由于模块引用计数refcnt>0无法卸载，所以必须先卸载依赖模块，在这个例子中就是必须先卸载 list-test 模块，然后卸载 list-sync 模块

  除此以外，如果一个模块要使用其他模块导出的内核符号(函数或者变量)，必须先extern声明这个内核符号再使用，例如

```c
// 要使用 task_info_print_list
void task_info_print_list(const char *msg) //被依赖模块代码
{
	struct list_head *p;
	struct task_info *ti;

	pr_info("%s: [ ", msg);

	/* TODO 1: Protect list, is this read or write access? */
	read_lock(&lock);
	list_for_each(p, &head) {
		ti = list_entry(p, struct task_info, list);
		pr_info("(%d, %lu) ", ti->pid, ti->timestamp);
	}
	/* TODO 1: Critical section ends here */
	read_unlock(&lock);
	pr_info("]\n");
}
EXPORT_SYMBOL(task_info_print_list);
// 必须先 extern task_info_print_list
extern void task_info_print_list(const char *msg); //依赖模块代码
```

> 很多东西都是背板式的，如果每个不熟悉的符号都使用LXR或cscope去查询，会消耗大量时间而且不一定能查找正确，学习linux内核编程有如学习一个语法无比复杂的语言，与其先背下来所有单词和认识所有语法后再实践练习使用，不如先开口把最常用最实用的操作记下来，让自己变得熟练，那么以前那些晦涩难懂的知识也就比较容易理解了



## 三  字符设备驱动程序

 实验目标

- 理解字符设备驱动程序背后的概念
- 理解可以在字符设备上执行的各种操作
- 使用等待队列进行工作

### 0.简介

```c
// struct file -linux - linux-2.6.0\include\linux\fs.h
// struct file_operations - linux-2.6.0\include\linux\fs.h
// generic_ro_fops - linux-2.6.0\include\linux\fs.h
// vfs_read() - linux-2.6.0\fs\read_write.c
```

### 1.注册/注销

1. 使用 **mknod** 创建 **/dev/so2_cdev** 字符设备节点

```c
// 在QUMU上使用mknod命令
mknod /dev/so2_cdev c 42 0
// 42是主设备号，0是此设备号，均在so2_cdev.c中定义过
```

> 此时只是创建了一个节点，要使用register_chrdev_region完成注册才能在/proc/devices中看到设备文件

2. 实现设备的注册和注销

```c
/* TODO 1/6: register char device region for MY_MAJOR and NUM_MINORS starting at MY_MINOR */
err = register_chrdev_region(MKDEV(MY_MAJOR, MY_MINOR),
			NUM_MINORS, MODULE_NAME);
/* TODO 1/1: unregister char device region, for MY_MAJOR and NUM_MINORS starting at MY_MINOR */
unregister_chrdev_region(MKDEV(MY_MAJOR, MY_MINOR),NUM_MINORS);
```

> MKDEV的意思是从主设备号MY_MAJOR开始注册次设备号MY_MINOR，注册NUM_MINORS个设备文件，如果当前主设备号下的设备文件数大于NUM_MINORS，则让主设备号＋1继续注册

### 2.注册一个已经注册过的主设备号

```c
// 使用 cat proc/devices 看已有的设备文件的主设备号，然后替换掉下面的宏定义
#define MY_MAJOR		42
```

> 此时会返回错误码 -16，\#define EBUSY  16   /* Device or resource busy */ 表示是当前设备正忙无法被注册

### 3.打开和关闭

> 打开和关闭字符设备文件

1. 初始化设备

```c
struct so2_device_data {
	/* TODO 2/1: add cdev member */
	struct cdev cdev;
	/* TODO 4/2: add buffer with BUFSIZ elements */
	char buffer[BUFSIZ];
	size_t size;
	/* TODO 7/2: extra members for home */
	wait_queue_head_t wq;
	int flag;
	/* TODO 3/1: add atomic_t access variable to keep track if file is opened */
	atomic_t access;
};
```

> 为当前设备文件建立一个结构体，成员有cdev结构体，该结构体用于在系统中注册字符设备(供cdev_init和cdev_add函数使用)，字符数组buffer用于读操作，size用于指示传输数据的大小，access是一个原子变量，用于计数实现阻塞其它进程干涉，这里只需要关注 TODO 2/1

2. 实现打开和释放函数

```c
// 在结构体定义static const struct file_operations so2_fops中
/* TODO 2/2: add open and release functions */
.open = so2_cdev_open,
.release = so2_cdev_release,

// 在函数so2_cdev_open中
/* TODO 3/1: inode->i_cdev contains our cdev struct, use container_of to obtain a pointer to so2_device_data */
// 获取当前设备文件的结构体
data = container_of(inode->i_cdev, struct so2_device_data, cdev);
// 让file指针指向当前设备文件，实现打开
file->private_data = data;
```

> container_of 宏用于从一个结构体成员的地址反推出成员所在的结构体的首地址，用法是container_of(ptr, type, member)，在这个例子中container_of 从 inode->i_cdev（一个struct cdev 类型的指针，指向so2_device_data的成员cdev）反推出它所在的 struct so2_device_data 结构体的首地址，从而获得设备的私有数据指针 data

3. 显示消息

```c
// 使用pr_info函数，与printf类似
```

4. 再次读取

```c
// 使用 cat /dev/so2_cdev
```

### 4.访问限制

> 使用原子变量限制设备访问

1. 在设备结构体中添加 `atomic_t` 变量

```c
// 在结构体so2_device_data中
/* TODO 3/1: add atomic_t access variable to keep track if file is opened */
atomic_t access;
```

2. 在模块初始化时对该变量进行初始化

```c
// 在函数so2_cdev_init中
/* TODO 3/1: set access variable to 0, use atomic_set */
atomic_set(&devs[i].access, 0);
// 这里的devs是设备文件的实例化，在设备文件结构体下有定义
// struct so2_device_data devs[NUM_MINORS];
```

3. 在打开函数中使用该变量限制对设备的访问。我们建议使用 `atomic_cmpxchg()`

```C
// 在函数so2_cdev_open中
/* TODO 3/2: return immediately if access is != 0, use atomic_cmpxchg */
if (atomic_cmpxchg(&data->access, 0, 1) != 0)
    return -EBUSY;
```

> static inline int atomic_cmpxchg(atomic_t *ptr, int old, int new)
> 这个函数可以在一个原子操作中检查变量的旧值并将其设为新值，在上面的例子中，它表示如果当前的access等于旧值0就将access设为1，不等于0就不修改，无论是否发生替换，atomic_cmpxchg函数都会返回ptr指向的原始值（也就是操作之前的值）。

4. 在释放函数中重置该变量以恢复对设备的访问权限

```C
// 在函数so2_cdev_release中
/* TODO 3/1: reset access variable to 0, use atomic_set */
atomic_set(&data->access, 0);
```

5. 模拟休眠

```
// 在函数so2_cdev_open中
set_current_state(TASK_INTERRUPTIBLE);
schedule_timeout(1000);
```

> `set_current_state(TASK_INTERRUPTIBLE);`把当前进程（`current`）的状态设置为可中断睡眠 (`TASK_INTERRUPTIBLE`)。在这个状态下，进程会被调度器认为是“睡着的”，直到有事件唤醒它
> `schedule_timeout(10 * HZ);`把当前进程从 CPU 调度队列里移走，并设置一个定时器，在 `10 * HZ` 个 **jiffies** 后唤醒它。`HZ` 是内核的时钟频率（例如在 x86 上常见是 100、250 或 1000），`10 * HZ` 表示 10 秒。如果在这段时间内进程收到信号，会提前被唤醒。

### 5.读操作

> 在驱动程序中实现读取函数

1. 在 `so2_device_data` 结构中保持一个缓冲区，并用 `MESSAGE` 宏的值进行初始化。缓冲区的初始化在模块的 `init` 函数中完成

```c
// struct so2_device_data
/* TODO 4/2: add buffer with BUFSIZ elements */
char buffer[BUFSIZ];
size_t size;
// static int so2_cdev_init(void)
/*TODO 4/2: initialize buffer with MESSAGE string */
memcpy(devs[i].buffer, MESSAGE, sizeof(MESSAGE));
devs[i].size = sizeof(MESSAGE);
```

2. 在读取调用时，将内核空间缓冲区的内容复制到用户空间缓冲区

使用 `copy_to_user()` 函数将信息从内核空间复制到用户空间

```c
// static ssize_t so2_cdev_read(file, user_buffer, size, offset)
// 首先定义传输的字节数to_read以及获取设备文件指针
struct so2_device_data *data =
		(struct so2_device_data *) file->private_data;
size_t to_read = (size > data->size - *offset) ? (data->size - *offset) : size;
/* TODO 4/4: Copy data->buffer to user_buffer, use copy_to_user */
if (copy_to_user(user_buffer, data->buffer + *offset, to_read) != 0)
    return -EFAULT;
*offset += to_read;

return to_read;
```

> 需要将请求的字节数size和内部缓冲区大小data->size - *offset作比较，有可能请求的字节数要超过内部缓存区大小，从而引发错误，其实这里应该是判断 size+\*offset > data->size，可能为了防止越界写成了减去，注意要更新偏移参数，以便于用户达到文件内部缓冲区末尾时退出

> 读取函数so2_cdev_read调用返回的值是从内核空间缓冲区传输到用户空间缓冲区的字节数

### 6.写操作

### 7.ioctl 操作

### 8.带消息的 ioctl

### 9.使用等待队列的 ioctl

### 10.O_NONBLOCK 实现



## 四  I/O访问和中断

实验目标：

- 与外围设备进行通信

- 实现中断处理程序
- 将中断与进程上下文同步

关键词：IRQ，I/O 端口，I/O 地址，基地址，UART，request_region，release_region，inb，outb

### 0.简介

```c
//resource - /inclue/linux/ioport.h
struct resource {
	resource_size_t start;
	resource_size_t end;
	const char *name;
	unsigned long flags;
	unsigned long desc;
	struct resource *parent, *sibling, *child;
};
//request_region - /inclue/linux/ioport.h
#define request_region(start,n,name)\
		__request_region(&ioport_resource, (start), (n), (name), 0)
//__request_region() - /kernel/resource.c
struct resource *__request_region(struct resource *parent,
				  resource_size_t start, resource_size_t n,
				  const char *name, int flags)
//request_irq() - /include/linux/interrupt.h
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
//request_threaded_irq() - /kernel/irq/manage.c
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			 irq_handler_t thread_fn, unsigned long irqflags,
			 const char *devname, void *dev_id)
```

### 实现键盘驱动程序

  目标是创建一个使用键盘IRQ的驱动程序，检查传入的按键代码并将其存储在缓冲区中。通过字符设备驱动程序，用户空间可以访问该缓冲区。

> 如果说上一个实验字符设备驱动程序是关于如何驱动外设如何实现物理设备的读写I/O操作，那么这个实验是关于如何通过中断来操控外设

### 1.请求I/O端口

```c
// 在 kbd_init 函数中
/* TODO 1/8: request the keyboard I/O ports */
if (request_region(I8042_DATA_REG+1, 1, MODULE_NAME) == NULL) {
    err = -EBUSY;
    goto out_unregister;
}
if (request_region(I8042_STATUS_REG+1, 1, MODULE_NAME) == NULL) {
    err = -EBUSY;
    goto out_release_region;
}
out_release_regions:
	release_region(I8042_STATUS_REG+1, 1);
out_release_region:
	release_region(I8042_DATA_REG+1, 1);

out_unregister:
	unregister_chrdev_region(MKDEV(KBD_MAJOR, KBD_MINOR),
				 KBD_NR_MINORS);
out:
	return err;
// 在 kbd_exit 函数中
/* TODO 1/2: release keyboard I/O ports */
release_region(I8042_STATUS_REG+1, 1);
release_region(I8042_DATA_REG+1, 1);
```

> 请求 I/O 端口后，可以在/proc/ioports中找到自己的端口号
> 如果遇到报错提示-EBUSY，可能是因为没有执行release_region或者unregister_chrdev_region，需要在init函数开头写release_region或者unregister_chrdev_region然后重新插入模块

### 2.中断处理例程

> 所谓中断处理例程，就是中断发生时会执行的代码功能函数，一般是xxx_interrupt_handler()

```c
// 自己写的一个函数 kbd_interrupt_handle 注意类型是 irqreturn_t
/* TODO 2/27: implement interrupt handler */
irqreturn_t kbd_interrupt_handle(int irq_no, void *dev_id)
{
	// 当中断发生时输出
    pr_info("You just pressed A key!\n");
    
	/* TODO 3: read the scancode */
	/* TODO 3/2: interpret the scancode */
	/* TODO 3/2: display information about the keystrokes */
	/* TODO 3/7: store ASCII key to buffer */
	return IRQ_NONE;
}
// 在 kbd_init 函数中, 注册中断处理例程，使用 request_irq
/* TODO 2/7: Register IRQ handler for keyboard IRQ (IRQ 1). */
err = request_irq(I8042_KBD_IRQ,
        kbd_interrupt_handle,
        IRQF_SHARED, MODULE_NAME, &devs[0]);
if (err != 0) {
    pr_err("request_irq failed: %d\n", err);
    goto out_release_regions;
}
```

> 使用request_irq注册中断例程后，能在/proc/interrupts中看到自己的中断程序 kbd

![image-20250817211748806](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250817211748806.png)

> 按照“注意”的提示使用 `QEMU_DISPLAY=gtk make boot` 是无法启动成功的，因为QEMU中的Makefile指定需要下载yotco2.4版本的镜像系统，但是链接早就失效了，必须修改QEMU中的Makefile的yotco版本号为4.1才能正常下载系统镜像从而启动，如下图所示修改，如果下载的很慢或者还是下载不了，可能需要使用代理，使用clash-verge的TUN模式可以让虚拟机走代理

<img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250817211057473.png" alt="image-20250817211057473" style="zoom:67%;" />

> 没有报错之后应该能正常启动系统，如下所示

<img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250817211345350.png" alt="image-20250817211345350" style="zoom:67%;" />

> 但此时要退出来，因为没有在Docker内执行 `make copy`，执行前系统内是没有 `skels` 这个文件夹的

在QEMU中按下键盘按键会触发中断例程程序，使用dmesg可以看到：

<img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250817211707033.png" alt="image-20250817211707033" style="zoom: 80%;" />

### 3.将 ASCII 键存储到缓冲区

#### 读取数据存储器

```c
// 使用函数 inb 读取 I/O 端口的数据，只读一个字符大小的数据(1 Byte)
static inline u8 i8042_read_data(void)
{
	u8 val;
	/* TODO 3: Read DATA register (8 bits). */
	val = inb(I8042_DATA_REG);  // 此时读取的是寄存器中的扫描码，还需要转换才能成为ASCII码
	return val;
}
// 在 kbd_interrupt_handle 函数中
/* TODO 3: read the scancode */
scancode = i8042_read_data();
```

#### 解释扫描码

```c
// 在 kbd_interrupt_handle 函数中
/* TODO 3/2: interpret the scancode */
pressed = is_key_press(scancode); // 此函数能判断扫描码是按下键还是释放键
ch = get_ascii(scancode);  // 此函数能将扫描码转换为ASCII码
/* TODO 3/2: display information about the keystrokes */
pr_info("IRQ %d: scancode=0x%x (%u) pressed=%d ch=%c\n",
    irq_no, scancode, scancode, pressed, ch);

static int get_ascii(unsigned int scancode)
{
	static char *row1 = "1234567890";
	static char *row2 = "qwertyuiop";
	static char *row3 = "asdfghjkl";
	static char *row4 = "zxcvbnm";

	scancode &= ~SCANCODE_RELEASED_MASK;
	if (scancode >= 0x02 && scancode <= 0x0b)
		return *(row1 + scancode - 0x02);
	if (scancode >= 0x10 && scancode <= 0x19)
		return *(row2 + scancode - 0x10);
	if (scancode >= 0x1e && scancode <= 0x26)
		return *(row3 + scancode - 0x1e);
	if (scancode >= 0x2c && scancode <= 0x32)
		return *(row4 + scancode - 0x2c);
	if (scancode == 0x39)
		return ' ';
	if (scancode == 0x1c)
		return '\n';
	return '?';
}

static int is_key_press(unsigned int scancode)
{
	return !(scancode & SCANCODE_RELEASED_MASK);
}
```

#### 将字符存储到缓冲区

```c
// 在 kbd_interrupt_handle 函数中
/* TODO 3/7: store ASCII key to buffer */
if (pressed) {
    struct kbd *data = (struct kbd *)dev_id; // 获取设备结构体
	// 使用自旋锁，确保共享资源缓冲区是同步访问的，需要先在init函数中初始化
    spin_lock(&data->lock); 
    // 读取字符，将 I/O 端口的字符读取到设备缓冲区中，以便于用户空间读取
    put_char(data, ch);
    spin_unlock(&data->lock);
}
// 在 kbd_init 函数中
/* TODO 3: initialize spinlock */
spin_lock_init(&devs[0].lock);
// put_char 函数定义
static void put_char(struct kbd *data, char c)
{
    // 如果缓冲区中的字符数量大于缓冲区的容量，则丢弃当前读取的字符
	if (data->count >= BUFFER_SIZE)
		return;
	// put->idx指向缓冲区下一个写入位置的索引
	data->buf[data->put_idx] = c;
    // 更新索引
	data->put_idx = (data->put_idx + 1) % BUFFER_SIZE;
	data->count++;
}
```

### 4.读取缓冲区

> 为了访问键盘记录器的数据，我们需要将其发送到用户空间。我们将使用 */dev/kbd* 字符设备来实现这一点。当从该设备读取数据时，我们将从内核空间的缓冲区中获取按键数据

```c
// 使用 get_char() 从缓冲区中读取一个字符，并使用 put_user() 将其存储到用户缓冲区中
static bool get_char(char *c, struct kbd *data)
{
	/* TODO 4/6: get char from buffer; update count and get_idx */
	if (data->count > 0) { // 如果缓冲区中的字符数量大于0，则读取字符
		*c = data->buf[data->get_idx];
		data->get_idx = (data->get_idx + 1) % BUFFER_SIZE;
		data->count--;
		return true;
	}
	return false;
}
// 在 kbd_read 函数中
static ssize_t kbd_read(struct file *file,  char __user *user_buffer,
			size_t size, loff_t *offset)
{
	struct kbd *data = (struct kbd *) file->private_data;
	size_t read = 0;
	/* TODO 4/18: read data from buffer */
	unsigned long flags;
	char ch;
	bool more = true;
	while (size--) {
        // 在读取函数中，使用 spin_lock_irqsave() 和 spin_unlock_irqrestore() 进行加锁
		spin_lock_irqsave(&data->lock, flags);
		more = get_char(&ch, data);
		// 原子上下文中不允许访问用户空间
        spin_unlock_irqrestore(&data->lock, flags);
		if (!more)
			break;
		// 关键的一句在这，使用put_user函数将字符存储到用户缓存区
        if (put_user(ch, user_buffer++))  // 参数是ch(字符)和user_buffer(传入的用户缓冲区)
			return -EFAULT;
		read++;
	}
	return read;
}
```

为了测试是否成功实现了从键盘这一物理设备驱动程序的中断中实现数据读取，应该先使用`mknod`创建`/dev/kbd`字符设备驱动程序，正如在`kbd_init`函数中注册字符驱动设备文件的函数 `register_chrdev_region` 中的参数一样，要使用同样的`KBD_MAJOR`和`MODULE_NAME`以及`KBD_MINOR`，也就是 `42`、`kbd`、`0`，使用命令 `mknod /dev/kbd c 42 0` 在QEMU中创建设备驱动文件kbd，构建复制和启动QEMU，加载模块后，能在`/proc/devices`中看到自己的设备文件`42 kbd`，最后

```c
// 使用 cat /dev/kbd 读取缓冲区中的数据吧，因为 cat 是用户空间使用的命令，因此此时应该显示
```

<img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250817214906662.png" alt="image-20250817214906662" style="zoom: 80%;" />

### 5.重置缓冲区

> 使用 `echo "clear" > /dev/kbd` 将数据写入设备

```c
// 实现 reset_buffer
static void reset_buffer(struct kbd *data)
{
	/* TODO 5/3: reset count, put_idx, get_idx */
	data->count = 0;
	data->put_idx = 0;
	data->get_idx = 0;
}
// 当写操作发生时，应该执行...
/* TODO 5/12: add write operation and reset the buffer */
static ssize_t kbd_write(struct file *file, const char __user *user_buffer,
			 size_t size, loff_t *offset)
{
	struct kbd *data = (struct kbd *) file->private_data;
	unsigned long flags;

	spin_lock_irqsave(&data->lock, flags);
	reset_buffer(data);
	spin_unlock_irqrestore(&data->lock, flags);

	return size;
}
// 添加写操作
static const struct file_operations kbd_fops = {
	.owner = THIS_MODULE,
	.open = kbd_open,
	.release = kbd_release,
	.read = kbd_read,
	/* TODO 5: add write operation */
	.write = kbd_write,
};
```

此时执行命令 `echo "clear" > /dev/kbd` 会让缓冲区清空

<img src="https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250817215433645.png" alt="image-20250817215433645" style="zoom:80%;" />



## 五  延迟工作

实验目标

- 理解延迟工作（即在稍后时间执行的代码）
- 实现使用延迟工作的常见任务
- 理解延迟工作的同步特性

> 关键词：softirq、tasklet、struct tasklet_struct、下半部处理程序、jiffies、HZ、timer、struct timer_list、spin_lock_bh、spin_unlock_bh、workqueue、struct work_struct、内核线程、events/x

```c
使用 LABS="deferred_work" make skels 构建骨架
```

### 0.简介

```markdown
jiffies：全局变量，保存了自系统启动以来经过的"时钟节拍数"，每次时钟中断触发时，内核会把它加1，类型为unsigned long，用于时间比较和定时。
时间换算：秒数 = jiffies / HZ
HZ宏：内核每秒时钟中断数
- x86 每秒中断1000次
- ARM/嵌入式可能100或250次
```

```c
timer_list 是内核定时器机制的核心结构体
struct timer_list {
	/* 把定时器挂到全局定时器链表/红黑树上的节点 */
	struct hlist_node entry;
	/* 超时时间点（以 jiffies 为单位） */
	unsigned long expires;
	/* 回调函数指针 */
	void (*function)(struct timer_list *);
	/* 回调函数参数（新内核用 from_timer() 宏来获取） */
	u32 flags;
	/* 调试和统计字段略 */
};
timer_list负责在延迟时间expires到时把work_struct放进wrokqueue
```

```c
spin_lock_bh()会获取一个自旋锁，保证多cpu并发访问的互斥性，并且关闭本地软中断的下半部处理
```

### 1.定时器

> 创建一个简单的内核模块，在模块的内核加载后的第 *TIMER_TIMEOUT* 秒显示一条消息

1. 创建骨架 `LABS="deferred_work" make skels`
2. 在timer.c文件中修改

```c
// 1.初始化一个timer
static int __init timer_init(void)
{
	pr_info("[timer_init] Init module\n");
	/* TODO 1: initialize timer */
	timer_setup(&timer, timer_handler, 0);
	/* TODO 1: schedule timer for the first time */
	mod_timer(&timer, jiffies + TIMER_TIMEOUT * HZ);
	return 0;
}
// 2.在定时器处理程序中打印信息
static void timer_handler(struct timer_list *tl)
{
	/* TODO 1/4: print a message */
	static size_t nseconds;
	nseconds += TIMER_TIMEOUT;
	pr_info("[timer_handler] nseconds = %d\n", nseconds);
}
// 3.注销定时器
static void __exit timer_exit(void)
{
	pr_info("[timer_exit] Exit module\n");
	/* TODO 1: cleanup; make sure the timer is not running after we exit */
	del_timer_sync(&timer);
}
```

![image-20250903153027875](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250903153027875.png)

### 2.周期性定时器

> 修改前面的模块，使消息每隔 TIMER_TIMEOUT 秒显示一次

```c
// 在定时器处理程序中，递归调用处理程序
/* TODO 2: rechedule timer */
mod_timer(tl, jiffies + TIMER_TIMEOUT * HZ);
```

![image-20250903153503385](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250903153503385.png)

### 3.使用 ioctl 控制定时器

> 在从用户空间接收到 ioctl 调用后的第 N 秒显示有关当前进程的信息，N作为 ioctl 参数传递

```c
// 1.在设备文件结构体中添加定时器字段
/* TODO 1: add timer */
struct timer_list timer;

// 2.实现定时器处理程序
/* TODO 1/44: implement timer handler */
struct my_device_data *my_data = from_timer(my_data, tl, timer);
pr_info("[timer_handler] pid = %d, comm = %s\n",
    current->pid, current->comm);

// 3.在中断程序中，实现定时器调度和取消定时器
case MY_IOCTL_TIMER_SET:
    /* TODO 1: schedule timer */
    mod_timer(&my_data->timer, jiffies + arg * HZ);
    break;
case MY_IOCTL_TIMER_CANCEL:
    /* TODO 1: cancel timer */
    del_timer(&my_data->timer);
    break;

// 4.初始化一个定时器
/* TODO 1: Initialize timer. */
timer_setup(&dev.timer, timer_handler, 0);

// 5.注销定时器
/* TODO 1: Cleanup: make sure the timer is not running after exiting. */
del_timer_sync(&dev.timer);
```

- 使用kernel/下的makenode脚本注册设备字符文件，然后插入模块
- 在user/下执行 `./test s 3` 在三秒后启用定时器

![image-20250903161654957](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250903161654957.png)

- 执行 `./test c` 停用定时器

![image-20250903161731535](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250903161731535.png)

### 4.阻塞操作

> 在定时器处理程序中执行阻塞操作，由于定时器处理程序运行在中断上下文中，所以当执行阻塞操作时，会发生OOPS错误

```c
// 1.为了区分定时器处理程序中的功能，在设备结构体中添加flag字段
/* TODO 2: add flag */
int flag;
// 2.在定时器处理程序中，为不同的flag设置不同的功能
	/* TODO 2/38: check flags: TIMER_TYPE_SET or TIMER_TYPE_ALLOC */
switch (my_data->flag) {
	case TIMER_TYPE_SET:
		break;
	case TIMER_TYPE_ALLOC:
#ifdef ALLOC_IO_DIRECT
		alloc_io();
#else
		
#endif
		break;
    default:
		break;
}
// 3.设置不同参数下的中断程序标志
case MY_IOCTL_TIMER_SET:
    /* TODO 2: set flag */
    my_data->flag = TIMER_TYPE_SET;
case MY_IOCTL_TIMER_ALLOC:
    /* TODO 2/2: set flag and schedule timer */
    my_data->flag = TIMER_TYPE_ALLOC;
    mod_timer(&my_data->timer, jiffies + arg * HZ);
    break;
// 4.初始化设备flag标志位
/* TODO 2: Initialize flag. */
dev.flag = TIMER_TYPE_NONE;
```

![image-20250903164345594](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250903164345594.png)

### 5.工作队列

> 使用工作队列调用alloc_io()，即在定时器处理程序中安排一个工作项。工作项处理程序在进程上下文中运行

```C
// 1.在设备结构体中添加工作项字段
/* TODO 3: add work */
struct work_struct work;

// 2.定义工作项处理程序
/* TODO 3/4: define work handler */
static void work_handler(struct work_struct *work)
{
	alloc_io();
}

// 3.执行工作项
/* TODO 3: schedule work */
schedule_work(&my_data->work);

// 4.初始化工作项
/* TODO 3: Initialize work. */
INIT_WORK(&dev.work, work_handler);

// 5.注销工作项
/* TODO 3: Cleanup: make sure the work handler is not scheduled. */
flush_scheduled_work();
```

![image-20250904084842901](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250904084842901.png)

### 6.内核线程

> 创建一个显示当前进程标识符的内核线程

```c
// 1.初始化等待队列和标志位
/* TODO/4: init the waitqueues and flags */
init_waitqueue_head(&wq_stop_thread);
atomic_set(&flag_stop_thread, 0);
init_waitqueue_head(&wq_thread_terminated);
atomic_set(&flag_thread_terminated, 0);

// 2.创建和开始内核线程
/* TODO: create and start the kernel thread */
kthread_run(my_thread_f, NULL, "%skthread%d", "my", 0);

// 3.定义线程处理函数
int my_thread_f(void *data)
{
	pr_info("[my_thread_f] Current process id is %d (%s)\n",
		current->pid, current->comm);
    
// 4.等待命令去移除在等待队列中的模块
/* TODO: Wait for command to remove module on wq_stop_thread queue. */
wait_event_interruptible(wq_stop_thread,atomic_read(&flag_stop_thread) != 0);

// 5.设置标志位标记内核线程结束
/* TODO: set flag to mark kernel thread termination */
atomic_set(&flag_thread_terminated, 1);

// 6.拉起中断，通知其他进程可以上来了
/* TODO: notify the unload process that we have exited */
wake_up_interruptible(&wq_thread_terminated);

	pr_info("[my_thread_f] Exiting\n");
	do_exit(0);
}

// 7.线程退出函数
static void __exit kthread_exit(void)
{
	/* TODO/2: notify the kernel thread that its time to exit */
	atomic_set(&flag_stop_thread, 1);
	wake_up_interruptible(&wq_stop_thread);
	/* TODO: wait for the kernel thread to exit */
	wait_event_interruptible(wq_thread_terminated, atomic_read(&flag_thread_terminated) != 0);
	pr_info("[kthread_exit] Exit module\n");
}
```

![image-20250904091929236](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250904091929236.png)



## 六  块设备驱动程序

实验目标：

- 了解 Linux 中 I/O 子系统的行为
- 在块设备的结构和函数上进行实际操作
- 通过解决练习，掌握块设备的 API 使用基础技能

### 0.简介

| 符号                             | 定义位置（头文件）    | 简要说明                               |
| -------------------------------- | --------------------- | -------------------------------------- |
| `struct bio`                     | `<linux/bio.h>`       | 块 I/O 的基本结构，包含 bio_vec 数组等 |
| `struct bio_vec`                 | `<linux/bio.h>`       | I/O segment 描述：页面、偏移量、长度   |
| `bio_for_each_segment`           | `<linux/bio.h>`（宏） | 遍历 bio 中每个 segment 的迭代宏       |
| `struct gendisk`                 | `<linux/genhd.h>`     | 通用磁盘结构，代表块设备抽象           |
| `struct block_device_operations` | `<linux/blkdev.h>`    | 块设备操作函数集合                     |
| `struct request`                 | `<linux/blkdev.h>`    | 块 I/O 请求结构，含多个 bio            |

### 1.块设备

> 创建一个内核模块，允许你注册或者取消注册块设备

```c
// 1.在初始化函数中注册块设备
/* TODO 1/5: register block device */
err = register_blkdev(MY_BLOCK_MAJOR, MY_BLKDEV_NAME);
if (err < 0) {
    printk(KERN_ERR "register_blkdev: unable to register\n");
    return err;
}

// 2.在卸载函数中注销块设备
/* TODO 1/1: unregister block device */
unregister_blkdev(MY_BLOCK_MAJOR, MY_BLKDEV_NAME);

// 3.定义设备号和设备名
#define MY_BLOCK_MAJOR		240
#define MY_BLKDEV_NAME		"mybdev"
```

```markdown
这里只是注册块设备，还没有创建块设备，只是告诉内核这里有块设备
```

![image-20250904161600953](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250904161600953.png)

### 2.磁盘注册

> 添加与驱动程序关联的硬盘，填充请求函数以处理请求

```c
// 1.完善请求处理函数
static blk_status_t my_block_request(struct blk_mq_hw_ctx *hctx,
				     const struct blk_mq_queue_data *bd)
{
	struct request *rq;
	struct my_block_dev *dev = hctx->queue->queuedata;
	/* TODO 2: get pointer to request */
	rq = bd->rq;
	/* TODO 2: start request processing. */
	blk_mq_start_request(rq);
	/* TODO 2/5: check fs request. Return if passthrough. */
	if (blk_rq_is_passthrough(rq)) {
		printk(KERN_NOTICE "Skip non-fs request\n");
		blk_mq_end_request(rq, BLK_STS_IOERR);
		goto out;
	}
	/* TODO 2/6: print request information */
	printk(KERN_LOG_LEVEL
		"request received: pos=%llu bytes=%u "
		"cur_bytes=%u dir=%c\n",
		(unsigned long long) blk_rq_pos(rq),
		blk_rq_bytes(rq), blk_rq_cur_bytes(rq),
		rq_data_dir(rq) ? 'W' : 'R');

#if USE_BIO_TRANSFER == 1
/* TODO 6/1: process the request by calling my_xfer_request */
#else
/* TODO 3/3:process the request by calling my_block_transfer */
#endif

	/* TODO 2/1: end request successfully */
	blk_mq_end_request(rq, BLK_STS_OK);

out:
	return BLK_STS_OK;
}

// 2.在初始化函数中创建块设备
/* TODO 2/3: create block device using create_block_device */
err = create_block_device(&g_dev);
if (err < 0)
    goto out;
return 0;
/* TODO 2/1: unregister block device in case of an error */
unregister_blkdev(MY_BLOCK_MAJOR, MY_BLKDEV_NAME);

// 3.在卸载函数中删除块设备
/* TODO 2/1: cleanup block device using delete_block_device */
delete_block_device(&g_dev);
```

![image-20250904165309029](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250904165309029.png)

### 3.RAM磁盘

> 创建一个 RAM 磁盘：对设备的请求将导致在一个内存区域中进行读写操作

```c
static void my_block_transfer(struct my_block_dev *dev, sector_t sector,
		unsigned long len, char *buffer, int dir)
{
	unsigned long offset = sector * KERNEL_SECTOR_SIZE;

	/* check for read/write beyond end of block device */
	if ((offset + len) > dev->size)
		return;

	/* TODO 3/4: read/write to dev buffer depending on dir */
	if (dir == 1)		/* write */
		memcpy(dev->data + offset, buffer, len);
	else
		memcpy(buffer, dev->data + offset, len);
}
/* TODO 3/3: process the request by calling my_block_transfer */
my_block_transfer(dev, blk_rq_pos(rq),
          blk_rq_bytes(rq),
          bio_data(rq->bio), rq_data_dir(rq));
```

![image-20250905091140376](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250905091140376.png)

### 4.从磁盘读取数据

> 从内核直接读取 `PHYSICAL_DISK_NAME` 磁盘（`/dev/vdb`）的数据。

```c
/* TODO 4/3: fill bio (bdev, sector, direction) */
bio->bi_disk = bdev->bd_disk;
bio->bi_iter.bi_sector = 0;
bio->bi_opf = dir;
/* TODO 4/3: submit bio and wait for completion */
printk(KERN_LOG_LEVEL "[send_test_bio] Submiting bio\n");
submit_bio_wait(bio);
printk(KERN_LOG_LEVEL "[send_test_bio] Done bio\n");
/* TODO 4/3: read data (first 3 bytes) from bio buffer and print it */
buf = kmap_atomic(page);
printk(KERN_LOG_LEVEL "read %02x %02x %02x\n", buf[0], buf[1], buf[2]);
kunmap_atomic(buf);
/* TODO 4/5: get block device in exclusive mode */
bdev = blkdev_get_by_path(name, FMODE_READ | FMODE_WRITE | FMODE_EXCL, THIS_MODULE);
if (IS_ERR(bdev)) {
    printk(KERN_ERR "blkdev_get_by_path\n");
    return NULL;
}
/* TODO 4/1: put block device */
blkdev_put(bdev, FMODE_READ | FMODE_WRITE | FMODE_EXCL);
```

![image-20250905092023588](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250905092023588.png)

### 5.将数据写入磁盘

> 在磁盘上写入消息（`BIO_WRITE_MESSAGE`）

```c
/* TODO 5/5: write message to bio buffer if direction is write */
if (dir == REQ_OP_WRITE) {
    buf = kmap_atomic(page);
    memcpy(buf, BIO_WRITE_MESSAGE, strlen(BIO_WRITE_MESSAGE));
    kunmap_atomic(buf);
}
static void __exit relay_exit(void)
{
	/* TODO 5/1: send test write bio */
	send_test_bio(phys_bdev, REQ_OP_WRITE);
	close_disk(phys_bdev);
}
```

![image-20250905093238273](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250905093238273.png)

### 6.在struct bio级别处理请求队列中的请求

> 处理来自请求队列中所有 `struct bio` 结构的所有 `struct bio_vec` 结构（也称为段）

```c
/* TODO 6/0: use bios for read/write requests */
#define USE_BIO_TRANSFER	1
/* TODO 6/10: iterate segments */
struct bio_vec bvec;
struct req_iterator iter;
rq_for_each_segment(bvec, req, iter) {
    sector_t sector = iter.iter.bi_sector;
    unsigned long offset = bvec.bv_offset;
    size_t len = bvec.bv_len;
    int dir = bio_data_dir(iter.bio);
    char *buffer = kmap_atomic(bvec.bv_page);
    printk(KERN_LOG_LEVEL "%s: buf %8p offset %lu len %u dir %d\n", __func__, buffer, offset, len, dir);
    /* TODO 6/3: copy bio data to device buffer */
    my_block_transfer(dev, sector, len, buffer + offset, dir);
    kunmap_atomic(buffer);
}
/* TODO 6/1: process the request by calling my_xfer_request */
my_xfer_request(dev, rq);
```

![image-20250905100157263](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250905100157263.png)

## 七  文件系统驱动程序(第一部分)

实验目标：

- 了解 Linux 中虚拟文件系统（VFS）的知识，理解有关“inode”、“dentry”、“文件”、“超级块”和数据块的概念。
- 理解在 VFS 内挂载文件系统的过程。
- 了解各种文件系统类型，并理解具有物理支持（在磁盘上）和没有物理支持的文件系统之间的区别。

### 1.注册和注销myfs文件系统

```c
static struct dentry *myfs_mount(struct file_system_type *fs_type,
		int flags, const char *dev_name, void *data)
{
	/* TODO 1/1: call superblock mount function */
	return mount_nodev(fs_type, flags, data, myfs_fill_super);
}
/* TODO 1/6: define file_system_type structure */
static struct file_system_type myfs_fs_type = {
	.owner		= THIS_MODULE,
	.name		= "myfs",
	.mount		= myfs_mount,
	.kill_sb	= kill_litter_super,
};
/* TODO 1/1: register */
err = register_filesystem(&myfs_fs_type);
/* TODO 1/1: unregister */
unregister_filesystem(&myfs_fs_type);
```

![image-20250908093600392](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250908093600392.png)

### 2.完成myfs超级块填充

> 为了能够挂载文件系统，我们需要填充其超级块的字段，即类型为 `struct super_block` 的通用 VFS 结构，从而完成文件系统超级块的初始化

```c
/* TODO 2/4: define super_operations structure */
static const struct super_operations myfs_ops = {
	.statfs		= simple_statfs,
	.drop_inode	= generic_drop_inode,
};
/* TODO 2/5: fill super_block
 *   - blocksize, blocksize_bits
 *   - magic
 *   - super operations
 *   - maxbytes
 */
sb->s_maxbytes = MAX_LFS_FILESIZE;
sb->s_blocksize = MYFS_BLOCKSIZE;
sb->s_blocksize_bits = MYFS_BLOCKSIZE_BITS;
sb->s_magic = MYFS_MAGIC;
sb->s_op = &myfs_ops;
```

### 3.初始化myfs根索引节点

> 为了成功完成挂载操作，还需要初始化根索引节点，根索引节点是文件系统根目录（即 `/`）的索引节点。

```c
/* TODO 3/3: fill inode structure
 *     - mode
 *     - uid
 *     - gid
 *     - atime,ctime,mtime
 *     - ino
 */
inode_init_owner(inode, dir, mode);
inode->i_atime = inode->i_mtime = inode->i_ctime = current_time(inode);
inode->i_ino = 1;
/* TODO 3/2: set inode operations for dir inodes. */
inode->i_op = &simple_dir_inode_operations;
inode->i_fop = &simple_dir_operations;
/* TODO 3/1: directory inodes start off with i_nlink == 2 (for "." entry).
 * Directory link count should be incremented (use inc_nlink).
 */
inc_nlink(inode);
```

### 4.测试myfs的挂载和卸载

> 执行脚本 `./test-myfs.sh` 测试整个功能

![image-20250908095833026](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250908095833026.png)

### 5.minfs

> 接下来实现一个支持磁盘的文件系统

#### 1.注册和注销minfs文件系统

```c
static struct dentry *minfs_mount(struct file_system_type *fs_type,
		int flags, const char *dev_name, void *data)
{
	/* TODO 1/1: call superblock mount function */
	return mount_bdev(fs_type, flags, dev_name, data, minfs_fill_super);
}
static struct file_system_type minfs_fs_type = {
	.owner		= THIS_MODULE,
	.name		= "minfs",
	/* TODO 1/3: add mount, kill_sb and fs_flags */
	.mount		= minfs_mount,
	.kill_sb	= kill_block_super,
	.fs_flags	= FS_REQUIRES_DEV,
};
```

![image-20250908144034036](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250908144034036.png)

#### 2.完善minfs超级块

> 为了能够挂载文件系统，你需要在 `minfs_fill_super` 函数中填充超级块（即类型为 `struct super_block` 的结构体）

```c
struct inode *myfs_get_inode(struct super_block *sb, const struct inode *dir,
		int mode)
{
	struct inode *inode = new_inode(sb);

	if (!inode)
		return NULL;

	/* TODO 3/3: fill inode structure
	 *     - mode
	 *     - uid
	 *     - gid
	 *     - atime,ctime,mtime
	 *     - ino
	 */
	inode_init_owner(inode, dir, mode);
	inode->i_atime = inode->i_mtime = inode->i_ctime = current_time(inode);
	inode->i_ino = 1;

	/* TODO 5/1: Init i_ino using get_next_ino */
	inode->i_ino = get_next_ino();

	/* TODO 6/1: Initialize address space operations. */
	inode->i_mapping->a_ops = &minfs_aops;

	if (S_ISDIR(mode)) {
		/* TODO 3/2: set inode operations for dir inodes. */
		inode->i_op = &simple_dir_inode_operations;
		inode->i_fop = &simple_dir_operations;

		/* TODO 5/1: use myfs_dir_inode_operations for inode
		 * operations (i_op).
		 */
		inode->i_op = &minfs_dir_inode_operations;

		/* TODO 3/1: directory inodes start off with i_nlink == 2 (for "." entry).
		 * Directory link count should be incremented (use inc_nlink).
		 */
		inc_nlink(inode);
	}

	/* TODO 6/4: Set file inode and file operations for regular files
	 * (use the S_ISREG macro).
	 */
	if (S_ISREG(mode)) {
		inode->i_op = &minfs_file_inode_operations;
		inode->i_fop = &minfs_file_operations;
	}

	return inode;
}
static int minfs_fill_super(struct super_block *s, void *data, int silent)
{
	struct minfs_sb_info *sbi;
	struct minfs_super_block *ms;
	struct inode *root_inode;
	struct dentry *root_dentry;
	struct buffer_head *bh;
	int ret = -EINVAL;

	sbi = kzalloc(sizeof(struct minfs_sb_info), GFP_KERNEL);
	if (!sbi)
		return -ENOMEM;
	s->s_fs_info = sbi;

	/* Set block size for superblock. */
	if (!sb_set_blocksize(s, MINFS_BLOCK_SIZE))
		goto out_bad_blocksize;

	/* TODO 2/3: Read block with superblock. It's the first block on
	 * the device, i.e. the block with the index 0. This is the index
	 * to be passed to sb_bread().
	 */
	bh = sb_bread(s, MINFS_SUPER_BLOCK);
	if (bh == NULL)
		goto out_bad_sb;

	/* TODO 2/1: interpret read data as minfs_super_block */
	ms = (struct minfs_super_block *) bh->b_data;

	/* TODO 2/2: check magic number with value defined in minfs.h. jump to out_bad_magic if not suitable */
	if (ms->magic != MINFS_MAGIC)
		goto out_bad_magic;

	/* TODO 2/2: fill super_block with magic_number, super_operations */
	s->s_magic = MINFS_MAGIC;
	s->s_op = &minfs_ops;

	/* TODO 2/2: Fill sbi with rest of information from disk superblock
	 * (i.e. version).
	 */
	sbi->version = ms->version;
	sbi->imap = ms->imap;

	/* allocate root inode and root dentry */
	/* TODO 2/0: use myfs_get_inode instead of minfs_iget */
	root_inode = myfs_get_inode(s, MINFS_ROOT_INODE, S_IFDIR | S_IRWXU | S_IRGRP |
			S_IXGRP | S_IROTH | S_IXOTH);
	if (!root_inode)
		goto out_bad_inode;

	root_dentry = d_make_root(root_inode);
	if (!root_dentry)
		goto out_iput;
	s->s_root = root_dentry;

	/* Store superblock buffer_head for further use. */
	sbi->sbh = bh;

	return 0;

out_iput:
	iput(root_inode);
out_bad_inode:
	printk(LOG_LEVEL "bad inode\n");
out_bad_magic:
	printk(LOG_LEVEL "bad magic number\n");
	brelse(bh);
out_bad_sb:
	printk(LOG_LEVEL "error reading buffer_head\n");
out_bad_blocksize:
	printk(LOG_LEVEL "bad block size\n");
	s->s_fs_info = NULL;
	kfree(sbi);
	return ret;
}
```

![image-20250908153002651](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250908153002651.png)

> 挂载成功

#### 3.创建和销毁minfs索引节点

```c
static struct inode *minfs_alloc_inode(struct super_block *s)
{
	struct minfs_inode_info *mii;

	/* TODO 3/4: Allocate minfs_inode_info. */
	mii = kzalloc(sizeof(struct minfs_inode_info), GFP_KERNEL);
	if (mii == NULL)
		return NULL;

	/* TODO 3/1: init VFS inode in minfs_inode_info */
	inode_init_once(&mii->vfs_inode);

	return &mii->vfs_inode;
}
static void minfs_destroy_inode(struct inode *inode)
{
	/* TODO 3/1: free minfs_inode_info */
	kfree(container_of(inode, struct minfs_inode_info, vfs_inode));
}
```

#### 4.初始化minfs根索引节点

```c
/* TODO 4/2: Read block with inodes. It's the second block on
 * the device, i.e. the block with the index 1. This is the index
 * to be passed to sb_bread().
 */
if (!(bh = sb_bread(s, MINFS_INODE_BLOCK)))
    goto out_bad_sb;
/* TODO 4/1: Get inode with index ino from the block. */
mi = ((struct minfs_inode *) bh->b_data) + ino;

/* TODO 4/6: fill VFS inode */
inode->i_mode = mi->mode;
i_uid_write(inode, mi->uid);
i_gid_write(inode, mi->gid);
inode->i_size = mi->size;
inode->i_blocks = 0;
inode->i_mtime = inode->i_atime = inode->i_ctime = current_time(inode);

/* TODO 4/2: Fill dir inode operations. */
inode->i_op = &simple_dir_inode_operations;
inode->i_fop = &simple_dir_operations;

/* TODO 4/1: Directory inodes start off with i_nlink == 2.
 * (use inc_link) */
inc_nlink(inode);
/* TODO 4/1: uncomment after the minfs_inode is initialized */
mii->data_block = mi->data_block;

/* Free resources. */
/* TODO 4/1: uncomment after the buffer_head is initialized */
brelse(bh);
//brelse(bh);
unlock_new_inode(inode);

static const struct super_operations minfs_ops = {
	.statfs		= simple_statfs,
	.put_super	= minfs_put_super,
	/* TODO 4/2: add alloc and destroy inode functions */
	.alloc_inode	= minfs_alloc_inode,
	.destroy_inode	= minfs_destroy_inode,
	/* TODO 7/1:	= set write_inode function. */
	.write_inode	= minfs_write_inode,
};
```

#### 5.minfs挂载和卸载的测试

> 验证文件系统是否已挂载

![image-20250908154050862](https://leeway2zcblog-1373523181.cos.ap-guangzhou.myqcloud.com/img/image-20250908154050862.png)

> 挂载成功











