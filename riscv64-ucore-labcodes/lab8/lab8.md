### **练习 1: 完成读文件操作的实现（需要编码）**

首先了解打开文件的处理流程，然后参考本实验后续的文件读写操作的过程分析，填写在 kern/fs/sfs/sfs_inode.c中的 sfs_io_nolock() 函数，实现读文件中数据的代码。

 

在 sfs_io_nolock 函数中完成操作如下：

1. 先计算一些辅助变量，并处理一些特殊情况（比如越界），然后有 sfs_buf_op = sfs_rbuf,sfs_block_op = sfs_rblock，设置读取的函数操作。 

2. 接着进行实际操作，先处理起始的没有对齐到块的部分，再以块为单位循环处理中间的部分，最后处理末尾剩余的部分。

3. 每部分中都调用 sfs_bmap_load_nolock 函数得到 blkno 对应的 inode 编号，并调用 sfs_rbuf 或 sfs_rblock函数读取数据（中间部分调用 sfs_rblock，起始和末尾部分调用 sfs_rbuf），调整相关变量。

4. 完成后如果 offset + alen > din->fileinfo.size（写文件时会出现这种情况，读文件时不会出现这种情况，alen 为实际读写的长度），则调整文件大小为 offset + alen 并设置 dirty 变量。 

 

实现代码如下：

```c++
// 处理首块不对齐

  if (offset % SFS_BLKSIZE != 0) {

​    blkoff = offset % SFS_BLKSIZE;

​    size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);

​    // 调用相应的函数，处理首块不对齐的读取/写入操作

​    ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino);

​    if (ret != 0) {

​    goto out;

​    }

​    ret = sfs_buf_op(sfs, buf, size, ino, blkoff);

​    if (ret != 0) {

​      goto out;

​    }

​    alen += size;

​    if (nblks == 0) {

​      goto out;

​    }

​    blkno++;

​    buf += size;

​    nblks--;

  }

 

  // 处理对齐的块

  while (nblks > 0) {

​    // 调用相应的函数，处理对齐块的读取/写入操作

​    ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino);

​    if (ret != 0) {

​      goto out;

​    }

​    ret = sfs_block_op(sfs, buf, ino, 1);

​    if (ret != 0) {

​      goto out;

​    }

​    alen += SFS_BLKSIZE;

​    buf += SFS_BLKSIZE;

​    blkno++;

​    nblks--;

}

 

  // 处理尾块不对齐

  if (endpos % SFS_BLKSIZE != 0) {

​    size = endpos % SFS_BLKSIZE;

​    // 调用相应的函数，处理尾块不对齐的读取/写入操作

​    ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino);

​    if (ret != 0) {

​      goto out;

​    }

​    ret = sfs_buf_op(sfs, buf, size, ino, 0);  // 从块的开头开始读取/写入

​    if (ret != 0) {

​      goto out;

​    }

​    alen += size;

  }
```



### 练习 2: 完成基于文件系统的执行程序机制的实现（需要编码）

改写 proc.c 中的 load_icode 函数和其他相关函数，实现基于文件系统的执行程序机制。执行：make qemu。如果能看看到 sh 用户程序的执行界面，则基本成功了。如果在 sh 用户界面上可以执行”ls”,”hello”等其他放置在 sfs 文件系统中的其他执行程序，则可以认为本实验基本成功。 

 

load_icode 函数的主要工作就是给用户进程建立一个能够让用户进程正常运行的用户环境。此函数完成了如下重要工作：

1. 调用 mm_create 函数来申请进程的内存管理数据结构 mm 所需内存空间，并对 mm 进行初始化；调用setup_pgdir 来申请一个页目录表所需的一个页大小的内存空间，并把描述 ucore 内核虚空间映射的内核页表（boot_pgdir 所指）的内容拷贝到此新目录表中，最后让 mm->pgdir 指向此页目录表，这就是进程 新的页目录表了，且能够正确映射内核虚空间； 

2. 根据应用程序文件的文件描述符（fd）通过调用 load_icode_read（）函数来加载和解析此 ELF 格式的执行程序，并调用 mm_map 函数根据 ELF 格式的执行程序说明的各个段（代码段、数据段、BSS 段等）的起始位置和大小建立对应的 vma 结构，并把 vma 插入到 mm 结构中，从而表明了用户进程的合法用 户态虚拟地址空间； 

