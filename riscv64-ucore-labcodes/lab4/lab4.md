**练习 1：分配并初始化一个进程控制块（需要编码）**

`alloc_proc` 函数（位于` kern/process/proc.c `中）负责分配并返回一个新的` struct proc_struct `结构，用于存储新建立的内核线程的管理信息。ucore 需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。 

【提示】在 `alloc_proc` 函数的实现中，需要初始化的 `proc_struct` 结构中的成员变量至少包括： 

`state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name`。 

请在实验报告中简要说明你的设计实现过程。请回答如下问题： 

• 请说明 `proc_struct` 中` struct context context `和 `struct trapframe *tf` 成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来） 

 

`alloc_proc`函数需要将`proc_struct`结构体`proc`中的成员变量初始化，除了一些成员变量设置了一些特殊值，其余均设为0。

```c
  proc->state = PROC_UNINIT; //设置进程为“初始”态

  proc->pid = -1; //设置进程pid的未初始化值

  proc->runs = 0; //设置运行次数为0

  proc->kstack = 0;

  proc->need_resched = 0;

  proc->parent = NULL;

  proc->mm = NULL;

  memset(&(proc->context), 0, sizeof(struct context));

  proc->tf = NULL;

  proc->cr3 = boot_cr3; //使用内核页目录表的基址

  proc->flags = 0;

  memset(proc->name, 0, PROC_NAME_LEN);
```

`proc_struct` 中` struct context context `和 `struct trapframe *tf `成员变量含义和在本实验中的作用为:

• `context`：`context` 中保存了进程执行的上下文，也就是几个关键的寄存器的值。这些寄存器的值用于在进程切换中还原之前进程的运行状态（进程切换的详细过程在后面会介绍）。切换过程的实现在`kern/process/switch.S`。 

• `tf`：`tf `里保存了进程的中断帧。当进程从用户空间跳进内核空间的时候，进程的执行状态被保存在了中断帧中（注意这里需要保存的执行状态数量不同于上下文切换）。系统调用可能会改变用户寄存器的值，我们可以通过调整中断帧来使得系统调用返回特定的值。

 

**练习 2：为新创建的内核线程分配资源（需要编码）**

创建一个内核线程需要分配和设置好很多资源。kernel_thread 函数通过调用 do_fork 函数完成具体内核线程的创建工作。`do_kernel `函数会调用` alloc_proc `函数来分配并初始化一个进程控制块，但 `alloc_proc` 只是找到 了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore 一般通过` do_fork `实际创建新的内 核线程。`do_fork `的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存 储位置不同。因此，我们实际需要”`fork`”的东西就是` stack` 和` trapframe`。在这个过程中，需要给新内核线 程分配资源，并且复制原进程的状态。你需要完成在 `kern/process/proc.c `中的 `do_fork `函数中的处理过程。它的大致执行步骤包括： 

• 调用 `alloc_proc`，首先获得一块用户信息块。 

• 为进程分配一个内核栈。 

• 复制原进程的内存管理信息到新进程（但内核线程不必做此事）

• 复制原进程上下文到新进程 

• 将新进程添加到进程列表 

• 唤醒新进程 

• 返回新进程号 

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

• 请说明 ucore 是否做到给每个新 `fork` 的线程一个唯一的 id？请说明你的分析和理由。

 

实现过程为：

- 调用 alloc_proc 分配一个 proc_struct 
- 调用 setup_kstack 为子进程分配一个内核栈 
- 调用 copy_mm 根据 clone_flag 复制或共享 mm（内存管理）信息
- 调用 copy_thread 在 proc_struct 中设置 tf（线程结构）和上下文 
- 将 proc_struct 插入 hash_list 和 proc_list 
- 调用 wakeup_proc 使新的子进程处于可运行状态 
- 使用子进程的 pid 设置返回值

 

实现代码如下:

```c
proc=alloc_proc();

  if(proc==NULL){

​    goto fork_out;

  }

  if(setup_kstack(proc)){

​    goto bad_fork_cleanup_kstack;

  }

  if(copy_mm(clone_flags,proc)){

​    goto bad_fork_cleanup_proc;

  }

  copy_thread(proc,stack,tf);

  bool intr_flag;

  local_intr_save(intr_flag);

  {

​    proc->pid = get_pid();

​    hash_proc(proc);

​    list_add(&proc_list, &(proc->list_link));

​    nr_process++;

  }

  local_intr_restore(intr_flag);

  wakeup_proc(proc);

  ret=proc->pid;
```

