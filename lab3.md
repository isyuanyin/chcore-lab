# ChCore Lab3：用户进程和异常管理



​       

## 练习1

> 在kernel/process/thread.c、kernel/sched/context.c和kernel/sched/sche.c 请补全以下函数，以实现对第一个用户进程和线程的创建和执行：
>
> * load_binary：解析 ELF 文件，并将其内容加载到新线程的用户 内存空间中。
> * init_thread_ctx：初始化线程的上下文，以便启动当前线程。
> * switch_context：切换到当前线程的上下文。







## 练习2

> 请简要描述process_create_root这一函数所的逻辑。注意：描述中 需包含thread_create_main函数的详细说明，建议绘制函数调用图 以描述相关逻辑。



```gr
```





## 练习3

> 完善异常处理， 需要阅读与修改kernel/exception/下 的exception_table.s、exception.S和exception.c。需要修改的内容包括：
>
> * 修改exception_table.S的内容，可借助该文件中的某些宏，填 写异常向量表。
> * 完成exception.c中的exception_init函数，使得 kernel 启 动后能够正确设置异常向量表。
> * 修改异常处理函数，使得当发生异常指令异常时，让内核使用 kinfo 打印在esr.h中定义的宏UNKNOWN的信息，并调用sys_exit函数 中止用户进程。











## 练习4

> 和其他异常不同，ChCore 中的系统调用是通过使用汇编代 码直接跳转到syscall_table中的相应条目来处理的。请阅 读kernel/exception/exception_table.S中的代码，并简要 描述 ChCore 是如何将系统调用从异常向量分派到系统调用表中对应条 目的。











## 练习5

> 在user/lib/syscall.c中完成syscall这一用户库函数，在其中使 用SVC指令进入内核态并执行相应的系统调用。在此过程中，需要使用 到 GCC 内联汇编的相关内容，请参考“GCC 官方文档”。









## 练习6

> 在 ChCore 中完成以下系统调用。
>
> * sys_putc：printf所 使 用 的 基 本 系 统 调 用， 需 要 使 用uart_send标准输出一个字符
>
> * sys_exit：具有退出当前用户线程的功能，需要将对应编号的系 统调用分派到sys_exit这一函数上。
> * sys_create_pmo：用于测试缺页异常，实现在vm_syscall.c。
> * sys_map_pmo：用于测试缺页异常，实现在vm_syscall.c。
> * sys_handle_brk：用户线程将使用sys_handle_brk创建或扩 展用户堆。通过这一系统调用，当前进程的堆将被扩大至虚拟地 址 addr。更多详细信息请参见/kernel/mm/vm_syscall.c的注 释。