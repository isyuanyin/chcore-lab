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





## 问题3

<table>
	<tr><td bgcolor=#D1EBFD> 
        1. 请问在页表条目中填写的下一级页表的地址是物理地址还是虚拟地址? 
    </td></tr>
 	<tr><td bgcolor=#D1EBFD> 
        2. 在 ChCore 中检索当前页表条目的时候，使用的页表基地址是虚拟地 址还是物理地址？ 
	</td></tr>
</table>








## 问题4

<table>
    <tr><td bgcolor=#D1EBFD>  
        1. 如果我们有 4G 物理内存，管理内存需要多少空间开销? 这个开销是 如何降低的?
	</td></tr>
    <tr><td bgcolor=#D1EBFD>  
        2. 总结一下 x86-64 和 AArch64 地址翻译机制的区别，AArch64 MMU 架构设计的优点是什么?
	</td></tr>    
</table>






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


