# 探索

## 1 上电后的第一个moment


采用双终端的方式进行调试
```bash
# 终端一
make debug
```
```bash
# 终端二
make gdb
```
（以下操作均在gdb终端进行）

```bash
x/10i $pc
# 显示即将执行的10条汇编指令
```

输出如下
```Plain Text
=> 0x1000:      auipc   t0,0x0
   0x1004:      addi    a1,t0,32
   0x1008:      csrr    a0,mhartid
   0x100c:      ld      t0,24(t0)
   0x1010:      jr      t0
   0x1014:      unimp
   0x1016:      unimp
   0x1018:      unimp
   0x101a:      0x8000
   0x101c:      unimp
```


## 2 粗略观察这个moment

先简略看一下jr（跳转）以前每条指令的效果

|          指令       |       效果
|---------------------|-----------|
|auipc   t0,0x0       |将当前PC（程序计数器）的值左移12位并存储到寄存器t0中|
|addi    a1,t0,32     |将寄存器t0的值加上32，并将结果存储在寄存器a1中|
|csrr    a0,mhartid   |从CSR（控制和状态寄存器）中读取mhartid（核心硬件线程ID）的值，并将其存储在寄存器a0中|
|ld      t0,24(t0)    |从寄存器t0中读取地址（当前t0的值），然后将地址偏移24个字节（0x18个字节），并将结果存储回寄存器t0中|
|jr      t0           |跳转到寄存器t0中存储的地址，实现了无条件跳转。它将程序的控制权转移到t0中存储的地址处|
|unimp|伪指令，通常用于表示未实现或不支持的指令。在这里，它可能表示在这个位置上不应该有有效的指令|

然后周期性地执行以下命令，观察不同寄存器的值。其中，`si`命令用于指示qemu程序执行一条汇编，`info r`命令用于查询寄存器pc、t0、a1、a0的值。
```bash
info r pc t0 a1 a0
si

```

每轮的输出结果汇总如下
```Plain Text
pc             0x0000000000001000       4096
t0             0x0000000000000000       0
a1             0x0000000000000000       0
a0             0x0000000000000000       0
0x0000000000001004 in ?? ()

pc             0x0000000000001004       4100
t0             0x0000000000001000       4096
a1             0x0000000000000000       0
a0             0x0000000000000000       0
0x0000000000001008 in ?? ()

pc             0x0000000000001008       4104
t0             0x0000000000001000       4096
a1             0x0000000000001020       4128
a0             0x0000000000000000       0
0x000000000000100c in ?? ()


pc             0x000000000000100c       4108
t0             0x0000000000001000       4096
a1             0x0000000000001020       4128
a0             0x0000000000000000       0
0x0000000000001010 in ?? ()

pc             0x0000000000001010       4112
t0             0x0000000080000000       2147483648
a1             0x0000000000001020       4128
a0             0x0000000000000000       0
0x0000000080000000 in ?? ()
```

## 3 仔细分析这个moment的指令

### (1) auipc   t0,0x0

`auipc`是Add Upper Immediate to PC的缩写，它的指令格式如下：
```
auipc rd, imm[63:12]
```

在64位系统中，各个字段的含义和位数如下：

- `auipc`: 操作码字段，通常占据7位。
- `rd`: 目标寄存器，用于存储计算后的地址，通常占据5位。
- `imm[63:12]`: 52位的立即数，它将左移12位，然后添加到当前PC值上(&+5+52 = 52+12 = 64，这不是巧合，这是设计！！)。

很明显，像`auipc   t0,0x0`这样的指令就相当于把pc寄存器的值复制到t0寄存器中


### （2）addi    a1,t0,32

`addi` 指令用于将一个寄存器(t0)中的值与一个立即数（十进制的32）相加，并将结果存储在目标寄存器(a1)中。

### （3）csrr    a0,mhartid

`csrr` 指令用于查询`mhartid`（Hart ID）。`mhartid` 是 RISC-V 架构中的一个 CSR，它表示当前运行的硬件线程（或称为 "hart"，通常对应于一个处理器核心）的唯一标识符。

执行这条汇编代码后，`a0` 寄存器的值仍然是 0，我觉得可能的原因是 **CSR 不可访问**。也就是说，在某些情况下，特权级别可能不允许对某些 CSR 进行访问。如果当前的特权级别不足以访问 `mhartid`，则这条指令可能不会修改 `a0` 的值。

对此，我查到了一篇知乎文章[RISC-V特权等级与Linux内核的启动](https://zhuanlan.zhihu.com/p/164394603)，但是目前还没看





# 安装编译器

## 1 下载

在[官网](https://d2pn104n81t9m2.cloudfront.net/products/tools/) ，找到 “Prebuilt RISC?V GCC Toolchain and Emulator”，下载“GNU Embedded Toolchain ”中适合操作系统的版本

## 2 路径

```bash
vim ~/.bashrc
```

添加内容如下：
```
export RISCV=/home/blyg/Documents/RISCV/riscv64-unknown-elf-gcc-2018.07.0-x86_64-linux-ubuntu14
export PATH=$RISCV/bin:$PATH
```
```bash
source ~/.bashrc
```

# 安装模拟器

## 1 下载并安装
```bash
wget https://download.qemu.org/qemu-4.1.1.tar.xz
tar xvJf qemu-4.1.1.tar.xz
cd qemu-4.1.1
./configure --target-list=riscv32-softmmu,riscv64-softmmu
make -j
sudo make install
```

## 2 路径

```bash
vim ~/.bashrc
```

添加内容如下：
```
export QEMU411=/home/blyg/Documents/QEMU/qemu-4.1.1
export PATH=$QEMU411/riscv32-softmmu:$QEMU411/riscv64-softmmu:$PATH
```
```bash
source ~/.bashrc
```

