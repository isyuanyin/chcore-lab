# ChCore lab1：机器启动

注意：关于本文有任何疑问请联系我，通过[点击此处](https://github.com/isyuanyin/chcore-lab/blob/main/README.md)



本文为上海交大 ipads 研究所陈海波老师等人所著的《现代操作系统：原理与实现》课程实验的学习笔记的第一篇。

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

执行如下：

```make qemu
make qemu
```



## 练习1

<table><tr><td bgcolor=#D1EBFD> 浏览《ARM 指令集参考指南》的 A1、A3 和 D 部分，以熟悉 ARM ISA。 请做好阅读笔记，如果之前学习 x86-64 的汇编，请写下与 x86-64 相比 的一些差异。</td></tr></table>

文档的官方网址可能发生一点变更，懒人可以直接从[这里](https://github.com/isyuanyin/chcore-lab/tree/main/lab-info)下载。文档描述了AArch32和AArch64的基本指令集。分为A、B、C和D四个部分，其中A1是概述；A3和D部分是AArch64相关的；B部分是向量指令和浮点运算单元；A2和C是AArch32相关的。



## 练习2

<table><tr><td bgcolor=#D1EBFD> 启动带调试的 QEMU，使用 GDB 的where命令来跟踪入口（第一个函 数）及 bootloader 的地址。</td></tr></table>

打开两个中断界面，在一个执行 `make qemu-gdb` ，另一个 `make gdb`。则执行kernel.img，然后自动在 `0x0000000000080000` 处卡住，这是第一条指令的位置，也是 bootloader 开始的位置。这里执行 where 一下。可以看出

```
0x0000000000080000 in ?? ()
(gdb) where
#0  0x0000000000080000 in _start ()
```

所在函数是 `_start` ，而这个函数定义在 boot/start.S中（Tips：_start也是linux中的C语言程序main函数编译后的入口地址）。



## 练习3

<table><tr><td bgcolor=#D1EBFD> 结合readelf -S build/kernel.img读取符号表与练习 2 中的 GDB 调试信息，请找出请找出build/kernel.image入口定义在哪 个文件中。继续借助单步调试追踪程序的执行过程，思考一个问题：目 前本实验中支持的内核是单核版本的内核，然而在 Raspi3 上电后，所 有处理器会同时启动。结合boot/start.S中的启动代码，并说明挂起 其他处理器的控制流。 </td></tr></table>

执行下面命令查看elf文件头部信息：

```
$ readelf -h kernel.img
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0x80000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          138256 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         3
  Size of section headers:           64 (bytes)
  Number of section headers:         10
  Section header string table index: 9
```

执行下面命令查看段（Segment）信息：

```
$ readelf -S kernel.img
There are 10 section headers, starting at offset 0x21c10:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] init              PROGBITS         0000000000080000  00010000
       000000000000b628  0000000000000008 WAX       0     0     4096
  [ 2] .text             PROGBITS         ffffff000008c000  0001c000
       0000000000001ec8  0000000000000000  AX       0     0     8
  [ 3] .rodata           PROGBITS         ffffff0000090000  00020000
       000000000000026d  0000000000000000   A       0     0     8
  [ 4] .eh_frame         PROGBITS         ffffff0000090270  00020270
       000000000000051c  0000000000000000   A       0     0     8
  [ 5] .bss              NOBITS           ffffff0000090790  0002078c
       00000000000081d0  0000000000000000  WA       0     0     16
  [ 6] .comment          PROGBITS         0000000000000000  0002078c
       000000000000002a  0000000000000001  MS       0     0     1
  [ 7] .symtab           SYMTAB           0000000000000000  000207b8
       0000000000000ed0  0000000000000018           8    87     8
  [ 8] .strtab           STRTAB           0000000000000000  00021688
       0000000000000540  0000000000000000           0     0     1
  [ 9] .shstrtab         STRTAB           0000000000000000  00021bc8
       0000000000000046  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

目前我们只需要知道 .init 段放的是 bootloader 的代码，.text 段放的是 chcore 内核的代码。

练习 2 中我们知道了第一条执行的指令地址为 `0x0000000000080000`，对比上面的结果发现正好是 Entry point address 和 init 段的地址。查阅资料可知，ELF 文件加载后入口指令的位置由 Entry point address 确定，我们再来研究下这个 Entry point address 是在哪里定义的。



在 chcore 项目里全局搜索 `80000`，定位到 image.h 中，这里面定义了代码段偏移 TEXT OFFSET 和内核虚拟地址 KERNEL VADDR。

```h
// boot/image.h
#pragma once

#define SZ_16K			0x4000
#define SZ_64K                  0x10000

#define KERNEL_VADDR		0xffffff0000000000
#define TEXT_OFFSET		0x80000
```

其中 TEXT_OFFSET 就是我们要找到的 `80000`，再看看谁引用了 TEXT_OFFSET。全局搜索定位到 scripts/linker-aarch64.lds.in 中。往下讲前先提一下 CMakeLists.txt 中定义了一个变量 `init_object`，该变量表示 bootloader 对应的所有目标文件的集合，即编译好的 bootloader 的机器码。

```cmake
# 把 bootloader 所有目标文件的集合打包为 init_object 这个变量
set(init_object
        "${BINARY_KERNEL_IMG_PATH}/${BOOTLOADER_PATH}/start.S.o
        ${BINARY_KERNEL_IMG_PATH}/${BOOTLOADER_PATH}/mmu.c.o
        ${BINARY_KERNEL_IMG_PATH}/${BOOTLOADER_PATH}/tools.S.o
        ${BINARY_KERNEL_IMG_PATH}/${BOOTLOADER_PATH}/init_c.c.o
        ${BINARY_KERNEL_IMG_PATH}/${BOOTLOADER_PATH}/uart.c.o"
    )
```

再看 scripts/linker-aarch64.lds.in，lds 文件为 gcc 的链接器脚本文件，语法上只要明白 `.` 表示“当前指针”的位置，`.段名` 表示某个段的位置。

```
// scripts/linker-aarch64.lds.in

#include "../boot/image.h"

SECTIONS
{
    . = TEXT_OFFSET;  // 当前指针赋值为 TEXT_OFFSET，即 0x80000
    img_start = .;    // 镜像开始地址（ELF 文件入口地址）设为当前指针，即 0x80000
    init : {		  
        ${init_object} // 指定 .init 段的内容为 init_object，即 bootloader 编译后的机器码
    }
	// 定义结束后当前指针将自动更新为 .init 段结尾地址

    // ......
```

对关键部分进行简要分析可知我们把 `img_start` 和 init 段开始地址都指定为了 TEXT OFFSET 的值，所以前面 Entry point address 和 init 段的地址都等于 `0x80000`。

而把 `_start` 函数和 `0x80000` 关联的语句则在 CMakeLists.txt 中

```cmake
# 编译时使用以下命令
# -T 指定链接器脚本
# -e 指定入口函数
set_property(
    TARGET kernel.img
    APPEND_STRING
    PROPERTY
        LINK_FLAGS
        "-T ${CMAKE_CURRENT_BINARY_DIR}/${link_script} -e _start"
)
```

通过上述一系列文件最终规定了 ELF 首条指令为 Entry point address 的值 `0x80000`，而该地址对应的函数则为 `_start`。

查看 boot/start.S代码

```assembly
.extern arm64_elX_to_el1                                                   
.extern boot_cpu_stack                                                     
.extern secondary_boot_flag                                                
.extern clear_bss_flag                                                     
.extern init_c                                                             
                                                                           
BEGIN_FUNC(_start)                                                         
    mrs x8, mpidr_el1     // mpidr_el1记录了当前PE的cpuid                           
    and x8, x8, #0xFF     // 保留低8位                       
    cbz x8, primary       // 比较cpuid是否为0，为0的继续执行
    
  /* hang all secondary processors before we intorduce multi-processors */ 
secondary_hang:                                                            
    bl secondary_hang     // cpuid不为0的core进入循环
                                                                           
primary:                                                             
    /* Turn to el1 from other exception levels. */                         
    bl  arm64_elX_to_el1  // 切换到el1异常级别
                                                                           
    /* Prepare stack pointer and jump to C. */                             
    adr     x0, boot_cpu_stack                               
    add     x0, x0, #0x1000         // 这里表示                  
    mov     sp, x0                  // 设置栈位置
                                                                           
    bl  init_c                      // 进入C语言代码中的init_c函数                      
    /* Should never be here */                                             
    b   .                           // 在这里循环
END_FUNC(_start)                                                           
```

可知 chcore 挂起其他处理器的方法是通过 `mpidr_el1` 寄存器的值来判断当前 PE 的 cpuid，若为 0 则为首个 PE，正常执行后续代码；若不为 0，则非首个 PE，跳到一个死循环函数中来进行挂起。



```c
 void init_c(void)                                                
 {                                                                
         /* Clear the bss area for the kernel image */            
         clear_bss();                                             
                                                                  
         /* Initialize UART before enabling MMU. */               
         early_uart_init();                                       
         uart_send_string("boot: init_c\r\n");                    
                                                                  
         /* Initialize Boot Page Table. */                        
         uart_send_string("[BOOT] Install boot page table\r\n");  
         init_boot_pt();                                          
                                                                  
         /* Enable MMU. */                                        
         el1_mmu_activate();                                      
         uart_send_string("[BOOT] Enable el1 MMU\r\n");           
                                                                  
         /* Call Kernel Main. */                                  
         uart_send_string("[BOOT] Jump to kernel main\r\n");      
         start_kernel(secondary_boot_flag);                       
                                                                  
         /* Never reach here */                                   
 }                                                                
```



唯一可执行的 PE 在后续代码中完成切换到 EL1、初始化 UART、页表、MMU 的过程，最后通过 `start_kernel` 将控制权交给内核代码。



## 练习4

<table><tr><td bgcolor=#D1EBFD> 
查看build/kernel.img的objdump信息。比较每一个段中的VMA和 LMA 是否相同，为什么？在 VMA 和 LMA 不同的情况下，内核是如何将该段的地址从 LMA 变为 VMA？提示：从每一个段的加载和运行情况进行分析   
 </td></tr></table>



```
$ objdump -h build/kernel.img

build/kernel.img:     file format elf64-little

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 init          0000b628  0000000000080000  0000000000080000  00010000  2**12
                  CONTENTS, ALLOC, LOAD, CODE
  1 .text         00001ec8  ffffff000008c000  000000000008c000  0001c000  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .rodata       0000026d  ffffff0000090000  0000000000090000  00020000  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .eh_frame     0000051c  ffffff0000090270  0000000000090270  00020270  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .bss          000081d0  ffffff0000090790  0000000000090790  0002078c  2**4
                  ALLOC
  5 .comment      0000002a  0000000000000000  0000000000000000  0002078c  2**0
                  CONTENTS, READONLY
```



看一下链接脚本：

```
 #include "../boot/image.h"                         
                                                    
 SECTIONS                                           
 {                                                  
     . = TEXT_OFFSET;                               
     img_start = .;                                 
     init : {                                       
         ${init_object}                             
     }                                              
                                                    
     . = ALIGN(SZ_16K);                             
                                                    
     init_end = ABSOLUTE(.);                        
                                                    
     .text KERNEL_VADDR + init_end : AT(init_end) { 
         *(.text*)                                  
     }                                              
                                                    
     . = ALIGN(SZ_64K);                             
     .data : {                                      
         *(.data*)                                  
     }                                              
     . = ALIGN(SZ_64K);                             
                                                    
     .rodata : {                                    
         *(.rodata*)                                
     }                                              
     _edata = . - KERNEL_VADDR;                     
                                                    
     _bss_start = . - KERNEL_VADDR;                 
     .bss : {                                       
         *(.bss*)                                   
     }                                              
     _bss_end = . - KERNEL_VADDR;                   
     . = ALIGN(SZ_64K);                             
     img_end = . - KERNEL_VADDR;                    
 }
```



Tips：

其实什么LMA和VMA在内核开发中都无所谓，只要在制作镜像的时候按顺序做的，加载进入就行了，因为加载这种东西本来是OS对应用程序的操作，现在我们写的就是OS，没人加载它，只负责将它放在内存的某个位置。



## 练习5

<table><tr><td bgcolor=#D1EBFD>
以不同的进制打印数字的功能（例如 8、10、16）尚未实现，请在kernel/common/printk.c中 填充printk_write_num以完善printk的功能。
</td></tr></table>

这里文档描述得不太清楚要输出的格式是什么样的，这里先看一下对这个函数的使用：

printk->simple_vsprintf->printk_write_num

其中printk跟C语言中的printf差不多，看一下simple_vsprintf，如下：

```
static int printk_write_num(char **out, long long i, int base, int sign, int width, int flags, int letbase);

static int simple_vsprintf(char **out, const char *format, va_list ap)
{
	
	for (; *format != 0; ++format) {                                          
        if (*format == '%') { 
            switch (*format) {                                        
                case ('d'):                                               
                u.i = va_arg(ap, int);                            
                pc +=                                             
                printk_write_num(out, u.i, 10, 1, width, flags, 'a');             
                break;                                            

                case ('u'):                                               
                u.u = va_arg(ap, unsigned int);                   
                pc +=                                             
                printk_write_num(out, u.u, 10, 0, width, flags, 'a');                 
                break;                                            
}
```

可以看出printk_write_num函数中的参数作用：

* out：输出方式
* i：数据类型
* base：多少进制
* sign：是否有符号
* width：宽度
* flags：补零还是空白
* letbase：16进制中字母的大小写，'A'是大写，'a'是小写



然后就可以写了：

```
```





## 练习6

<table><tr><td bgcolor=#D1EBFD>
    为了熟悉 AArch64 上的函数调用惯例，请在kernel/main.c中通过 GDB找到stack_test函数的地址，在该处设置一个断点，并检查在内 核启动后的每次调用情况。每个stack_test递归嵌套级别将多少个 64 位值压入堆栈，这些值是什么含义？提示：GDB 可以将寄存器打印为地 址及其 64 位值或数组，例如：
</td></tr></table>

```
(gdb) x/g $x29 
0xffffff000020f330 <kernel_stack+7984>: 0xffffff000020f350 
(gdb) x/10g $x29 
0xffffff000020f330 <kernel_stack+7984>: 0xffffff000020f350 0xffffff00000d009c
0xffffff000020f340 <kernel_stack+8000>: 0x0000000000000001 0x000000000000003e
0xffffff000020f350 <kernel_stack+8016>: 0xffffff000020f370 0xffffff00000d009c
0xffffff000020f360 <kernel_stack+8032>: 0x0000000000000002 0x000000000000003e
0xffffff000020f370 <kernel_stack+8048>: 0xffffff000020f390 0xffffff00000d009c
```



初始化的代码定义在 start.S 中，是把 `boot_cpu_stack + 0x1000` 的值赋值给 SP，这一指令同时也会让 FP 等于 SP。

```c
/* Prepare stack pointer and jump to C. */
adr 	x0, boot_cpu_stack
add 	x0, x0, #0x1000
mov 	sp, x0
```

在 boot/init_c.c 中可找到 boot_cpu_stack 的定义，是定义好的 4 个 4096 字节的全局数组，每个 CPU 用其中的一个做自己的 bootloader 用的函数栈。注意当前的函数栈只在 bootloader 里用，一会进了内核后会分配另一个内核使用的函数栈。

```c
#define INIT_STACK_SIZE 0x1000
char boot_cpu_stack[PLAT_CPU_NUMBER][INIT_STACK_SIZE] ALIGN(16);
```

`boot_cpu_stack + 0x1000` 为 `boot_cpu_stack[0]` 这个数组的最高的最高处。之所以取最高处的是因为函数栈是由高地址向低地址增长的。

```assembly
mov     x3, #0                    
msr     TPIDR_EL1, x3             
                                  
ldr     x2, =kernel_stack         
add     x2, x2, KERNEL_STACK_SIZE 
mov     sp, x2                    
bl      main                      
```



C语言部分的栈的位置如下：

kernel/main.c

```
ALIGN(STACK_ALIGNMENT)                               
char kernel_stack[PLAT_CPU_NUM][KERNEL_STACK_SIZE];  
```



在开始练习 7 前先提一下 AArch64 上的函数调用惯例：用 GCC 编译后（没有开优化的情况下）的函数代码开头一般都由一小段特殊的初始化代码。通常是把父函数的 FP 压入栈来保存旧的 FP，然后再将当前 SP 复制到 FP 中。另外，还会记录发生调用后在父函数里中断的地址（有人称之为返回地址）、保存父函数的寄存器、保存当前函数传入的参数等等。其中返回地址被记录在链接寄存器（Link Register, LR）x30 中。根据这些惯例我们就可以递归的确定一个函数的调用栈和传入的参数了。



## 练习7

<table><tr><td bgcolor=#D1EBFD>  </td></tr></table>







![](images/image-20210809020226244.png)











## 练习8

<table><tr><td bgcolor=#D1EBFD>  </td></tr></table>









## 练习9

<table><tr><td bgcolor=#D1EBFD>  </td></tr></table>





参考：https://www.cnblogs.com/kangyupl/p/chcore_lab1.html
