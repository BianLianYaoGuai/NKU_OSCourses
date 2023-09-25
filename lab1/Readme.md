## 练习1：理解内核启动中的程序入口操作


阅读 kern/init/entry.S内容代码，结合操作系统内核启动流程，说明指令 la sp, bootstacktop 完成了什么操作，目的是什么？ tail kern_init 完成了什么操作，目的是什么？

### entry.S
```
#include <mmu.h>
#include <memlayout.h>

    .section .text,"ax",%progbits
    .globl kern_entry
kern_entry:
    la sp, bootstacktop

    tail kern_init

.section .data
    # .align 2^12
    .align PGSHIFT
    .global bootstack
bootstack:
    .space KSTACKSIZE
    .global bootstacktop
bootstacktop:

```
||||
|-|-|-|
|.section .text |汇编伪指令|用于设置代码段 |
|%progbits|标记|指以下内容是“可执行代码段” |
|.globl kern_entry|汇编伪指令|将 kern_entry 标记为一个全局符号，使得它可以在其他文件中引用|
|la sp, bootstacktop|汇编指令|将全局符号 bootstacktop 的地址加载到栈指针寄存器 sp 中|
|tail kern_init|汇编指令|它会跳转到 kern_init 函数并且在跳转之前保存当前的返回地址，以便在 kern_init 函数返回后继续执行|

<mark style="background-color: yellow;">指令 la sp, bootstacktop 目的是初始化栈指针(在此段代码末尾) </mark>

### init.c

```c
int kern_init(void) __attribute__((noreturn));
int kern_init(void) {
    extern char edata[], end[];
    memset(edata, 0, end - edata);

    cons_init();  // init the console

    const char *message = ":)os is loading ...\n";
    cprintf("%s\n\n", message);

    print_kerninfo();

    // grade_backtrace();

    idt_init();  // init interrupt descriptor table

    // rdtime in mbare mode crashes
    clock_init();  // init clock interrupt

    intr_enable();  // enable irq interrupt
    
    while (1)
        ;
}
```

其中，`print_kerninfo()`会打印如下内容，对我没什么用(因为看不懂)
```
Special kernel symbols:
  entry  0x000000008020000c (virtual)
  etext  0x0000000080200952 (virtual)
  edata  0x0000000080204008 (virtual)
  end    0x0000000080204018 (virtual)
Kernel executable memory footprint: 17KB
```



<mark style="background-color: yellow;">指令 tail kern_init 目的是跳转到 kern_init 函数，在这个函数会初始化一些东西（包括时钟中断） </mark>



## 练习2：完善中断处理 （需要编程）
请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写kern/trap/trap.c函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”，在打印完10行后调用sbi.h中的shut_down()函数关机。

要求完成问题1提出的相关函数实现，提交改进后的源代码包（可以编译执行），并在实验报告中简要说明实现过程和定时器中断中断处理的流程。实现要求的部分代码后，运行整个系统，大约每1秒会输出一次”100 ticks”，输出10行。


<mark style="background-color: yellow;">需要补全的部分代码如下 </mark>
```C
    clock_set_next_event();
    if(++ticks%TICK_NUM==0)
    {
        print_ticks();
        if(ticks==1000)
        {
            sbi_shutdown();
        }
    }
```

<mark style="background-color: yellow;">异常产生，会跳转到__alltraps标记（保存上下文），然后调用trapentry.S中的trap函数（trap_dispatch的简单封装）。trap_dispatch函数是个静态内联函数，负责按照中断类型进行分发，从而进行相关操作（在本例中，就是累加计数器、重设时钟中断等等）。完成结束后，调用trapentry.S中的RESTORE_ALL函数恢复上下文，中断处理结束。 </mark>
