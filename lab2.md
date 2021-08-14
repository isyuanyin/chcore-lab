# ChCore lab2：内存管理

关于本文有任何疑问请联系我，通过[点击此处](https://github.com/zhyuyi/chcore-lab/blob/main/README.md)



## 实验准备

防止使用了Docker，查看一切关于Docker的内容

```shell
yuanyin@LAPTOP-YUANYIN:~/chcore-lab$ grep -rn "docker" ./

./scripts/run_mm_test.sh:2:docker run -it --rm -u $(id -u ${USER}):$(id -g ${USER}) -v $(pwd):/chos -w /chos ipads/chcore_builder:v1.0 ./scripts/mm_test_script.sh
./scripts/gradelib.py:589:        os.system("./scripts/docker_build.sh %s > tmp.out" % binary)

Binary file ./scripts/__pycache__/gradelib.cpython-38.pyc matches
```

清理掉跟Docker有关的脚本中的相关内容：

```diff
--- a/scripts/gradelib.py
+++ b/scripts/gradelib.py
@@ -586,7 +586,7 @@ Failed to shutdown QEMU.  You might need to 'killall qemu' or
         return assert_lines_match_line(line_num, self.qemu.output, *args, **kwargs)

     def make_kernel(self, binary):
-        os.system("./scripts/docker_build.sh %s > tmp.out" % binary)
+        os.system("./scripts/build.sh %s > tmp.out" % binary)

     def file_match(self, file_name, r):
         f = open(file_name, 'r')
diff --git a/scripts/run_mm_test.sh b/scripts/run_mm_test.sh
index a236ab3..cf8d413 100755
--- a/scripts/run_mm_test.sh
+++ b/scripts/run_mm_test.sh
@@ -1,2 +1,2 @@
 #!/bin/bash
-docker run -it --rm -u $(id -u ${USER}):$(id -g ${USER}) -v $(pwd):/chos -w /chos ipads/chcore_builder:v1.0 ./scripts/mm_test_script.sh
```

重新编译

```
make clean
make
make grade
```

可以生成内核镜像，并且可以打分（虽然都是FAIL和0分），脚本可以执行。





## 问题1

<table><tr><td bgcolor=#D1EBFD> 请简单解释，在哪个文件或代码段中指定了 ChCore 物理内存布局。你 可以从两个方面回答这个问题: 编译阶段和运行时阶段。 </td></tr></table>



### 编译阶段

首先在编译脚本中scripts/linker-aarch64.lds.in中，指定了编译时刻的内核代码段的布局。



在`kernel/main.c:main`中调用了`kernel/mm:mm_init`函数，里面描述了页面和页面元数据的起始地址。

```c
 void mm_init(void)                                                             
 {                                                                              
    vaddr_t free_mem_start = 0;                                            
    struct page *page_meta_start = NULL;                                   
    u64 npages = 0;                                                        
	u64 start_vaddr = 0;                                                   

	free_mem_start = phys_to_virt(ROUND_UP((vaddr_t) (&img_end), PAGE_SIZE));          
    npages = NPAGES;                                                       
    start_vaddr = START_VADDR;                                             
    kdebug("[CHCORE] mm: free_mem_start is 0x%lx, free_mem_end is 0x%lx\n",
    	free_mem_start, phys_to_virt(PHYSICAL_MEM_END));                

    if ((free_mem_start + npages * sizeof(struct page)) > start_vaddr) {   
    	BUG("kernel panic: init_mm metadata is too large!\n");         
    }                                                                      

    page_meta_start = (struct page *)free_mem_start;                       
    kdebug("page_meta_start: 0x%lx, real_start_vadd: 0x%lx,"               
    	"npages: 0x%lx, meta_page_size: 0x%lx\n",                       
    	page_meta_start, start_vaddr, npages, sizeof(struct page));     

    /* buddy alloctor for managing physical memory */                      
    init_buddy(&global_mem, page_meta_start, start_vaddr, npages);         

    /* slab alloctor for allocating small memory regions */                
    init_slab();                                                           

    map_kernel_space(KBASE + (128UL << 21), 128UL << 21, 128UL << 21);     
    //check whether kernel space [KABSE + 256 : KBASE + 512] is mapped     
    kernel_space_check();                                                  
 }                                                                              
```



注意：这里的npages是一个固定的数值，对一个OS来说不太合理，不过这样就可以不用理有多少内存了，因为我们并不知道硬件信息。



### 运行时阶段







## 练习1

<table><tr><td bgcolor=#D1EBFD> 实现kernel/mm/buddy.c中的四个函数：buddy_get_pages()， split_page()，buddy_free_pages()，merge_page()。 请参考伙伴块索引等功能的辅助函数:get_buddy_chunk()。 </td></tr></table>

先看global_mem的定义：

```c
struct phys_mem_pool global_mem;
```

然后看看这个结构体描述了什么：

```c
/* Disjoint physical memory can be represented by several phys_mem_pool. */     
struct phys_mem_pool {                                            
    u64 pool_start_addr;        // 页面起始地址
    u64 pool_mem_size;          // 内存大小                                             
    u64 pool_phys_page_num;     // 物理页面个数                           
    struct page *page_metadata; // 页面的元数据
    struct free_list free_lists[BUDDY_MAX_ORDER]; // 空闲页面列表                        
};                                
```

接着看物理页面结构体：

```c
struct page {
    struct list_head node; // 链表结点
    int allocated;         // 表示是否已分配
    int order;             // 属于哪一阶
    void *slab;
};                                                                 
```

然后再看初始化过程，即kernel/mm/buddy.c:init_buddy函数中。然后分析过程，就不贴代码了。



## 问题2

<table><tr><td bgcolor=#D1EBFD> AArch64 采用了两个页表基地址寄存器，相较于 x86-64 架构中只有一个页表基地址寄存器，这样的好处是什么？请从性能与安全两个角度做 简要的回答。 </td></tr></table>



一般有分为内核态和用户态。







## 第二部分：虚拟内存映射

回顾前面的页表映射，在boot/main.c文件中第一次创建了页表和开启了MMU机制，开启的方式如下：

boot/tools.S

```assembly
adrp    x8, boot_ttbr0_l0   // TTBR0的一级页表基址地址                                
msr     ttbr0_el1, x8                                     
adrp    x8, boot_ttbr1_l0                                 
msr     ttbr1_el1, x8       // TTBR1的一级页表基址地址                        
```

也就是boot_ttbr0_l0和boot_ttbr0_l0两个数组存放了对应的页表，查看它们的初始化。下面是TTBR0所放页表的初始化方式。

先来看看TTBR0所指向的页表的初始化：

kernel/mmu.c:init_boot_pt

```c
	boot_ttbr0_l0[0] = ((u64) boot_ttbr0_l1) | IS_TABLE | IS_VALID; // 将二级页表的地址保存在一级页表数组元素中
	boot_ttbr0_l1[0] = ((u64) boot_ttbr0_l2) | IS_TABLE | IS_VALID; // 将三级页表的地址保存在二级页表数组元素中
	/* Usuable memory: PHYSMEM_START ~ PERIPHERAL_BASE */
	start_entry_idx = PHYSMEM_START / SIZE_2M;                      // 三级页表中的表示范围是2M
	end_entry_idx = PERIPHERAL_BASE / SIZE_2M;
	/* Map each 2M page */
	for (idx = start_entry_idx; idx < end_entry_idx; ++idx) {
		boot_ttbr0_l2[idx] = (PHYSMEM_START + idx * SIZE_2M)        // 这是按线性的方式赋值
		    | UXN	/* Unprivileged execute never */
		    | ACCESSED	/* Set access flag */
		    | INNER_SHARABLE	/* Sharebility */
		    | NORMAL_MEMORY	/* Normal memory */
		    | IS_VALID;
	}
	/* Peripheral memory: PERIPHERAL_BASE ~ PHYSMEM_END */
	/* Raspi3b/3b+ Peripherals: 0x3f 00 00 00 - 0x3f ff ff ff */
	start_entry_idx = end_entry_idx;
	end_entry_idx = PHYSMEM_END / SIZE_2M;
	/* Map each 2M page */
	for (idx = start_entry_idx; idx < end_entry_idx; ++idx) {
		boot_ttbr0_l2[idx] = (PHYSMEM_START + idx * SIZE_2M)
		    | UXN	/* Unprivileged execute never */
		    | ACCESSED	/* Set access flag */
		    | DEVICE_MEMORY	/* Device memory */
		    | IS_VALID;
	}
```

再来看看TTBR1所指向的页表的初始化：

```c
	kva = KERNEL_VADDR;
	boot_ttbr1_l0[GET_L0_INDEX(kva)] = ((u64) boot_ttbr1_l1) | IS_TABLE | IS_VALID;
	boot_ttbr1_l1[GET_L1_INDEX(kva)] = ((u64) boot_ttbr1_l2) | IS_TABLE | IS_VALID;
 
	start_entry_idx = GET_L2_INDEX(kva);
	/* Note: assert(start_entry_idx == 0) */
	end_entry_idx = start_entry_idx + PHYSMEM_BOOT_END / SIZE_2M;
	/* Note: assert(end_entry_idx < PTP_ENTIRES) */
	/*
	 * Map each 2M page
	 * Usuable memory: PHYSMEM_START ~ PERIPHERAL_BASE
	 */
	for (idx = start_entry_idx; idx < end_entry_idx; ++idx) {
		boot_ttbr1_l2[idx] = (PHYSMEM_START + idx * SIZE_2M)
		    | UXN	/* Unprivileged execute never */
		    | ACCESSED	/* Set access flag */
		    | INNER_SHARABLE	/* Sharebility */
		    | NORMAL_MEMORY	/* Normal memory */
		    | IS_VALID;
	}

	/* Peripheral memory: PERIPHERAL_BASE ~ PHYSMEM_END */
	start_entry_idx = start_entry_idx + PERIPHERAL_BASE / SIZE_2M;
	end_entry_idx = PHYSMEM_END / SIZE_2M;

	/* Map each 2M page */
	for (idx = start_entry_idx; idx < end_entry_idx; ++idx) {
		boot_ttbr1_l2[idx] = (PHYSMEM_START + idx * SIZE_2M)
		    | UXN	/* Unprivileged execute never */
		    | ACCESSED	/* Set access flag */
		    | DEVICE_MEMORY	/* Device memory */
		    | IS_VALID;
	}

	/*
	 * Local peripherals, e.g., ARM timer, IRQs, and mailboxes
	 *
	 * 0x4000_0000 .. 0xFFFF_FFFF
	 * 1G is enough. Map 1G page here.
	 */
	kva = KERNEL_VADDR + PHYSMEM_END;
	boot_ttbr1_l1[GET_L1_INDEX(kva)] = PHYSMEM_END | UXN	/* Unprivileged execute never */
	    | ACCESSED		/* Set access flag */
	    | DEVICE_MEMORY	/* Device memory */
	    | IS_VALID;
```



再来看看页表描述符的数据结构：

```c
typedef union {
	struct {
		u64 is_valid:1, is_table:1, ignored1:10, next_table_addr:36, reserved:4, ignored2:7, PXNTable:1,	// Privileged Execute-never for next level
		 XNTable:1,	// Execute-never for next level
		 APTable:2,	// Access permissions for next level
		 NSTable:1;
	} table;
	struct {
		u64 is_valid:1, is_table:1, attr_index:3,	// Memory attributes index
		 NS:1,		// Non-secure
		 AP:2,		// Data access permissions
		 SH:2,		// Shareability
		 AF:1,		// Accesss flag
		 nG:1,		// Not global bit
		 reserved1:4, nT:1, reserved2:13, pfn:18, reserved3:2, GP:1, reserved4:1, DBM:1,	// Dirty bit modifier
		 Contiguous:1, PXN:1,	// Privileged execute-never
		 UXN:1,		// Execute never
		 soft_reserved:4, PBHA:4;	// Page based hardware attributes
	} l1_block;
	struct {
		u64 is_valid:1, is_table:1, attr_index:3,	// Memory attributes index
		 NS:1,		// Non-secure
		 AP:2,		// Data access permissions
		 SH:2,		// Shareability
		 AF:1,		// Accesss flag
		 nG:1,		// Not global bit
		 reserved1:4, nT:1, reserved2:4, pfn:27, reserved3:2, GP:1, reserved4:1, DBM:1,	// Dirty bit modifier
		 Contiguous:1, PXN:1,	// Privileged execute-never
		 UXN:1,		// Execute never
		 soft_reserved:4, PBHA:4;	// Page based hardware attributes
	} l2_block;
	struct {
		u64 is_valid:1, is_page:1, attr_index:3,	// Memory attributes index
		 NS:1,		// Non-secure
		 AP:2,		// Data access permissions
		 SH:2,		// Shareability
		 AF:1,		// Accesss flag
		 nG:1,		// Not global bit
		 pfn:36, reserved:3, DBM:1,	// Dirty bit modifier
		 Contiguous:1, PXN:1,	// Privileged execute-never
		 UXN:1,		// Execute never
		 soft_reserved:4, PBHA:4,	// Page based hardware attributes
		 ignored:1;
	} l3_page;
	u64 pte;
} pte_t;

#define PTE_DESCRIPTOR_INVALID                    (0)

/* page_table_page type */
typedef struct {
	pte_t ent[PTP_ENTRIES];
} ptp_t;
```





## 问题3

<table>
    <tr><td bgcolor=#D1EBFD>  
        1. 请问在页表条目中填写的下一级页表的地址是物理地址还是虚拟地址? 
	</td></tr>
    <tr><td bgcolor=#D1EBFD>  
        2. 在 ChCore 中检索当前页表条目的时候，使用的页表基地址是虚拟地址还是物理地址？ 
	</td></tr>    
</table>

1. **请问在页表条目中填写的下一级页表的地址是物理地址还是虚拟地址?**

这个问题可以从反面思考，假设某一级的页表（一致地）放的是虚拟地址，那么在这一级页表查找下一级页表地址的时候，需要继续回头查找虚拟地址对应的物理地址，那么将陷入死循环中。

这里说一致是因为它可能可以一部分放虚拟地址，一部分放物理地址；但在ARM64中，放的是物理地址。



2. **在 ChCore 中检索当前页表条目的时候，使用的页表基地址是虚拟地址还是物理地址？**

虚拟地址。因为在操作数据的时候，MMU自动将对应的地址做一次转化，所以只能使用虚拟地址。由于在填充上一级页表项的时候，我们又需要知道页表的物理地址。

那么有问题了，如何通过物理地址知道虚拟地址，或者如何通过虚拟地址获取物理地址呢？






## 问题4

<table>
    <tr><td bgcolor=#D1EBFD>  
        1. 如果我们有 4G 物理内存，管理内存需要多少空间开销? 这个开销是 如何降低的?
	</td></tr>
    <tr><td bgcolor=#D1EBFD>  
        2. 总结一下 x86-64 和 AArch64 地址翻译机制的区别，AArch64 MMU 架构设计的优点是什么?
	</td></tr>    
</table>
1. 如果我们有 4G 物理内存，管理内存需要多少空间开销? 这个开销是 如何降低的?

管理内存需要多少空间开销这个问题有些复杂。管理内存的话需要有一套可以映射到所有内存空间的页表，每页4KB，那么需要1M个页描述符。

但是现代的计算机架构采用虚拟地址是因为两个需求：不同进程拥有隔离的地址空间以及每个进程拥有非常大的地址空间（1 << 64 大小）。由于每个虚拟地址都有可能需要映射到真实的物理地址上，如果每个虚拟地址都有映射的话，这需要非常大的空间。多级页表就能解决这个问题，但是多级页表也有自身的问题，就是需要访问多次。还有一种解决的方案是采用反向页表。

2.  总结一下 x86-64 和 AArch64 地址翻译机制的区别，AArch64 MMU 架构设计的优点是什么?

x86_64采用的是段页式管理机制，在地址翻译的过程中，除了加上页表外，还加上了段描述符保存的基址。AArch64 MMU省去了没有必要的段式管理，减少了一次计算。

AArch64 MMU 采用双页表基址寄存器，也减少了页表基址切换时带来的cache开销。



## 练习2

<table><tr><td bgcolor=#D1EBFD> 在文件kernel/mm/page_table中，实现map_range_in_pgtbl()， unmap_range_in_pgtbl()和query_in_pgtbl() 。可以调用辅助 函数：set_pte_flags(),get_next_ptp(),flush_tlb()。 </td></tr></table>





## 问题5

<table><tr><td bgcolor=#D1EBFD> 在 AArch64 MMU 架构中，使用了两个 TTBR 寄存器，ChCore 使用一 个 TTBR 寄存器映射内核地址空间，另一个寄存器映射用户态的地址空 间，那么是否还需要通过设置页表位的属性来隔离内核态和用户态的地 址空间? </td></tr></table>





## 问题6

<table>
    <tr><td bgcolor=#D1EBFD> 
        1. ChCore 为什么要使用块条目组织内核内存? 哪些虚拟地址空间在 Boot 阶段必须映射，哪些虚拟地址空间可以在内核启动后延迟? 
    </td></tr>    
    <tr><td bgcolor=#D1EBFD>         
        2. 为什么用户程序不能读写内核内存? 保护内核内存的具体机制是什么? 
    </td></tr>
</table>









## 练习3

<table><tr><td bgcolor=#D1EBFD> 完善map_kernel_space()函数，实现对内核空间的映射，并且可以 通过kernel_space_check()的检查。 </td></tr></table>









## 挑战！

<table>
    <tr><td bgcolor=#D1EBFD> 
        1. 以页粒度（4KB）映射内核空间。默认情况下（即实验 2 第二部分 中），只有用户进程可以以页粒度中映射虚拟地址，因此需要修改函 数set_pte_flags()，来支持页粒度内核空间映射。
    </td></tr>    
    <tr><td bgcolor=#D1EBFD>         
        2. 支持以块粒度（2MB）来管理用户态低空空间，修改page_table.c中 的map_range_in_pgtbl()以区分需要映射的页的大小。
    </td></tr>
</table>


