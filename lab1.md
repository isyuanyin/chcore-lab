# ChCore Lab1：机器启动

本文为上海交大 ipads 研究所陈海波老师等人所著的《现代操作系统：原理与实现》的课程实验（LAB）的学习笔记的第一篇。

书籍官网：[现代操作系统：原理与实现（银杏书）](https://ipads.se.sjtu.edu.cn/mospi/)

课程官网（含视频和PPT）：[OS: Operating Systems（交大课程网站）](https://ipads.se.sjtu.edu.cn/courses/os/) / [好大学在线课程](https://www.cnmooc.org/portal/course/5610/14956.mooc)

本文代码仓库：[我的Github仓库(内含实验说明、代码和笔记)](https://github.com/isyuanyin/chcore-lab)



## 前置知识

了解C语言，计算机组成原理（或者学过汇编语言）就可以学习这门课。



## 环境配置



这里使用Windows系统的WSL2 Ubuntu系统，现在WSL2支持内置GUI，也就是WSLg，所以如果想用WSL的同学可以先更新WSL2到最新版本，详情请看[WSLg的Github仓库](https://github.com/microsoft/wslg)和Microsoft的官方说明。如果使用的是Ubuntu的桌面系统，就不需要考虑上面的。

另外，由于WSL2使用Docker比较麻烦，这里就不使用课程中给的Docker镜像和脚本。针对lab1而言，更改详情请看我的[lab1更改文件](https://github.com/isyuanyin/chcore-lab/blob/main/patchs/lab1.patch)。下面是详细说明配置过程。

### 安装工具

首先，本课程实验中是针对ARM64（也称作AArch64）架构实现的OS内核，而一般大家用的机器（也就是我们的笔记本或台式电脑）用的是Intel或AMD的x86_64（或者说AMD64）架构的芯片。不同架构的指令集不一样，机器执行的二进制格式不一样，所以需要有一个模拟器，跑在x86_64架构上，运行ARM64的二进制文件，这就是QEMU。

不过安装QEMU之前，先安装必要的开发工具：

```shell
sudo apt update
sudo apt install build-essential cmake
```

然后安装AArch64的QEMU

```
sudo apt install qemu qemu-kvm qemu-system-arm
```

里面也会安装arm32版本的QEMU，执行如下验证是否安装成功：

```
qemu-system-aarch64 -machine virt
```

成功的话，会出现如下界面：

![image-20210720232805577](https://raw.githubusercontent.com/isyuanyin/picgo/main/images/image-20210720232805577.png)

最后，我们的机器上的Ubuntu系统默认安装的GCC编译套件是针对x86_64架构的，如果在x86_64架构上编译ARM64架构的C语言代码，需要用到交叉编译的方式。安装AArch64的交叉编译套件：

```
sudo apt install gcc-aarch64-linux-gnu
```

通过`ls /usr/bin/aarch64-linux-gnu-*`可以看到安装了如下编译套件：

```
# 编译套件
aarch64-linux-gnu-gcc  # 将C文件编译成汇编文件
aarch64-linux-gnu-as   # 汇编器
aarch64-linux-gnu-ld   # 链接器
aarch64-linux-gnu-ar   # 将多个.o文件打包成.a文件
aarch64-linux-gnu-ranlib  # 打包时的符号定位，ar包含了这个功能

# 查看二进制文件内容的工具
aarch64-linux-gnu-objdump
aarch64-linux-gnu-dwp
aarch64-linux-gnu-readelf
aarch64-linux-gnu-nm
aarch64-linux-gnu-size

aarch64-linux-gnu-strings
aarch64-linux-gnu-elfedit
aarch64-linux-gnu-strip
```



### 修改脚本

这里因为不使用Docker容器，直接使用本地的交叉编译工具。先查看Makefile的内容，里面有下面两段：

```makefile
build: FORCE
	./scripts/docker_build.sh

docker: FORCE	
	./scripts/run_docker.sh
```

run_docker.sh和docker_build.sh只是简单地启动Docker容器，删除build目录，然后执行build.sh脚本：

```
# docker_build.sh
rm -rf ./build

docker run -it --rm -u $(id -u ${USER}):$(id -g ${USER}) -v $(pwd):/chos -w /chos ipads/chcore_builder:v1.0 ./scripts/build.sh "$@"

# run_docker.sh
docker run -it --rm -u $(id -u ${USER}):$(id -g ${USER}) -v $(pwd):/chos -w /chos ipads/chcore_builder:v1.0
```

将这两个文件删除，然后删掉Makefile文件里面的两段，将 `rm -rf ./build` 放在build.sh文件里面。

### 尝试运行

执行

```make qemu
make qemu
```



## 练习1

<table><tr><td bgcolor=#D1EBFD> 浏览《ARM 指令集参考指南》的 A1、A3 和 D 部分，以熟悉 ARM ISA。 请做好阅读笔记，如果之前学习 x86-64 的汇编，请写下与 x86-64 相比 的一些差异。</td></tr></table>

此处省略



## 练习2

<table><tr><td bgcolor=#D1EBFD> 启动带调试的 QEMU，使用 GDB 的where命令来跟踪入口（第一个函 数）及 bootloader 的地址。</td></tr></table>





### 练习3

<table><tr><td bgcolor=#D1EBFD> 结合readelf -S build/kernel.img读取符号表与练习 2 中的 GDB 调试信息，请找出请找出build/kernel.image入口定义在哪 个文件中。继续借助单步调试追踪程序的执行过程，思考一个问题：目 前本实验中支持的内核是单核版本的内核，然而在 Raspi3 上电后，所 有处理器会同时启动。结合boot/start.S中的启动代码，并说明挂起 其他处理器的控制流。 </td></tr></table>









## 练习4

<table><tr><td bgcolor=#D1EBFD>  </td></tr></table>







## 练习5

<table><tr><td bgcolor=#D1EBFD>  </td></tr></table>







## 练习6

<table><tr><td bgcolor=#D1EBFD>  </td></tr></table>

