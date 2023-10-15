**练习 1：理解 first-fit 连续物理内存分配算法（思考题）** 

`first-fit` 连续物理内存分配算法作为物理内存分配一个很基础的方法，需要同学们理解它的实现过程。请大家仔细阅读实验手册的教程并结合 `kern/mm/default_pmm.c `中的相关代码，认真分析 `default_init，default_init_memmap，default_alloc_pages，default_free_pages` 等相关函数，并描述程序在进行物理内存分配的过程以及各个函数的作用。请在实验报告中简要说明你的设计实现过程。请回答如下问题：你的 `first fit`算法是否有进一步的改进空间？

 

`First-fit`连续物理内存分配算法的流程为：

1.管理内存块列表：操作系统维护一个内存块列表，其中包含了系统中可用的内存块的信息，包括起始地址和大小。

2.请求内存分配：当应用程序或进程需要分配一定大小的内存时，它向操作系统提出请求。

3.遍历内存块列表：操作系统从内存块列表的开头开始遍历，寻找第一个足够大的内存块来满足请求。

4.分配内存：一旦找到足够大的内存块，操作系统将该内存块分配给请求的进程，并在内存块列表中更新相关信息，如起始地址和剩余空闲内存块的大小。

5.完成分配：分配内存后，操作系统将内存块的起始地址返回给请求的进程，使其可以使用该内存块。

6.如果没有足够大的内存块可用，分配失败，操作系统可以采取不同的策略，例如等待，释放部分内存，或者合并已分配的内存块以腾出足够的空间。

 

在具体代码中，`default_init`函数用于初始化自由页帧的链表`free_list`，并将当前空闲块的数量定义为0。

 

`default_init_memmap`函数用于初始化`page`数组的内存块。接下来分析该函数是实现方法：

`assert(n > 0);`：这行代码用于确保传递给函数的页面数量 n 大于零，如果不满足条件，将会触发断言失败。

`struct Page *p = base;`：这一行声明一个指向 `struct Page` 结构体的指针 `p`，并将其初始化为指向传递给函数的` base `指针，表示要初始化的物理页面数组的起始地址。

`for (; p != base + n; p ++)`：这是一个循环，从 `base` 开始，依次遍历 n 个物理页面的结构体。

`assert(PageReserved(p));`：在每次循环迭代中，它使用宏` PageReserved` 来检查页面是否被保留。如果页面没有被保留，将会触发断言失败。这是用于确保被初始化的页面帧都是保留的。

`p->flags = p->property = 0;`：将页面的标志位和属性字段初始化为0。

`set_page_ref(p, 0);`：将页面的引用计数初始化为0。

`base->property = n;`：将传递给函数的 `base` 页面的 `property` 字段设置为 n，表示这些页面的属性为 n。

`SetPageProperty(base);`：调用宏 `SetPageProperty` 来设置 `base `页面的属性标志位，表示这些页面拥有某种属性。

`nr_free += n;`：增加全局变量 `nr_free`，该变量通常用于跟踪系统中可用的物理页面数量，这里增加了 n 个页面的数量。

`if (list_empty(&free_list))`：检查自由列表 `free_list` 是否为空。如果为空，将 `base` 页面的` page_link` 添加到自由列表中。如果自由列表不为空，进入下一个分支。

`list_entry_t* le = &free_list;`：声明一个指向自由列表头部的指针 `le`，用于遍历自由列表。

`while ((le = list_next(le)) != &free_list)`：这是一个循环，用于遍历自由列表中的页面帧。

`struct Page* page = le2page(le, page_link);`：将当前遍历到的链表节点 `le `转换为相应的` struct Page` 结构体的指针。

`if (base < page)`：比较当前要初始化的` base `页面和遍历到的` page `页面的地址。如果` base` 的地址小于 `page`，则将 `base` 插入到 `page `之前，以确保页面链表的有序性。

