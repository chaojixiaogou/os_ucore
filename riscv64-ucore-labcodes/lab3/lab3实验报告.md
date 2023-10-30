#### **练习 1：理解基于 FIFO 的页面替换算法（思考题）** 

描述 FIFO 页面置换算法下，一个页面从被换入到被换出的过程中，会经过代码里哪些函数/宏的处理（或者说，需要调用哪些函数/宏），并用简单的一两句话描述每个函数在过程中做了什么？（为了方便同学们完成练习，所以实际上我们的项目代码和实验指导的还是略有不同，例如我们将 FIFO 页面置换算法头文件的大部分代码放在了 `kern/mm/swap_fifo.c `文件中，这点请同学们注意） 

• 至少正确指出 10 个不同的函数分别做了什么？如果少于 10 个将酌情给分。我们认为只要函数原型不同，就算两个不同的函数。要求指出对执行过程有实际影响, 删去后会导致输出结果不同的函数（例如`assert`）而不是` cprintf` 这样的函数。如果你选择的函数不能完整地体现”从换入到换出“的过程，比如10 个函数都是页面换入的时候调用的，或者解释功能的时候只解释了这 10 个函数在页面换入时的功能，那么也会扣除一定的分数

 

1. `struct Page *result = alloc_page();`中`alloc_page()`函数分配了一个新的页面（`Page`）结构，用于存储从交换文件中加载回来的数据。

   

2. `assert() `函数是一个用于在程序中进行断言（assertion）检查的宏，通常用于调试和验证程序的正确性。它的主要作用是在运行时检查某个条件是否为真，如果条件为假，将导致程序终止并生成一个错误消息。

   

3. `pte_t *ptep = get_pte(mm->pgdir, addr, 0);`中`get_pte(mm->pgdir, addr, 0) `函数是用于通过 `mm `参数提供的页目录（`pgdir`）来查找给定虚拟地址 `addr` 在页表中的页表项指针 `ptep`。

    

4. `pde_t *pdep1 = &pgdir[PDX1(la)];`中`PDX1(la)`宏作用为计算出线性地址` la `对应的一级页目录项（PDE）的指针` pdep1`

    

5. `set_page_ref(page, 1) `函数作用为设置`page`页面的`ref`值为1。

    

6. `uintptr_t pa = page2pa(page);`中`page2pa(page)` 函数作用为获取该页面的物理地址 `pa`。

    

7. `memset(KADDR(pa), 0, PGSIZE)` 函数作用为将`page`页面清零。

    

8. `*pdep1 = pte_create(page2ppn(page), PTE_U | PTE_V);`中`pte_create(page2ppn(page), PTE_U | PTE_V)`宏作用为创建一个页表项，并将其写入一级页目录项 `*pdep1` 中，同时设置 `PTE_U `和` PTE_V `标志以表示用户可访问和页面存在。

    

9. `return &((pte_t *)KADDR(PDE_ADDR(*pdep0)))[PTX(la)];`中`PTX(la)`宏作用为计算出页内偏移，并返回指向相应页表项的内核虚拟地址。

    

10. `r = swapfs_read((*ptep), result)) != 0`中`swapfs_read((*ptep), result))` 函数作用是尝试从交换文件中读取数据到之前分配的页面` result` 中。`(*ptep)` 传递了一个交换文件中的位置或索引，而 `result `是目标页面。如果读取操作失败，它将返回一个非零值，否则为零。

     

11. `return ide_read_secs(SWAP_DEV_NO, swap_offset(entry) * PAGE_NSECT, page2kva(page), PAGE_NSECT);`中`ide_read_secs(SWAP_DEV_NO, swap_offset(entry) * PAGE_NSECT, page2kva(page), PAGE_NSECT) `函数作用是从IDE硬盘中读取指定扇区的数据，并将数据复制到指定的内存位置。

     

12. `int r = sm->swap_out_victim(mm, &page, in_tick);`中`swap_out_victim(mm, &page, in_tick) `函数作用为调用页面置换算法的接口，尝试找到一个可以被换出的页面，如果找到了则返回0。

     

13. `swapfs_write( (page->pra_vaddr/PGSIZE+1)<<8, page) != 0`中`swapfs_write( (page->pra_vaddr/PGSIZE+1)<<8, page)` 函数作用为将页面写入交换文件中，其中` (page->pra_vaddr/PGSIZE+1)<<8 `用于确定要写入的交换文件中的位置。

     

14. `sm->map_swappable(mm, v, page, 0);`中`map_swappable(mm, v, page, 0) `函数作用为将当前页面标记为可交换的状态。 

    

15. `free_page(page)` 函数作用为释放当前页面。 

    

