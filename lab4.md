# ChCore Lab4：多核处理



## 练习1

> 阅读boot/start.S中的汇编代码start。比较实验三和本实验之间start部分代码的差异（可使用git diff lab3命令）。说 明_start如何选定主 CPU，并阻塞其他副 CPU 的执行。

直接看代码注释：

```assembly
#include <common/asm.h>

.extern arm64_elX_to_el1
.extern boot_cpu_stack
.extern secondary_boot_flag
.extern secondary_init_c
.extern clear_bss_flag
.extern init_c

BEGIN_FUNC(_start)
	mrs	x8, mpidr_el1					/* 每个CPU都获取自己的cpuid放在x8寄存器中 */
	and	x8, x8,	#0xFF
	cbz	x8, primary						/* 主CPU跳转到primary，副CPU继续执行 */

	/* Wait for bss clear */
wait_for_bss_clear:
	adr	x0, clear_bss_flag				/* 判断主CPU是否执行了clear_bss函数的标志位 */
	ldr	x1, [x0]
	cmp     x1, #0
	bne	wait_for_bss_clear

	/* Turn to el1 from other exception levels. */
	bl 	arm64_elX_to_el1				/* 设置异常级别为EL1 */

	/* Prepare stack pointer and jump to C. */
	mov	x1, #0x1000
	mul	x1, x8, x1
	adr 	x0, boot_cpu_stack			/* 通过偏移获取对应CPU的栈 */
	add	x0, x0, x1
	add	x0, x0, #0x1000
        mov	sp, x0						/* 设置对应CPU的栈顶 */

wait_until_smp_enabled:					/* 副CPU在这里循环，直到secondary_boot_flag被修改 */
	/* CPU ID should be stored in x8 from the first line */
	mov	x1, #8							/* cpuid保存在x8中 */
	mul	x2, x8, x1
	ldr	x1, =secondary_boot_flag		/* x1获取secondary_boot_flag的地址 */
	add	x1, x1, x2						/* 加上对应CPU 的secondary_boot_flag值得偏移 */
	ldr	x3, [x1]						/* x3获取secondary_boot_flag的值 */
	cbz	x3, wait_until_smp_enabled		/* 直到主CPU修改secondary_boot_flag值，才相等并使能 */

	/* Set CPU id */
	mov	x0, x8					/* 此时x8保存着cpuid的值，作为secondary_init_c的第一个参数 */
	bl 	secondary_init_c

primary:

	/* Turn to el1 from other exception levels. */
	bl 	arm64_elX_to_el1

	/* Prepare stack pointer and jump to C. */
	adr 	x0, boot_cpu_stack
	add 	x0, x0, #0x1000
	mov 	sp, x0

	bl 	init_c

	/* Should never be here */
	b	.
END_FUNC(_start)

```



主CPU的执行路径如下：

```shell
_start						# boot/start.S
\-- init_c					# boot/init.c
	\-- clear_bss
	\-- early_uart_init
	\-- wakeup_other_cores  # 发送事件(Send Event)指令SEV唤醒CPU,因为可能某个CPU在等待事件(WFE, wait for event)
	\-- init_boot_pt		# 创建基本使用的页表 page table
	\-- el1_mmu_active		# 激活MMU
	\-- start_kernel		# kenerl/head.S 切换内核栈
		\-- main			# kernel/main.c
			\-- uart_init
			\-- mm_init
			\-- exception_init
			\-- kernel_lock_init
			\-- sched_init				# 初始化调度器
			\-- enable_smp_cores		# 使能副CPU
			\-- process_create_root
			\-- sched
			\-- eret_to_thread(switch_context)
	
```



副CPU的执行路径

```shell
_start
\-- secondary_init_c			# boot/init.c
	\-- el1_mmu_activate
	\-- secondary_start			# kernel/main.c
		\-- exception_init_per_cpu
		\-- sched
		\-- eret_to_thread(switch_context)
```





## 练习2

> 完 善 主 CPU 激 活 各 个 副 CPU 的 函 数： enable_smp_cores()和kernel/main.c中 的secondary_start()。同时，请注意测试代码会要求各个副 CPU 按序被依次激活。完成该练习后应能够通过smp测试，并获得测试 的前 5 分。

代码：

```c
void enable_smp_cores(void *addr)
{
	int i = 0;
	long *secondary_boot_flag;

	/* Set current cpu status */
	cpu_status[smp_get_cpu_id()] = cpu_run;
	secondary_boot_flag = (long *)phys_to_virt(addr);
	for (i = 0; i < PLAT_CPU_NUM; i++) {
		/* Lab4
		 * You should set one flag to enable the APs to continue in
		 * _start of `start.S`. Then, what's the flag?
		 * You only need to write one line of code.
		 */
		secondary_boot_flag[i] = 1;

		/* Lab4
		 * The BSP waits for the currently initializing AP finishing
		 * before activating the next one
		 */
		while (cpu_status[i] != cpu_run) { }
	}

	/* This information is printed when all CPUs finish their initialization */
	kinfo("All %d CPUs are active\n", PLAT_CPU_NUM);
}
```