`else if (list_next(le) == &free_list)`：如果` base `不小于当前遍历的` page `且已经遍历到了自由列表的末尾，则将 `base` 添加到自由列表的末尾，以确保有序性。

 

`default_alloc_pages`函数用于为程序的请求分配一组物理页面帧内存，接下来分析该函数是实现方法：

`assert(n > 0);`：这行代码用于确保请求分配的页面数量 n 大于零，如果不满足条件，将会触发断言失败。

`if (n > nr_free)`：这行代码检查系统中可用的物理页面数量是否足够分配请求的页面数量 n。如果可用页面数量不足，返回 `NULL`，表示分配失败。

`struct Page *page = NULL;`：声明一个指向 `struct Page `结构体的指针 `page`，用于存储分配到的第一个页面帧的描述符。初始值为` NULL`，表示暂时没有分配成功。

`list_entry_t *le = &free_list;`：声明一个指向自由列表 `free_list` 头部的指针` le`，用于遍历自由列表中的页面帧。

`while ((le = list_next(le)) != &free_list)`：这是一个循循环，用于遍历自由列表中的页面帧。

`struct Page *p = le2page(le, page_link);`：将当前遍历到的链表节点 `le `转换为相应的` struct Page `结构体的指针。

`if (p->property >= n)`：比较当前遍历到的页面帧 p 的属性（property）是否大于等于请求的页面数量 n。如果是，表示找到了足够大的页面帧，将 `page `指针指向这个页面帧，并跳出循环。

`if (page != NULL)`：如果成功找到足够大的页面帧。

`list_entry_t* prev = list_prev(&(page->page_link));`：获取` page` 页面帧的前一个页面帧的链表节点。

`list_del(&(page->page_link));`：从自由列表中删除 `page `页面帧的链表节点。

`if (page->property > n)`：检查 `page `页面帧的属性是否大于请求的页面数量 n。如果是，表示` page `页面帧剩余了一部分空闲空间。创建一个新的 `struct Page` 结构体 `p`，表示剩余的页面帧。

`p->property = page->property - n;`：设置` p `页面帧的属性，表示它的大小。

`SetPageProperty(p);`：调用宏 SetPageProperty 来设置 `p `页面帧的属性标志位，表示它拥有某种属性。

`list_add(prev, &(p->page_link));`：将 `p `页面帧的链表节点插入到自由列表中，保持链表有序。

`nr_free -= n;`：减少全局变量 `nr_free`，表示已分配的页面数量。

`ClearPageProperty(page);`：调用宏 `ClearPageProperty `来清除 `page` 页面帧的属性标志位，表示它不再具有某种属性。

 

返回 `page `页面帧的描述符指针，表示成功分配的第一个页面帧。

  

`default_free_pages`函数用于释放一组物理页面帧并将它们添加到`free_list`中，接下来分析该函数是实现方法：

`assert(n > 0);`：这行代码用于确保要释放的页面数量 n 大于零，如果不满足条件，将会触发断言失败。

`struct Page *p = base;`：这一行声明一个指向` struct Page `结构体的指针` p`，并将其初始化为指向传递给函数的 `base `指针，表示要释放的物理页面数组的起始地址。

`for (; p != base + n; p ++)`：这是一个循环，从` base `页面开始，逐个遍历 n 个物理页面帧的描述符。

`assert(!PageReserved(p) && !PageProperty(p));`：在每次循环迭代中，它使用宏` PageReserved` 和 `PageProperty` 来确保页面没有被保留并且没有某种属性。如果页面被保留或具有属性，将会触发断言失败。

`p->flags = 0;`：将页面的标志位清零。

`set_page_ref(p, 0);`：将页面的引用计数设置为0，表示页面不再被引用。

`base->property = n;`：将传递给函数的` base` 页面的` property` 字段设置为 n，表示这些页面的属性为 n。