对于ucore 是否做到给每个新` fork` 的线程一个唯一的 id，分析`get_pid`函数。

```c
// get_pid - alloc a unique pid for process

static int

get_pid(void) {

  static_assert(MAX_PID > MAX_PROCESS);

  struct proc_struct *proc;

  list_entry_t *list = &proc_list, *le;

  static int next_safe = MAX_PID, last_pid = MAX_PID;

  if (++ last_pid >= MAX_PID) {

​    last_pid = 1;

​    goto inside;

  }

  if (last_pid >= next_safe) {

  inside:

​    next_safe = MAX_PID;

  repeat:

​    le = list;

​    while ((le = list_next(le)) != list) {

​      proc = le2proc(le, list_link);

​      if (proc->pid == last_pid) {

​        if (++ last_pid >= next_safe) {

​          if (last_pid >= MAX_PID) {

​            last_pid = 1;

​          }

​          next_safe = MAX_PID;

​          goto repeat;

​        }

​      }

​      else if (proc->pid > last_pid && next_safe > proc->pid) {

​        next_safe = proc->pid;

​      }

​    }

  }

  return last_pid;

}
```

上述代码通过维护两个关键变量 `last_pid `和 `next_safe `以及对进程链表的遍历，确保为每个新的`fork`线程分配一个唯一的进程ID。`last_pid` 是上一个分配的进程ID，通过不断递增它并检查是否被其他进程使用，确保新分配的ID是唯一的。同时，`next_safe` 用于追踪当前可用ID的上界，以避免重复分配。通过在链表上遍历，检查是否有其他进程使用了当前的` last_pid`，并根据情况更新` last_pid `和` next_safe`，代码保证了每次调用 `get_pid `都会成功分配一个唯一的进程ID给新的`fork`线程。

 

**练习 3：编写 proc_run 函数（需要编码）**

`proc_run` 用于将指定的进程切换到 CPU 上运行。它的大致执行步骤包括： 

• 检查要切换的进程是否与当前正在运行的进程相同，如果相同则不需要切换。

• 禁用中断。你可以使用`/kern/sync/sync.h`定义好的宏` local_intr_save(x)` 和`local_intr_restore(x)`来实现关、开中断。 

• 切换当前进程为要运行的进程。 

• 切换页表，以便使用新进程的地址空间。`/libs/riscv.h`中提供了` lcr3(unsigned int cr3)`函数，可实现修改 `CR3 `寄存器值的功能。 

• 实现上下文切换。`/kern/process `中已经预先编写好了` switch.S`，其中定义了 `switch_to() `函数。可实现两个进程的 `context` 切换。 

• 允许中断。 

请回答如下问题： 

• 在本实验的执行过程中，创建且运行了几个内核线程？

 

完善`proc_run`函数如下：

```c
// proc_run - make process "proc" running on cpu

// NOTE: before call switch_to, should load base addr of "proc"'s new PDT

void

proc_run(struct proc_struct *proc) {

  if (proc != current) {

​    // LAB4:EXERCISE3 YOUR CODE

​    /*

​    \* Some Useful MACROs, Functions and DEFINEs, you can use them in below implementation.

​    \* MACROs or Functions:

​    \*  local_intr_save():     Disable interrupts

​    \*  local_intr_restore():   Enable Interrupts

​    \*  lcr3():          Modify the value of CR3 register

​    \*  switch_to():        Context switching between two processes

​    */

​    bool intr_flag;

​    local_intr_save(intr_flag);

​    {

​      struct proc_struct *last=current;

​      current=proc;

​      lcr3(proc->cr3);

​      switch_to(&(last->context),&(proc->context));

​    }

​    local_intr_restore(intr_flag);

  }

}
```

本次实验的执行过程中，创建且运行了两个内核线程，分别为第 0 个内核线程` idleproc`表示空闲进程，用于完成内核中各个子线程的创建以及初始化和第 1 个内核线程` initproc`执行`init_main`。

 

 

**扩展练习 Challenge：**

• 说明语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag); `是如何实现开关中断的？ 

 

这里使用了两个宏：

`local_intr_save(x)`：这个宏的作用是保存当前中断状态，并将其保存在变量` x `即`intr_flag`中。具体的实现是通过调用` __intr_save()` 函数来获取当前中断状态，并将其保存在 x 变量即`intr_flag`中。

`local_intr_restore(x)`：这个宏的作用是恢复之前保存的中断状态，即将中断状态设置为` x` 变量即`intr_flag`保存的值。具体的实现是通过调用 `__intr_restore(x)` 函数来完成的。

 

 
