##### 上电到复位地址

在RISC-V 硬件加电后，使用x/10i $pc指令显示即将执行的 10 条汇编指令为
`0x1000:	auipc	t0,0x0`
`0x1004:	addi	a1,t0,32`
`0x1008:	csrr	a0,mhartid`
`0x100c:	ld	t0,24(t0)`
`0x1010:	jr	t0`
`0x1014:	unimp`
`0x1016:	unimp`
`0x1018:	unimp`
`0x101a:	0x8000`
`0x101c:	unimp`

发现第一条指令的地址为0x1000说明CPU的复位地址为0x1000。对此查看qemu4.1.1的源码，在CPU初始化的文件cpu.c中114行处发现下面的代码
`set_resetvec(env, DEFAULT_RSTVEC);`

在该代码中宏定义DEFAULT_RSTVEC被赋值给了resetvec。在cpu_bits.h中479行处查看该宏定义的代码为
`/* Default Reset Vector adress */`
`#define DEFAULT_RSTVEC      0x1000`

发现其的值正是复位地址0x1000。在cpu.c文件中继续查找resetvec在297行处发现如下代码
`env->pc = env->resetvec;`

则说明resetvec复位地址被赋值给了pc，0x1000正是CPU的复位地址。

##### 从复位地址到boot loader

在初始化riscv开发板的文件virt.c文件中447行处发现如下代码
rom_add_blob_fixed_as("mrom.reset", reset_vec, sizeof(reset_vec),
                     memmap[VIRT_MROM].base, &address_space_memory);

说明将复位地址reset_vec放在了VIRT_MROM的基址处。

同时在408行处发现
riscv_find_and_load_firmware(machine, BIOS_FILENAME,
                                 memmap[VIRT_DRAM].base);//memmap[VIRT_MROM].base=0x80000000

说明在此时将ROM加载到了boot loader 0x80000000的位置。

由以上分析可以得知CPU从上电到加载boot loader的过程为，在ROM中存放了一段复位代码，CPU上电后从复位地址0x1000处开始执行复位代码，然后在复位代码中跳转到boot loader 0x80000000处。

使用gdb逐条执行指令，直至跳转到0x80000000处，结果如下
`(gdb) si`
`0x0000000000001004 in ?? ()`
`(gdb) si`
`0x0000000000001008 in ?? ()`
`(gdb) si`
`0x000000000000100c in ?? ()`
`(gdb) si`
`0x0000000000001010 in ?? ()`
`(gdb) si`
`0x0000000080000000 in ?? ()`

发现执行完0x1010:	jr  t0 指令后跳转到0x80000000处，说明该指令即为跳转到boot loader的指令代码。
通过查看各寄存器的值可了解0x1000到0x1010之间各指令的作用。

- auipc  t0,0x0     将0x0扩展到当前pc的高20位，并将绝对地址0x1000储存在寄存器t0中。
- addi	a1,t0,32      将t0中的值加上32储存在寄存器a1中
- csrr	a0,mhartid  读取CSR寄存器mmhartid中的数据储存在a0中
- ld	   t0,24(t0)    从位于 t0 寄存器所指示的内存地址中，偏移 24 字节的位置处加载一个双字（64位）的数据，并将其放入寄存器 t0 中，此时t0的值即为boot loader 0x80000000
- jr	   t0          跳转到t0地址即为0x80000000

##### 从0x80000000到0x80200000

接着查看并执行0x80000000后的指令。在0x80200000处设置断点后运行到断点位置，发现在qemu模拟计算机的终端处输出

![image-20230917154025019](C:\Users\24505\Desktop\lab0.5.assets\image-20230917154025019.png)


说明OPENSBI程序成功执行将操作系统加载到内存并将要执行应用程序的第一条指令。