## 练习3

> 熟悉启动副 CPU 的控制流程（同主 CPU 类似），并回答以下问题：初 始化时，主 CPU 同时激活所有副 CPU 而不是依次激活每个副 CPU 的 设计是否正确？换言之，并行启动副 CPU 是否会导致并发问题？ 提示：检查每个 CPU 核心是否共享相同的内核堆栈以及控制流中的每 个函数调用是否会导致数据竞争。









## 练习4

> 请熟悉排号锁的基本算法，并在kernel/common/lock.c中完成 unlock()和is_locked()的代码。至此，实验代码应通过mutex测试， 并获得对应的 5 分。 注意：本练习无需使用任何汇编代码（例如内存屏障等），且需要编写的 代码少于五行。





## 练习5

> 在kernel/common/lock.c中 实 现kernel_lock_init()、 lock_kernel()和unlock_kernel()。如上所述，通过在适当 的位置调用lock_kernel()和unlock_kernel()，使用大内核锁来 处理可能的并发问题。至此，实验代码应通过big lock测试，并获得 对应的 5 分。



## 练习6

> 为了保护寄存器上下文，在el0_syscall调用lock_kernel()时，在栈上保存了寄存器的值。然而，在exception_return中调 用unlock_kernel()时，却不需要将寄存器的值保存到栈中，试分析 其原因。

因为el0_syscall的时候，进行了用户态到内核态的切换，需要保存用户态的寄存器。exception_return时候，内核态的寄存器无所谓，不需要保存。







## 练习7

> 完善kernel/sched/policy_rr.c中的调度功能。完成本练习后应 能够运行cooperative测试并获得 10 分。现在，调度器应可以在一个 CPU 核心上工作。









## 练习8

> 如 果异常是从内核态捕获的，CPU核心不会在kernel/exception/irq.c的handle_irq中获得了大内核锁。但是，有一种特殊情况，即如果空闲线程（以内核态运行）中捕获 了错误，则 CPU 核心还应该获取大内核锁。否则，内核可能会被永远阻塞。请思考一下原因。















## 练习9

> 现在，尽管调度器尚未完成，但已经可以运行一些简单的用户态程 序。在syscall.c中实现系统调用sys_get_cpu_id()，它告诉用 户态程序正在运行的 CPU 核心的 ID。在sched.c中实现系统调 用sys_yield()，使用户态程序可以启动线程调度。完成本练习后应 能够运行/yield_single.bin并获得以下输出：
>
> ```
> ...
> Hello, I am thread 0 
> Hello, I am thread 1 
> Iteration 0, thread 0, cpu 0 
> Iteration 0, thread 1, cpu 0 
> Iteration 1, thread 0, cpu 0 
> Iteration 1, thread 1, cpu 0 ...
> ```
>
> 由于测试脚本会首先运行内核测试，然后再运行用户程序测试。虽然目前 可以正确地运行用户态程序，但是还无法运行make grade获得yield single测试的 5 分。



## 练习10

> 在exception_init_per_cpu()中恢复被注释代码timer_init()， 这将在用户态下启用硬件定时器中断。内核的终端处理逻辑结束后会调 度线程。此时，yield_spin.bin应可以正常工作：主线程应能在一定 时间后重新获得对 CPU 核心的控制并正常终止。由于测试脚本的原因， 在完成该练习时还无法通过make grade来获得yield spin测试的 5 分。







## 练习11

> 在kernel/sched/policy_rr.c中修改调度器逻辑，以便它可以支 持预算机制。按照上述说明，实现rr_sched_handle_timer_irq()。 不要忘记在kernel/sched/sched.c的sys_yield()中重置“预算”， 确保sys_yield()在被调用后可以立即调度当前线程。完成本练习后 应能够preemptive测试并获得 5 分。



## 练习 16

> 在kernel/ipc/中实现了大多数 IPC 相关的代码，请根据注释完成其 余代码。完成本练习后应能够通过ipc data和ipc mem测试，并获得 15 分。 注意：除使用正确代码逻辑替换所有LAB4_IPC_BLANK外，本练习有还 有少量额外代码需要编写。



## 练习17

> 熟 悉 IPC 的 工 作 流 程， 并 实 现 一 个 简 化 的 系 统 调 用sys_ipc_reg_call()。 它 与sys_ipc_call()相 似， 唯 一 的 区别是sys_ipc_reg_call()的第二个参数是 64 位值，而不是共享 内存的地址。该参数应作为ipc_dispatcher()的唯一参数直接传递 到服务器线程。完成本练习后应能够通过ipc reg测试，并获得 5 分。





## 额外练习

> 在完成所有先前的练习和问题后，创建一个名为 lab4-bonus 的新分支以 完成此部分练习。然后，为 ChCore 实现ipc_send()和ipc_recv()。 在新的分支中，可以根据需要修改任何代码。







