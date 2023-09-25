## ��ϰ1������ں������еĳ�����ڲ���


�Ķ� kern/init/entry.S���ݴ��룬��ϲ���ϵͳ�ں��������̣�˵��ָ�� la sp, bootstacktop �����ʲô������Ŀ����ʲô�� tail kern_init �����ʲô������Ŀ����ʲô��

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
|.section .text |���αָ��|�������ô���� |
|%progbits|���|ָ���������ǡ���ִ�д���Ρ� |
|.globl kern_entry|���αָ��|�� kern_entry ���Ϊһ��ȫ�ַ��ţ�ʹ���������������ļ�������|
|la sp, bootstacktop|���ָ��|��ȫ�ַ��� bootstacktop �ĵ�ַ���ص�ջָ��Ĵ��� sp ��|
|tail kern_init|���ָ��|������ת�� kern_init ������������ת֮ǰ���浱ǰ�ķ��ص�ַ���Ա��� kern_init �������غ����ִ��|

<mark style="background-color: yellow;">ָ�� la sp, bootstacktop Ŀ���ǳ�ʼ��ջָ��(�ڴ˶δ���ĩβ) </mark>

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

���У�`print_kerninfo()`���ӡ�������ݣ�����ûʲô��(��Ϊ������)
```
Special kernel symbols:
  entry  0x000000008020000c (virtual)
  etext  0x0000000080200952 (virtual)
  edata  0x0000000080204008 (virtual)
  end    0x0000000080204018 (virtual)
Kernel executable memory footprint: 17KB
```



<mark style="background-color: yellow;">ָ�� tail kern_init Ŀ������ת�� kern_init ������������������ʼ��һЩ����������ʱ���жϣ� </mark>



## ��ϰ2�������жϴ��� ����Ҫ��̣�
��������trap.c�е��жϴ�����trap���ڶ�ʱ���жϽ��д���Ĳ�����дkern/trap/trap.c�����д���ʱ���жϵĲ��֣�ʹ����ϵͳÿ����100��ʱ���жϺ󣬵���print_ticks�ӳ�������Ļ�ϴ�ӡһ�����֡�100 ticks�����ڴ�ӡ��10�к����sbi.h�е�shut_down()�����ػ���

Ҫ���������1�������غ���ʵ�֣��ύ�Ľ����Դ����������Ա���ִ�У�������ʵ�鱨���м�Ҫ˵��ʵ�ֹ��̺Ͷ�ʱ���ж��жϴ�������̡�ʵ��Ҫ��Ĳ��ִ������������ϵͳ����Լÿ1������һ�Ρ�100 ticks�������10�С�


<mark style="background-color: yellow;">��Ҫ��ȫ�Ĳ��ִ������� </mark>
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

<mark style="background-color: yellow;">�쳣����������ת��__alltraps��ǣ����������ģ���Ȼ�����trapentry.S�е�trap������trap_dispatch�ļ򵥷�װ����trap_dispatch�����Ǹ���̬�����������������ж����ͽ��зַ����Ӷ�������ز������ڱ����У������ۼӼ�����������ʱ���жϵȵȣ�����ɽ����󣬵���trapentry.S�е�RESTORE_ALL�����ָ������ģ��жϴ�������� </mark>