`SetPageProperty(base);`：调用宏` SetPageProperty `来设置` base `页面的属性标志位，表示这些页面拥有某种属性。

`nr_free += n;`：增加全局变量` nr_free`，表示系统中可用的物理页面数量增加了 n 个。

`if (list_empty(&free_list))`：检查自由列表 `free_list `是否为空。如果为空，将 `base` 页面的 `page_link` 添加到自由列表中。

如果自由列表不为空，进入下一个分支。

`list_entry_t* le = &free_list;`：声明一个指向自由列表头部的指针 `le`，用于遍历自由列表。

`while ((le = list_next(le)) != &free_list)`：这是一个循环，用于遍历自由列表中的页面帧。

`struct Page* page = le2page(le, page_link);`：将当前遍历到的链表节点` le `转换为相应的 `struct Page `结构体的指针。

`if (base < page)`：比较当前要释放的` base` 页面和遍历到的` page` 页面的地址。如果 `base `的地址小于 `page`，则将 `base` 插入到` page` 之前，以确保页面链表的有序性。

`else if (list_next(le) == &free_list)`：如果 `base `不小于当前遍历的` page `且已经遍历到了自由列表的末尾，则将` base `添加到自由列表的末尾，以确保有序性。

`list_entry_t* le = list_prev(&(base->page_link));`：获取` base `页面帧的前一个页面帧的链表节点。

如果前一个页面帧存在（不是自由列表的头节点），继续下面的操作。

`p = le2page(le, page_link);`：将前一个页面帧的链表节点` le `转换为相应的 `struct Page `结构体的指针。

检查是否当前要释放的 `base` 页面与前一个页面帧相邻。如果相邻，将它们合并成一个更大的页面帧，更新属性和清除 `base` 页面帧的属性标志位，从自由列表中删除 `base `页面帧。

`le = list_next(&(base->page_link));`：获取` base `页面帧的下一个页面帧的链表节点。

如果下一个页面帧存在，继续下面的操作。

`p = le2page(le, page_link);`：将下一个页面帧的链表节点` le `转换为相应的` struct Page `结构体的指针。

检查是否当前要释放的` base `页面与下一个页面帧相邻。如果相邻，将它们合并成一个更大的页面帧，更新属性和清除下一个页面帧的属性标志位，从自由列表中删除下一个页面帧。

 

`first-fit`算法的优点是非常简单，容易实现，但它可能导致内存碎片问题，因为它只会选择第一个符合条件的内存块，而不会尝试最佳匹配。这意味着在内存中可能会出现大量小的碎片，这些碎片可能不足以满足稍后的内存请求。为了解决碎片问题，可以使用其他内存分配算法，如`best-fit`和 `worst-fit`算法，以更好地管理内存碎片。

 

 

**练习 2：实现 Best-Fit 连续物理内存分配算法（需要编程）** 

在完成练习一后，参考 `kern/mm/default_pmm.c `对` First Fit `算法的实现，编程实现 `Best Fit` 页面分配算法，算法的时空复杂度不做要求，能通过测试即可。请在实验报告中简要说明你的设计实现过程，阐述代码是如何对物理内存进行分配和释放，并回答如下问题：

• 你的 `Best-Fit` 算法是否有进一步的改进空间？

 

先定义一个`min_size`用于储存最小的自由内存块大小与所需内存块大小的差，遍历自由页帧的列表`free_list`，如果找到一个内存块的大小大于所需内存块的大小且它们之差小于`min_size`，则将当前的内存块的首个指针赋值给`page`，并将差赋值给`min_size`，这样遍历完`free_list`后就能够找到大小最接近所需内存的块。

`Best-Fit`算法的缺点在于需要在可用内存块列表中搜索，这可能会导致较长的搜索时间，特别是在内存块较多时。并且这样的内存分割可能会导致进一步的碎片化，因为分割后剩下的内存块会特别小可能不能够容纳未来的进程。

 

 