3. 调用根据执行程序各个段的大小分配物理内存空间，并根据执行程序各个段的起始位置确定虚拟地址，并在页表中建立好物理地址和虚拟地址的映射关系，然后把执行程序各个段的内容拷贝到相应的内核虚拟地址中，至此应用程序执行码和数据已经根据编译时设定地址放置到虚拟内存中了；

4. 需要给用户进程设置用户栈，为此调用 mm_mmap 函数建立用户栈的 vma 结构，明确用户栈的位置在用户虚空间的顶端，大小为 256 个页，即 1MB，并分配一定数量的物理内存且建立好栈的虚地址 <–>物理地址映射关系； 

5. 至此, 进程内的内存管理 vma 和 mm 数据结构已经建立完成，于是把 mm->pgdir 赋值到 cr3 寄存器中，即更新了用户进程的虚拟内存空间，，设置 uargc 和 uargv 在用户栈中。此时的 initproc 已经被代码和数据覆盖，成为了第一个用户进程，但此时这个用户进程的执行现场还没建立好； 

6. 先清空进程的中断帧，再重新设置进程的中断帧，使得在执行中断返回指令“iret”后，能够让 CPU 转到用户态特权级，并回到用户态内存空间，使用用户态的代码段、数据段和堆栈，且能够跳转到用户进程的第一条指令执行，并确保在用户态能够响应中断；

实验 8 和实验 5 中 load_icode（）函数代码最大不同的地方在于读取 EFL 文件的方式，实验 5 中是通过获取 ELF 在内存中的位置，根据 ELF 的格式进行解析，而在实验 8 中则是通过 ELF 文件的文件描述符调用 

load_icode_read（）函数来进行解析程序。

 

代码实现如下：

