# ChCore Lab3：用户进程和异常管理





```c
struct process {
	struct slot_table slot_table; 		// 进程的资源列表
	struct list_head thread_list;		// 线程链表
};

struct slot_table {
	unsigned int slots_size;			// 资源数量
	struct object_slot **slots;			// 资源指针 数组
	/* 如果 full_slots_bmp 某个bit是 1, 那么 slots_bmpt 对应的sizeof(unsigned long)个bits都置位 */
	unsigned long *full_slots_bmp;		//
	unsigned long *slots_bmp;			// 
};
```





先来看看系统的执行过程：

```shell
start
main
\-- uart_init
\-- mm_init
\-- exception_init
	\-- exception_init_per_cpu
\-- process_create_root
	\-- ramdisk_read_file
	\-- process_create
    	\-- process_init
    	\-- alloc_slot_id
    	\-- kzalloc					# slot = kzalloc(sizeof(*slot)) 给process.slot_table
    	\-- init_list_head			# init_list_head(&slot->copies)
    	\-- obj_alloc				# vmspace = obj_alloc(TYPE_VMSPACE, sizeof(*vmspace))
    	\-- vmspace_init			# vmspace_init(vmspace)
    	\-- cap_alloc				# cap_alloc(process, vmspace, 0)
    \-- thread_create_main
    	\-- obj_get
    	\-- obj_get/obj_put			# obj_get(root_process, thread_cap, TYPE_THREAD)
\-- switch_context
\-- eret_to_thread					# eret_to_thread(switch_context())
	\-- switch_thread_vmspace_to	# switch_thread_vmspace_to(target_thread)
```





 

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

> 在user/lib/syscall.c中完成syscall这一用户库函数，在其中使用SVC指令进入内核态并执行相应的系统调用。在此过程中，需要使用 到 GCC 内联汇编的相关内容，请参考“GCC 官方文档”。



```c
u64 syscall(u64 sys_no, u64 arg0, u64 arg1, u64 arg2, u64 arg3, u64 arg4,
	    u64 arg5, u64 arg6, u64 arg7, u64 arg8)
{

	u64 ret = 0;
	/*
	 * Lab3: Your code here
	 * Use inline assembly to store arguments into x0 to x7, store syscall number to x8,
	 * And finally use svc to execute the system call. After syscall returned, don't forget
	 * to move return value from x0 to the ret variable of this function
	 */
	asm volatile(	"mov x0, %[arg0]\n\t"
			"mov x1, %[arg1]\n\t"
			"mov x2, %[arg2]\n\t"
			"mov x3, %[arg3]\n\t"
			"mov x4, %[arg4]\n\t"
			"mov x5, %[arg5]\n\t"
			"mov x6, %[arg6]\n\t"
			"mov x7, %[arg7]\n\t"
			"mov x8, %[sys_no]\n\t"
			"svc #0\n\t"
			"mov %[ret], x0"
			:[ret] "=r" (ret)
			:[arg0] "r" (arg0), [arg1] "r" (arg1), [arg2] "r" (arg2), [arg3] "r" (arg3),
			 [arg4] "r" (arg4), [arg5] "r" (arg5), [arg6] "r" (arg6), [arg7] "r" (arg7),
			 [arg8] "r" (arg8), [sys_no] "r" (sys_no)
			:"x0", "x1", "x2", "x3", "x4", "x5", "x6", "x7", "x8");

	return ret;
}
```







## 练习6

> 在 ChCore 中完成以下系统调用。
>
> * sys_putc：printf所 使 用 的 基 本 系 统 调 用， 需 要 使 用uart_send标准输出一个字符
>
> * sys_exit：具有退出当前用户线程的功能，需要将对应编号的系 统调用分派到sys_exit这一函数上。
> * sys_create_pmo：用于测试缺页异常，实现在vm_syscall.c。
> * sys_map_pmo：用于测试缺页异常，实现在vm_syscall.c。
> * sys_handle_brk：用户线程将使用sys_handle_brk创建或扩 展用户堆。通过这一系统调用，当前进程的堆将被扩大至虚拟地 址 addr。更多详细信息请参见/kernel/mm/vm_syscall.c的注 释。



用户态的编程接口：

```c
void usys_putc(char ch)
{
	syscall(SYS_putc, ch, 0, 0, 0, 0, 0, 0, 0, 0);
}

void usys_exit(int ret)
{
	syscall(SYS_exit, ret, 0, 0, 0, 0, 0, 0, 0, 0);
}

int usys_create_pmo(u64 size, u64 type)
{
	return syscall(SYS_create_pmo, size, type, 0, 0, 0, 0, 0, 0, 0);
}

int usys_map_pmo(u64 process_cap, u64 pmo_cap, u64 addr, u64 rights)
{
	return syscall(SYS_exit, process_cap, pmo_cap, addr, rights, 0, 0, 0, 0, 0);
}

u64 usys_handle_brk(u64 addr)
{
	return syscall(SYS_handle_brk, addr, 0, 0, 0, 0, 0, 0, 0, 0);
}
```



先复习一下`vmspace`结构：

```c
struct vmregion {
	struct list_head node;	// vmr_list
	vaddr_t start;
	size_t size;
	vmr_prop_t perm;
	struct pmobject *pmo;
};

struct vmspace {
	/* list of vmregion */
	struct list_head vmr_list;
	/* root page table */
	vaddr_t *pgtbl;

	struct vmregion *heap_vmr;
	vaddr_t user_current_heap;
};
```