16. `tlb_invalidate(mm->pgdir, v) `函数作用为刷新TLB。

 

#### 练习 2：深入理解不同分页模式的工作原理（思考题）

`get_pte()` 函数（位于` kern/mm/pmm.c`）用于在页表中查找或创建页表项，从而实现对指定线性地址对应的物理页的访问和映射操作。这在操作系统中的分页机制下，是实现虚拟内存与物理内存之间映射关系非常重要的内容。

• `get_pte()` 函数中有两段形式类似的代码，结合 sv32，sv39，sv48 的异同，解释这两段代码为什么如此相像。

• 目前 `get_pte() `函数将页表项的查找和页表项的分配合并在一个函数里，你认为这种写法好吗？有没有必要把两个功能拆开？

 

1. sv32，sv39，sv48分别有二级页表、三级页表和四级页表。两段相似的代码分别为访问一级页表和访问二级页表，而在sv39三级页表中，页表的层次结构之间是相似的，因此两段代码会存在相似之处。
2. 将两个功能合并在一个函数中能够减少代码的复杂性，并且效率更高，避免了多次查找相同虚拟地址的开销。但拆分功能可以使代码更模块化，每个函数都专注于一个具体的任务，提高了代码的可维护性、清晰性和可拓展性。

 

#### 练习 3：给未被映射的地址映射上物理页（需要编程）

补充完成 `do_pgfault（mm/vmm.c）`函数，给未被映射的地址映射上物理页。设置访问权限的时候需要参考页面所在 VMA 的权限，同时需要注意映射物理页时需要操作内存控制结构所指定的页表，而不是内核的页表。请在实验报告中简要说明你的设计实现过程。请回答如下问题：

• 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中组成部分对 ucore 实现页替换算法的潜在用处。 

• 如果 ucore 的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

• 数据结构 Page 的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

 

**设计过程：**

1. 使用swap_in函数加载磁盘页面内容到页面管理的内存中。
2. 使用page_insert函数设置页面物理地址到逻辑地址的映射。
3. 使用swap_map_swapable函数设置页面可交换。

```c++
if (swap_init_ok) {

      struct Page *page = NULL;

      // 你要编写的内容在这里，请基于上文说明以及下文的英文注释完成代码编写

      //(1）According to the mm AND addr, try

      //to load the content of right disk page

      //into the memory which page managed.

      //(2) According to the mm,

      //addr AND page, setup the

      //map of phy addr <--->

      //logical addr

      //(3) make the page swappable.

      swap_in(mm, addr, &page);

      page_insert(mm->pgdir, page, addr, perm);

      swap_map_swappable(mm, addr, page, 1);

      page->pra_vaddr = addr;

    } else {

      cprintf("no swap_init_ok but ptep is %x, failed\n", *ptep);

      goto failed;

    }
```

页目录项中的基地址是指向页表的基地址，它决定了虚拟地址空间的哪个部分在物理内存中的位置。在页替换算法中，可以通过修改基地址来切换不同的页表，以实现不同的页面替换策略。权限位用于控制页面的访问权限，例如读、写、执行等。在某些情况下，可以根据替换策略来更改这些权限位，以限制或允许页面的访问。

页表项中的物理页帧号是虚拟页面在物理内存中的实际位置。在页替换算法中，可以根据替换策略来选择要替换的页面，并将其物理页帧号更新为新的页面。有效位用于标记一个页面是否在物理内存中有效。当页面被交换到磁盘上或无效时，可以清除这个位来指示页面需要被替换。

 

当出现页访问异常时，硬件会先保存发生异常的地址和异常原因，然后传递给pgfault_handler函数处理异常。

 

page表示物理内存中的一页，页目录项用于管理一整个页表，页表项则记录了page在虚拟地址空间中的位置即虚拟页号以及相应的权限信息等。

 

#### 练习 4：补充完成 Clock 页替换算法（需要编程） 

通过之前的练习，相信大家对 FIFO 的页面替换算法有了更深入的了解，现在请在我们给出的框架上，填写

代码，实现 Clock 页替换算法（`mm/swap_clock.c`）。请在实验报告中简要说明你的设计实现过程。请回答如 

下问题： 

• 比较 Clock 页替换算法和 FIFO 算法的不同。

 

**设计实现过程：**

1. **_clock_init_mm函数**

先使用list_init函数初始化pra_list_head为空链表，再将当前指针curr_ptr和mm的私有成员指针sm_priv指向pra_list_head。