```c++
assert(argc >= 0 && argc <= EXEC_MAX_ARG_NUM);

  

  if (current->mm != NULL) {

​    panic("load_icode: current->mm must be empty.\n");

  }

 

  int ret = -E_NO_MEM;

  struct mm_struct *mm;

  //(1) create a new mm for current process

  if ((mm = mm_create()) == NULL) {

​    goto bad_mm;

  }

  //(2) create a new PDT, and mm->pgdir= kernel virtual addr of PDT

  if (setup_pgdir(mm) != 0) {

​    goto bad_pgdir_cleanup_mm;

  }

  //(3) copy TEXT/DATA section, build BSS parts in binary to memory space of process

  struct Page *page;

  //(3.1) get the file header of the bianry program (ELF format)

  struct elfhdr elf_content;

  struct elfhdr *elf = &elf_content;

  //(3.2) get the entry of the program section headers of the bianry program (ELF format)

  struct proghdr ph_content;

  struct proghdr *ph = &ph_content;

  // 读取ELF文件头

  if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0) {

​    goto bad_elf_cleanup_pgdir;

}

 

  //(3.3) This program is valid?

  if (elf->e_magic != ELF_MAGIC) {

​    ret = -E_INVAL_ELF;

​    goto bad_elf_cleanup_pgdir;

  }

  

  uint32_t vm_flags, perm, phnum = 0;

  struct proghdr *ph_end = ph + elf->e_phnum;

  // 遍历 ELF 文件的程序头表

  for (; phnum < elf->e_phnum; phnum ++) {

​    if ((ret = load_icode_read(fd, ph, sizeof(struct proghdr), elf->e_phoff + sizeof(struct proghdr) * phnum)) != 0) {

​      goto bad_cleanup_mmap;

​    }

​    //(3.4) find every program section headers

​    if (ph->p_type != ELF_PT_LOAD) {

​      continue ;

​    }

​    if (ph->p_filesz > ph->p_memsz) {

​      ret = -E_INVAL_ELF;

​      goto bad_cleanup_mmap;

​    }

​    if (ph->p_filesz == 0) {

​      // continue ;

​    }

​    //(3.5) call mm_map fun to setup the new vma ( ph->p_va, ph->p_memsz)

​    vm_flags = 0, perm = PTE_U | PTE_V;

​    if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;

​    if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;

​    if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;

​    // modify the perm bits here for RISC-V

​    if (vm_flags & VM_READ) perm |= PTE_R;

​    if (vm_flags & VM_WRITE) perm |= (PTE_W | PTE_R);

​    if (vm_flags & VM_EXEC) perm |= PTE_X;

​    if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0) {

​      goto bad_cleanup_mmap;

​    }

​    off_t offset = ph->p_offset;

​    size_t off, size;

​    uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);

 

​    ret = -E_NO_MEM;

 

​    //(3.6) alloc memory, and  copy the contents of every program section (from, from+end) to process's memory (la, la+end)

​    end = ph->p_va + ph->p_filesz;

​    //(3.6.1) copy TEXT/DATA section of bianry program

​    while (start < end) {

​      if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {

​        ret = -E_NO_MEM;

​        goto bad_cleanup_mmap;

​      }

​      off = start - la, size = PGSIZE - off, la += PGSIZE;

​      if (end < la) {

​        size -= la - end;

​      }

​      if ((ret = load_icode_read(fd, page2kva(page) + off, size, offset)) != 0) {

​        goto bad_cleanup_mmap;

​      }

​      start += size, offset += size;

​    }

 

​    //(3.6.2) build BSS section of binary program

​    end = ph->p_va + ph->p_memsz;

​    if (start < la) {

​      /* ph->p_memsz == ph->p_filesz */

​      if (start == end) {

​        continue ;

​      }

​      off = start + PGSIZE - la, size = PGSIZE - off;

​      if (end < la) {

​        size -= la - end;

​      }

​      memset(page2kva(page) + off, 0, size);

​      start += size;

​      assert((end < la && start == end) || (end >= la && start == la));

​    }

​    while (start < end) {

​      if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {

​        ret = -E_NO_MEM;

​        goto bad_cleanup_mmap;

​      }

​      off = start - la, size = PGSIZE - off, la += PGSIZE;

​      if (end < la) {

​        size -= la - end;

​      }

​      memset(page2kva(page) + off, 0, size);

​      start += size;

​    }

}

 

sysfile_close(fd);

 

  //(4) build user stack memory

  vm_flags = VM_READ | VM_WRITE | VM_STACK;

  if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {

​    goto bad_cleanup_mmap;

  }

  assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);

  assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);

  assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);

  assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);

  

  //(5) set current process's mm, sr3, and set CR3 reg = physical addr of Page Directory

  mm_count_inc(mm);

  current->mm = mm;

  current->cr3 = PADDR(mm->pgdir);

lcr3(PADDR(mm->pgdir));

 

  // 计算 argv_size

  uint32_t argv_size = 0;

  int i;

  for (i = 0; i < argc; i++) {

​    argv_size += strnlen(kargv[i], EXEC_MAX_ARG_LEN + 1) + 1;

}

 

  // 将用户栈的顶部 USTACKTOP 向下调整，以便为参数和参数指针腾出足够的空间

  uintptr_t stacktop = USTACKTOP - (argv_size / sizeof(long) + 1) * sizeof(long);

  // 为参数指针数组分配内存

  char **uargv = (char **)(stacktop - argc * sizeof(char *));

  // 复制参数字符串到用户栈

  argv_size = 0;

  for (i = 0; i < argc; i++) {

​    uargv[i] = strcpy((char *)(stacktop + argv_size), kargv[i]);

​    argv_size += strnlen(kargv[i], EXEC_MAX_ARG_LEN + 1) + 1;

  }

  // 为参数个数分配空间并设置用户栈指针

  stacktop = (uintptr_t)uargv - sizeof(int);

*(int *)stacktop = argc;

 

  //(6) setup trapframe for user environment

  struct trapframe *tf = current->tf;

  // Keep sstatus

  uintptr_t sstatus = tf->status;

  memset(tf, 0, sizeof(struct trapframe));

  /* LAB5:EXERCISE1 YOUR CODE

   \* should set tf->gpr.sp, tf->epc, tf->status

   \* NOTICE: If we set trapframe correctly, then the user level process can return to USER MODE from kernel. So

   \*      tf->gpr.sp should be user stack top (the value of sp)

   \*      tf->epc should be entry point of user program (the value of sepc)

   \*      tf->status should be appropriate for user program (the value of sstatus)

   \*      hint: check meaning of SPP, SPIE in SSTATUS, use them by SSTATUS_SPP, SSTATUS_SPIE(defined in risv.h)

   */

  // sp设置为用户栈的顶部

  tf->gpr.sp = USTACKTOP;

  // epc设置为ELF的入口点

  tf->epc = elf->e_entry;

  // SPP置0表示用户模式，SPIE置1表示允许中断

tf->status = (sstatus & ~SSTATUS_SPP) | SSTATUS_SPIE;

 

  ret = 0;

out:

  return ret;

bad_cleanup_mmap:

  exit_mmap(mm);

bad_elf_cleanup_pgdir:

  put_pgdir(mm);

bad_pgdir_cleanup_mm:

  mm_destroy(mm);

bad_mm:

  goto out;
```



 