```c++
static int

_clock_init_mm(struct mm_struct *mm)

{   

   /*LAB3 EXERCISE 4: 2112106*/ 

   // 初始化pra_list_head为空链表

   // 初始化当前指针curr_ptr指向pra_list_head，表示当前页面替换位置为链表头

   // 将mm的私有成员指针指向pra_list_head，用于后续的页面替换算法操作

   //cprintf(" mm->sm_priv %x in fifo_init_mm\n",mm->sm_priv);

   list_init(&pra_list_head);

   curr_ptr=&pra_list_head;

   mm->sm_priv=&pra_list_head;

   cprintf(" mm->sm_priv %x in fifo_init_mm\n",mm->sm_priv);

   return 0;

}


```

2. **_clock_map_swappable函数**

先获取链表的头部节点储存在head中，由于clock算法维护的是环形链表，因此使用list_add_before函数将新加入的页面插入到head之前。最后将页面的visited标志为设为1。

```c++
static int

_clock_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)

{

  list_entry_t *entry=&(page->pra_page_link);

 

  assert(entry != NULL && curr_ptr != NULL);

  //record the page access situlation

  /*LAB3 EXERCISE 4: 2112106*/ 

  // link the most recent arrival page at the back of the pra_list_head qeueue.

  // 将页面page插入到页面链表pra_list_head的末尾

  // 将页面的visited标志置为1，表示该页面已被访问

  list_entry_t *head=(list_entry_t*) mm->sm_priv;

  list_add_before(head, entry);

  page->visited=1;

  return 0;

}
```

3. **_clock_swap_out_victim函数**

从当前指针curr_ptr开始循环遍历链表，如果遇到头部节点head则跳过，如果当前页面的visited标志位为1，则设置其visited标志位为0并继续查找，如果当前页面的visited标志位为0，则将该页面从页面链表中删除，并将该页面指针赋值给ptr_page作为换出页面。

```c++
static int

_clock_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)

{

   list_entry_t *head=(list_entry_t*) mm->sm_priv;

     assert(head != NULL);

   assert(in_tick==0);

   /* Select the victim */

   //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue

   //(2)  set the addr of addr of this page to ptr_page

  while (1) {

    /*LAB3 EXERCISE 4: 2112106*/ 

    // 编写代码

    // 遍历页面链表pra_list_head，查找最早未被访问的页面

    // 获取当前页面对应的Page结构指针

    // 如果当前页面未被访问，则将该页面从页面链表中删除，并将该页面指针赋值给ptr_page作为换出页面

    // 如果当前页面已被访问，则将visited标志置为0，表示该页面已被重新访问

    curr_ptr=list_next(curr_ptr);

    if (curr_ptr==head)

    {

      curr_ptr=list_next(curr_ptr);

    }

    struct Page *p=le2page(curr_ptr, pra_page_link);

    if (p->visited==0)

    {

      list_del(curr_ptr);

      *ptr_page=p;

      break;

    }

    else

    {

      p->visited=0;

    }

  }

  cprintf("curr_ptr %p\n", curr_ptr);

  return 0;

}
```



Clock 页替换算法和 FIFO 算法的不同:clock算法维护的是一个环形链表并且有一个指向当前页面的指针，而FIFO算法维护的为直链链表；而且clock算法中关心每个页面是否被访问过并以此作为是否换出的依据。

 

#### 练习 5：阅读代码和实现手册，理解页表映射方式相关知识（思考题）

如果我们采用”一个大页“的页表映射方式，相比分级页表，有什么好处、优势，有什么坏处、风险？

 

**好处和优势：**

1. 采用大页的方式可以减少页表项的数量，因为每个页表项映射一个大的连续地址范围，而不是小的单页。这减少了页表的大小，节省了内存。
2. 因为页表更小，遍历页表以查找虚拟地址到物理地址的映射会更快，这提高了访问性能。
3. TLB是用于缓存虚拟地址到物理地址的映射的硬件缓存。大页表减少了页表项的数量，减轻了TLB缓存的负担，提高了TLB的命中率，从而提高了访问内存的速度。
4. 大页表适用于需要大块连续内存的应用程序，如数据库管理系统，图形处理等。这样可以减少内存碎片化。

 

**劣势和风险：**

1. 采用大页的方式可能导致内部碎片，因为如果一个进程只使用了大页面的一部分，其余部分将浪费。这可能会浪费内存。
2. 如果应用程序需要小的内存块，例如只需要几个字节或几KB的内存，使用大页面可能会浪费大量内存，因为每个页面都是一个大的固定大小。
3. 如果一个大页面中的任何部分发生页面置换，将导致整个页面被换出到磁盘，这可能会导致更大的性能开销。
4. 不适用于虚拟地址空间大的进程，对于虚拟地址空间非常大的进程，大页表可能会导致更多的内存消耗。

 

 

 
