# Lab 1: Booting a PC

阅读：https://pdos.csail.mit.edu/6.828/2018/labs/lab1/

这个实验由三部分组成，第一部分主要是为了熟悉使用 x86 汇编语言、QEMU x86 仿真器、以及 PC 的加电引导过程。第二部分查看我们的 6.828 内核的引导加载器，它位于 lab 的 boot 目录中。第三部分深入到名为 JOS 的 6.828 内核模型内部，它在 kernel 目录中。

## 1. 环境配置

    % mkdir ~/6.828
    % cd ~/6.828
    % git clone https://pdos.csail.mit.edu/6.828/2018/jos.git lab
    Cloning into lab...
    % cd lab
    % 

接下来阅读 [tools](https://pdos.csail.mit.edu/6.828/2018/tools.html) 进行环境配置。


环境：WSL2 ubuntu20.04

    sudo apt-get install -y build-essential gdb
    sudo apt-get install gcc-multilib

    git clone https://github.com/mit-pdos/6.828-qemu.git qemu
    sudo apt-get install libsdl1.2-dev libtool-bin libglib2.0-dev libz-dev libpixman-1-dev
    ./configure --disable-kvm --disable-werror --target-list="i386-softmmu x86_64-softmmu"

出错：

    /usr/bin/ld: qga/commands-posix.o: in function `dev_major_minor':
    /home/yunwei/qemu/qga/commands-posix.c:633: undefined reference to `major'
    /usr/bin/ld: /home/yunwei/qemu/qga/commands-posix.c:634: undefined reference to `minor'
    collect2: error: ld returned 1 exit status

在 `qga/commands-posix.c` 文件中加上头文件: `#include<sys/sysmacros.h>`

    make && make install

进入 lab 报错：

    $ make
    + ld obj/kern/kernel
    ld: warning: section `.bss' type changed to PROGBITS
    ld: obj/kern/printfmt.o: in function `printnum':
    lib/printfmt.c:41: undefined reference to `__udivdi3'
    ld: lib/printfmt.c:49: undefined reference to `__umoddi3'
    make: *** [kern/Makefrag:71: obj/kern/kernel] Error 1

解决方案是安装 4.8 的 gcc ，但是报错，原因是这个包没有在这个源中。

    $ sudo apt-get install -y gcc-4.8-multilib
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    E: Unable to locate package gcc-4.8-multilib
    E: Couldn't find any package by glob 'gcc-4.8-multilib'
    E: Couldn't find any package by regex 'gcc-4.8-multilib'

经过一番折腾，看到了这篇[文章](https://blog.csdn.net/feinifi/article/details/121793945)。简单来说就是这个包在 Ubuntu16.04 下可以正常下载，那么增加这个办的源即可。在 `/etc/apt/sources.list` 中添加如下内容：

    deb http://dk.archive.ubuntu.com/ubuntu/ xenial main
    deb http://dk.archive.ubuntu.com/ubuntu/ xenial universe

切记，需要更新

    sudo apt-get update

然后再次启动 qemu 依旧报错（此时已经过去一天了🥲）

    $ make
    + ld obj/kern/kernel
    ld: warning: section `.bss' type changed to PROGBITS
    ld: obj/kern/printfmt.o: in function `printnum':
    lib/printfmt.c:41: undefined reference to `__udivdi3'
    ld: lib/printfmt.c:49: undefined reference to `__umoddi3'
    make: *** [kern/Makefrag:71: obj/kern/kernel] Error 1

经过分析，发现 gcc 版本没有修改

    $ gcc --version
    gcc (Ubuntu 8.4.0-3ubuntu2) 8.4.0
    Copyright (C) 2018 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

于是将 gcc 版本改为 4.8 。删除原来的软连接，增加指向 4.8 版本的 软连接。查看版本更新成功。

    $ sudo rm /usr/bin/gcc
    $ sudo ln -s /usr/bin/gcc-4.8 /usr/bin/gcc
    $ gcc --version
    gcc (Ubuntu 4.8.5-4ubuntu2) 4.8.5
    Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

再次编译，没有问题了！

    $ make
    + ld obj/kern/kernel
    ld: warning: section `.bss' type changed to PROGBITS
    + as boot/boot.S
    + cc -Os boot/main.c
    + ld boot/boot
    boot block is 380 bytes (max 510)
    + mk obj/kern/kernel.img

    $ sudo make qemu

至此，环境配置完成：

![20220417223135](https://cdn.jsdelivr.net/gh/weijiew/pic/images/20220417223135.png)

接下来继续阅读 lab1 ：https://pdos.csail.mit.edu/6.828/2018/labs/lab1/ 

使用 `make grade` 来测试，验证程序是否正确。

## Part 1: PC Bootstrap

介绍 x86 汇编语言和 PC 引导过程，熟悉 QEMU 和 QEMU/GDB 调试。不用写代码但是需要回答问题。

### Exercise 1.

熟悉汇编语言。

### The PC's Physical Address Space

`make qemu` 和 `make qemu-nox` 都是用来启动 qemu ，区别是后者不带图形界面。

    +------------------+  <- 0xFFFFFFFF (4GB)
    |      32-bit      |
    |  memory mapped   |
    |     devices      |
    |                  |
    /\/\/\/\/\/\/\/\/\/\

    /\/\/\/\/\/\/\/\/\/\
    |                  |
    |      Unused      |
    |                  |
    +------------------+  <- depends on amount of RAM
    |                  |
    |                  |
    | Extended Memory  |
    |                  |
    |                  |
    +------------------+  <- 0x00100000 (1MB)
    |     BIOS ROM     |    BIOS 基本的输入输出
    +------------------+  <- 0x000F0000 (960KB)
    |  16-bit devices, |
    |  expansion ROMs  |    
    +------------------+  <- 0x000C0000 (768KB)
    |   VGA Display    |    用于视频显示
    +------------------+  <- 0x000A0000 (640KB)
    |                  |
    |    Low Memory    |  早期 PC 唯一可以访问的区域
    |                  |  实际上早期 PC 一般内存大小为 16KB, 32KB, 或 64KB
    +------------------+  <- 0x00000000


早期的 PC 是 16bit ，例如 8088 处理器，只能处理 1MB 的物理内存。所以物理空间从 0x00000000 开始到 0x000FFFFF 结束，并非是 0xFFFFFFFF 结束。

1. 其中前 640KB 是低内存，这是早期 PC 唯一可以随机访问的区域，此外早期 PC 的内存可以设置为 16KB，32KB 或 64KB 。

2. 从 0x000A0000 到 0x000FFFFF 这片内存区域留给硬件使用，例如视频显示的缓冲区，Basic Input/Output System (BIOS) 。起初这篇区域是用 ROM 来实现的，也就是只能读不能写，目前是用 flash 来实现，读写均可。此外 BIOS 负责初始化，完成后会将 OS 加载到内存中，然后将控制权交给 OS 。

随着时代的发展，PC 开始支持 4GB 内存，所以地址空间扩展到了 0xFFFFFFFF 。但是为了兼容已有的软件，保留了 0 - 1MB 之间的内存布局。0x000A0000 到 0x00100000 这区域看起来像是一个洞，前 640kb 是传统内存，剩余的部分是扩展内存。此外在 32 位下，PC 顶端的一些空间保留给 BIOS ，方便 32 位 PCI 设备使用。但是支持的内存空间已经超过了 4GB 的物理内存，也就是物理内存可以扩展到 0xFFFFFFFF 之上。但是依旧为了兼容 32 位设备的映射，在 32 位高地址部分留给 BIOS 的这片内存区域依旧保留，看起来像第二个洞。本实验中， JOS 只使用了前 256MB，可以假设只有 32 位的物理内存。

* 为什么 16 位 PC 的寻址空间是 1MB ？

以 8086 CPU 为例，数据线是 16 位($2^{16} = 64KB$)的，而地址线是 20 位( $2^{20} = 1MB$)。数据线决定了一次能获取的数据量，所以一次只能取 64KB，地址线决定了可寻址空间大小，所以寻址空间是 1MB 。这也解释了实模式下为什么段长是 64KB 。

* 新的问题产生了，寄存器都是 16 位的，怎么表示 20 位的地址？

既然一个寄存器无法表示那么就用两个寄存器来表示，也就是分段。将 1MB 的空间在逻辑上以 64KB 为单位切分，段长就是 64KB 。地址由段基地址和段内偏移两部分组成，其中段基址左移四位，加上段内偏移。

### The ROM BIOS

这一部分将会使用 qemu 的 debug 工具来研究计算机启动。

![20220504195748](https://cdn.jsdelivr.net/gh/weijiew/pic/images/20220504195748.png)

可以用 tmux 开两个窗口，一个窗口输入 `make qemu-nox-gdb` 另一个窗口输入 `make gdb` 摘取其中一行输入信息：

    [f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b

PC 从 0x000ffff0 开始执行，第一条要执行的指令是 jmp，跳转到分段地址 CS=0xf000 和 IP=0xe05b 。

起初因特尔是这样设计的，而 BIOS 处于 0x000f0000 和 0x000fffff 之间。这样设计确保了 PC 启动或重启都能获得机器的控制权。

QEMU 自带 BIOS 并且会将其放置在模拟的物理地址空间的位置上，当处理器复位时，模拟的处理器进入实模式，将 CS 设置为 0xf000，IP 设置为 0xfff0 。然后就在 CS:IP 段处开始执行。

分段地址 0xf000:ffff0 如何变成物理地址？这里面有一个公式：

    address = 16 * segment + offset

例如：

    16 * 0xf000 + 0xfff0   # in hex multiplication by 16 is
    = 0xf0000 + 0xfff0     # easy--just append a 0.
    = 0xffff0 

0xffff0 是 BIOS 结束前的16个字节（0x100000）。如果继续向后执行， 16 字节 BIOS 就结束了，这么小的空间能干什么？

### Exercise 2.

使用 gdb 的 si 指令搞清楚 BIOS 的大致情况，不需要搞清楚所有细节。

使用 si 逐行查看指令：

    [f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b  # 跳转到 `$0xfe05b` 处
    [f000:e05b]    0xfe05b: cmpl   $0x0,%cs:0x6ac8  # 若 0x6ac8 处的值为零则跳转
    [f000:e062]    0xfe062: jne    0xfd2e1
    [f000:e066]    0xfe066: xor    %dx,%dx          # 将 dx 寄存器清零
    [f000:e068]    0xfe068: mov    %dx,%ss          # 将 ss 寄存器清零
    [f000:e06a]    0xfe06a: mov    $0x7000,%esp     # esp = 0x7000 esp 始终指向栈顶
    [f000:e070]    0xfe070: mov    $0xf34c2,%edx    # edx = 0xf34c2 
    [f000:e076]    0xfe076: jmp    0xfd15c          # 跳转到 0xfd15c
    [f000:d15c]    0xfd15c: mov    %eax,%ecx        # ecx = eax
    [f000:d15f]    0xfd15f: cli                     # 关闭硬件中断
    [f000:d160]    0xfd160: cld                     # 设置了方向标志，表示后续操作的内存变化
    [f000:d161]    0xfd161: mov    $0x8f,%eax       # eax = 0x8f  接下来的三条指令用于关闭不可屏蔽中断
    [f000:d167]    0xfd167: out    %al,$0x70        # 0x70 和 0x71 是用于操作 CMOS 的端口
    [f000:d169]    0xfd169: in     $0x71,%al        # 从CMOS读取选择的寄存器
    [f000:d16b]    0xfd16b: in     $0x92,%al        # 读取系统控制端口A
    [f000:d16d]    0xfd16d: or     $0x2,%al         
    [f000:d16f]    0xfd16f: out    %al,$0x92        # 启动 A20
    [f000:d171]    0xfd171: lidtw  %cs:0x6ab8       # 加载到 IDT 表
    [f000:d177]    0xfd177: lgdtw  %cs:0x6a74       # 加载到 GDT 表
    [f000:d17d]    0xfd17d: mov    %cr0,%eax        # eax = cr0
    [f000:d180]    0xfd180: or     $0x1,%eax        # 
    [f000:d184]    0xfd184: mov    %eax,%cr0        # 打开保护模式
    [f000:d187]    0xfd187: ljmpl  $0x8,$0xfd18f    # 通过 ljmp 进入保护模式
    => 0xfd18f:     mov    $0x10,%eax               # 设置段寄存器
    => 0xfd194:     mov    %eax,%ds
    => 0xfd196:     mov    %eax,%es

当 BIOS 启动的时候会先设置中断描述表，然后初始化各种硬件，例如 VGA 。

当初始化 PCI 总线和 BIOS 知晓的所有重要设备后，将会寻找一个可启动的设备，如软盘、硬盘或CD-ROM。

最终，当找到一个可启动的磁盘时，BIOS 从磁盘上读取 boot loader 并将控制权转移给它。

## Part 2: The Boot Loader

磁盘是由扇区组成，一个扇区为 512 B。磁盘的第一个扇区称为 boot sector ，这里面存放着 boot loader 。

BIOS 将 512B 的 boot sector 从磁盘加载到内存 0x7c00 到 0x7dff 之间。然后使用 jmp 指令设置 CS:IP 为 0000:7c00 最后将控制权传递给引导装载程序。在 6.828 中使用传统的硬盘启动机制，也就是 boot loader 不能超过 512B 。

boot loader 由汇编语言 `boot/boot.S` 和一个 C 语言文件 `boot/main.c` 组成。需要搞明白这两个文件的内容。

Boot Loader 负责两个功能：

1. boot loader 从实模式切换到 32 位的保护模式，因为只有在保护模式下软件才能访问超过 1MB 的物理内存。此外在保护模式下，段偏移量就变为了 32 而非 16 。

2. 其次，Boot Loader 通过 x86 的特殊 I/O 指令直接访问 IDE 磁盘设备寄存器，从硬盘上读取内核。

理解了 Boot Loader 的源代码后，看看 `obj/boot/boot.asm` 文件。这个文件是 GNUmakefile 在编译 Boot Loader 后创建的 Boot Loader 的反汇编。这个反汇编文件使我们很容易看到 Boot Loader 的所有代码在物理内存中的位置，也使我们更容易在 GDB 中跟踪 Boot Loader 发生了什么。同样的，`obj/kern/kernel.asm` 包含了 JOS 内核的反汇编，这对调试很有用。

在 gdb 中使用 b *0x7c00 在该地址处设置断点，然后使用 c 或 si 继续执行。c 将会跳转到下一个断点处，而 si 跳转到下一条指令，si N 则一次跳转 N 条指令。

使用 `x/Ni ADDR` 来打印地址中存储的内容。其中 N 是要反汇编的连续指令的数量，ADDR 是开始反汇编的内存地址。

### Exercise 3. 

阅读 [lab tools guide](https://pdos.csail.mit.edu/6.828/2018/labguide.html)，即使你已经很熟悉了，最好看看。

在 0x7c00 设置一个断点，启动扇区将会加载到此处。跟踪 `boot/boot.S` 并使用 `obj/boot/boot.asm` 来定位当前执行位置。使用 GDB 的 x/i 命令来反汇编 Boot Loader 中的指令序列并和 `obj/boot/boot.asm` 比较。

* 阅读 `obj/boot/boot.asm` 下面是一些总结：

在汇编中 . 开头的是汇编器指令，功能是告诉汇编器如何做，而不是做什么。汇编器指令并不会直接翻译为机器码，汇编指令会直接翻译为机器码。首先设置实模式的标志，进入实模式。然后关闭中断，防止执行时被打断，接下来设置字符串指针的移动方向。做了一些初始化工作，例如寄存器清零，开启 A20 数据线，为切换到 32 位做准备。处理 GDT 。

跟踪 boot/main.c 中的 bootmain() 函数，此后追踪到 readsect() 并研究对应的汇编指令，然后返回到 bootmain() 。确定从磁盘上读取内核剩余扇区的for循环的开始和结束。找出循环结束后将运行的代码，在那里设置一个断点，并继续到该断点。然后逐步完成 Boot Loader 的剩余部分。

* 回答下面的问题：

1. 在什么时候，处理器开始执行32位代码？究竟是什么原因导致从16位到32位模式的转换？

* 从 boot.S 的第 55 行开始切换为 32 位代码，切换到 32 位后会有更多的寻址空间。

2. Boot Loader 执行的最后一条指令是什么，它刚刚加载的内核的第一条指令是什么？

* 最后一条指令是 `boot/main.c` 的 `((void (*)(void)) (ELFHDR->e_entry))();` 
*  `movw $0x1234, 0x472`

3. 内核的第一条指令在哪里？

* 内核的第一条指令在 0x1000c 处，对应的源码位于 kern/entry.S 中。

4. Boot Loader如何决定它必须读取多少个扇区才能从磁盘上获取整个内核？它在哪里找到这些信息？

> 这些信息存放在 Proghdr 中。

接下来进一步研究 `boot/main.c` 中的 C 语言部分。

### Exercise 4.

建议阅读 'K&R' 5.1 到 5.5 搞清楚指针，此外弄清楚 [pointers.c](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/pointers.c) 的输出，否则后续会很痛苦。

需要了解 ELF 二进制文件才能搞清楚 `boot/main.c` 。

当编译链接一个 C 语言程序时，首先需要将 .c 文件编译为 .o 结尾的 object 文件，其中包含了相应的二进制格式的汇编指令。

链接器将所有的 .o 文件链接为单个二进制镜像，例如 `obj/kern/kernel` ，这是一个 ELF 格式的二进制文件，全称叫做 “Executable and Linkable Format” 。

此处可以简单的将 ELF 认为该文件头部带有加载信息，然后是是程序部分，每部分都是连续的代码或数据块，将指定的地址加载到内存中。Boot Loader 将其加载到内存中并开始执行。

ELF 的二进制文件头部的长度是固定的，然后是长度可变的程序头，其中包含了需要加载的程序部分。在 `inc/elf.h` 中包含了 ELF 文件头部的定义。

* .text: 程序指令.
* .rodata: 只读数据，例如由 C 编译器生成的 ASCII 字符常量。(这个只读并没有在硬件层面实现)
* .data: 数据部分，包含了程序初始化的数据，例如声明的全局变量 x = 5 。

当链接器计算一个程序的内存布局之时，它为没有初始化的程序保留了空间，例如 int x ，在内存中紧随.data之后的一个名为.bss的部分。C 默认未初始化的全局变量为零，所以 .bss 此时没有存储内容，因此链接器只记录 .bss 部分的地址和大小并将其置为零。

通过键入检查内核可执行文件中所有部分的名称、大小和链接地址的完整列表。

    $ objdump -h obj/kern/kernel

    obj/kern/kernel:     file format elf32-i386

    Sections:
    Idx Name          Size      VMA       LMA       File off  Algn
    0 .text         00001917  f0100000  00100000  00001000  2**4
                    CONTENTS, ALLOC, LOAD, READONLY, CODE
    1 .rodata       00000714  f0101920  00101920  00002920  2**5
                    CONTENTS, ALLOC, LOAD, READONLY, DATA
    2 .stab         00003889  f0102034  00102034  00003034  2**2
                    CONTENTS, ALLOC, LOAD, READONLY, DATA
    3 .stabstr      000018af  f01058bd  001058bd  000068bd  2**0
                    CONTENTS, ALLOC, LOAD, READONLY, DATA
    4 .data         0000a300  f0108000  00108000  00009000  2**12
                    CONTENTS, ALLOC, LOAD, DATA
    5 .bss          00000648  f0112300  00112300  00013300  2**5
                    CONTENTS, ALLOC, LOAD, DATA
    6 .comment      00000023  00000000  00000000  00013948  2**0
                    CONTENTS, READONLY

VMA 是逻辑地址，LMA 是加载到内存中的物理地址。通常这两个地址是相同的。

boot loader 根据 ELF 文件的头部决定加载哪些部分。程序头部指定了哪些信息需要加载及其地址。可以通过下面的命令来查看程序头部。

    athena% objdump -x obj/kern/kernel

程序头部已经在 "Program Headers" 下列出，ELF 对象的区域需要加载到内存中然后被标记为 "LOAD"。

每个程序头的其他信息也被给出，如虚拟地址（"vaddr"），物理地址（"paddr"），以及加载区域的大小（"memsz "和 "filesz"）。

回到 `boot/main.c` 每一个程序的 `ph->p_pa` 字段包含了段的物理地址。此处是一个真正的物理地址，尽管 ELF 对这个描述不清晰。

BIOS 将 boot sector 加载到内存中并从 0x7c00 处开始，这是 boot sector 的加载地址。boot sector 从这里开始执行。这也是 boot sector 执行的地方，所以这也是它的链接地址。

在 `boot/Makefrag` 中通过 -Ttext 0x7C00 设置了启动地址。

### Exercise 5.

再次追踪 Boot Loader 的前几条指令，找出第一条指令，如果把 Boot Loader 的链接地址弄错了，就会 "中断 "或做错事。然后把`boot/Makefrag` 中的链接地址改成错误的，运行make clean，用make重新编译实验室，并再次追踪到boot loader，看看会发生什么。不要忘了把链接地址改回来，然后再做一次清理。

修改 `boot/Makefrag` 中的 `-Ttext 0x7C00` ，查看结果，例如将 其改为 `-Ttext 0x0C00` 。起初依旧加载到 0x7c00 处，但是跳转的时候出现问题。也就是最初的指令并不依赖地址，跳转的时候依赖。

![20220505143831](https://cdn.jsdelivr.net/gh/weijiew/pic/images/20220505143831.png)

回头看内核加载和链接的地址，和 Boot Loader 不同的是，这两个地址并不相同。内核告诉 Boot Loader 在一个低的地址（1 兆字节）将其加载到内存中，但它希望从一个高的地址执行。我们将在下一节中深入探讨如何使这一工作。

此外 ELF 还有很多重要的信息。例如 e_entry 是程序 entry point 的地址。可以通过如下命令查看：

    $ objdump -f obj/kern/kernel

    obj/kern/kernel:     file format elf32-i386
    architecture: i386, flags 0x00000112:
    EXEC_P, HAS_SYMS, D_PAGED
    start address 0x0010000c

kernel 是从 0x0010000c 处开始执行。

此时应当理解 `boot/main.c` 中的 ELF loader 。它将内核的每个部分从磁盘上读到内存中的该部分的加载地址，然后跳转到内核的入口点。

### Exercise 6.

可以使用 GDB 的 x 命令来查看内存。此处知晓 `x/Nx ADDR` 就够用了，在ADDR处打印N个字的内存。

重新打开 gdb 检测，在 BIOS 进入 Boot Loader 时检查 0x00100000 处的 8 个内存字，然后在 Boot Loader 进入内核时再次检查。为什么它们会不同？在第二个断点处有什么？(你不需要用 QEMU 来回答这个问题，只需要思考一下。)

> 不同是因为内核加载进来了，内核指令。

## Part 3: The Kernel

最初先执行汇编，然后为 C 语言执行做一些准备。使用虚拟内存来解决位置依赖的问题。

Boot Loader 的虚拟地址和物理地址是相同的，但是内核的虚拟地址和物理地址是不同的，更为复杂，链接和加载地址都在 `kern/kernel.ld` 的顶部。

OS 内核一般在比较高的虚拟地址上链接和运行，例如0xf0100000，而地址空间的低部分留给了用户程序使用。

许多机器在地址0xf0100000处没有任何物理内存，所以我们不能指望能在那里存储内核。相反，我们将使用处理器的内存管理硬件将虚拟地址 0xf0100000（内核代码期望运行的链接地址）映射到物理地址0x00100000（引导加载器将内核加载到物理内存的地方）。这样，尽管内核的虚拟地址足够高，可以为用户进程留下足够的地址空间，但它将被加载到物理内存中，位于PC的RAM的1MB处，就在BIOS ROM上方。这种方法要求PC至少有几兆字节的物理内存（这样物理地址0x00100000才行），但这可能是1990年以后制造的任何PC的真实情况。

事实上，在下一个实验中，我们将把PC的整个底部256MB的物理地址空间，从物理地址0x00000000到0x0fffffff，分别映射到虚拟地址0xf0000000到0xffffffff。现在你应该明白为什么JOS只能使用前256MB的物理内存了。

现在，我们只需映射前 4MB 的物理内存，这就足以让我们开始运行。我们使用`kern/entrypgdir.c`中手工编写的、静态初始化的页目录和页表来做这件事。现在，你不需要了解这个工作的细节，只需要了解它的效果。在`kern/entry.S`设置 CR0_PG 标志之前，内存引用被视为物理地址（严格来说，它们是线性地址，但`boot/boot.S`设置了从线性地址到物理地址的映射，我们永远不会改变）。一旦CR0_PG被设置，内存引用就是虚拟地址，被虚拟内存硬件翻译成物理地址。 entry_pgdir 将 0xf0000000 到 0xf0400000 范围内的虚拟地址翻译成物理地址0x00000000到0x00400000，以及虚拟地址0x00000000到0x00400000到物理地址0x00000000到0x00400000。任何不在这两个范围内的虚拟地址都会引起硬件异常，由于我们还没有设置中断处理，这将导致QEMU转储机器状态并退出（如果你没有使用6.828补丁版本的QEMU，则会无休止地重新启动）。

### Exercise 7.

使用QEMU和GDB追踪到JOS的内核，在 `movl %eax, %cr0` 处停止。检查0x00100000和0xf0100000处的内存。现在，使用stepi GDB命令对该指令进行单步操作。再次，检查0x00100000和0xf0100000处的内存。确保你明白刚刚发生了什么。

在新的映射建立后的第一条指令是什么，如果映射没有建立，它将不能正常工作？把`kern/entry.S`中的`movl %eax, %cr0`注释，追踪到它，看看你是否正确。

![20220505155904](https://cdn.jsdelivr.net/gh/weijiew/pic/images/20220505155904.png)

在 0x00100000 处打断点，比较两个地址中存储的数据后发现不一样。

![20220505160121](https://cdn.jsdelivr.net/gh/weijiew/pic/images/20220505160121.png)

> 然后执行几条指令，执行完 `mov    %eax,%cr0` 后发现两个地址中存储的数据一致。说明此时启用了页表明完成了地址映射。

大多数人认为 printf() 这样的函数是理所当然的，有时甚至认为它们是C语言的 "原语"。但是在操作系统的内核中，我们必须自己实现所有的I/O。

### Formatted Printing to the Console

阅读 `kern/printf.c`、`lib/printfmt.c` 和 `kern/console.c`，并确保你理解它们之间的关系。在后面的实验中会清楚为什么 `printfmt.c` 位于单独的 lib 目录中。

`kern/printf.c` 中的 `cprintf()` 函数调用了 `vcprintf()` 函数，该函数又调用了 `lib/printfmt.c` 中的 `vprintfmt()` 函数。

	va_start(ap, fmt);
	cnt = vcprintf(fmt, ap);
	va_end(ap);

接下来研究 `cprintf()` 函数，函数签名是 `int cprintf(const char *fmt, ...)` 其中 ... 表示可变参数。

然后是 [va_start](https://en.cppreference.com/w/c/variadic/va_start) ，简单来说，就是将可变参数放置到 ap 中。

然后调用 vcprintf 函数，将得到的参数 ap 传进去，最后调用 `va_end` 释放参数列表。此外，这部分还涉及到了 va_arg ，后面会用到，例如 `va_arg(*ap, int)` 表示用 int 来解析 ap 。


其中 putch 函数作为参数传入，而 putch 函数调用了 cputchar 函数，该函数再次调用了 cons_putc 函数，根据注释可知该函数负责将字符输出到终端。根据调用关系，可以简单的认为 putch 实现了将数据打印到终端的功能，至于实现细节后续再研究。

接下来回头研究 `lib/printfmt.c` 中的 `vprintfmt()` 函数，因为 `kern/printf.c` 中的 `cprintf()` 最终调用了该函数。

```vprintfmt(void (*putch)(int, void*), void *putdat, const char *fmt, va_list ap)```

该函数的函数签名中共四个参数，下面是四个参数的解释：

1. 第一个参数是 putch 函数，之前已经解释过了，负责实现打印到终端。
2. 第二个参数 putdat 初始值为零，目前还不知道负责什么功能。
3. 第三个参数 fmt 是输入的字符串。
4. 第四个参数 ap 是 va_list 类型，这个参数实现了可变参数，也就是可以处理不同数量的参数。

cons_putc 分别调用了 serial_putc，lpt_putc 和 cga_putc 三个函数。

### Exercise 8.

我们省略了一小段代码--使用"%o "形式的模式打印八进制数字所需的代码。找到并填入这个代码片段。

		case 'u':
			num = getuint(&ap, lflag);
			base = 10;
			goto number;

		// (unsigned) octal
		case 'o':
			// Replace this with your code.
			num = getuint(&ap, lflag);
			base = 8;
			goto number;
			break;


回答以下问题: 

1. 解释一下 printf.c 和 console.c 之间的接口。具体来说，console.c输出了什么函数？这个函数是如何被printf.c使用的？

printf.c 中的 putch() 函数调用了 console.c 中的 cputchar() 函数，该函数再次调用了 cons_putc() 函数，这个函数负责将数据打印到终端。

2. 从 console.c 中解释如下：

```c
    1      if (crt_pos >= CRT_SIZE) {
    2              int i;
    3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
    4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
    5                      crt_buf[i] = 0x0700 | ' ';
    6              crt_pos -= CRT_COLS;
    7      }
```

这段函数源自 `console.c` 文件中的 `cga_putc()` 函数，该函数会被 `cons_putc()` 函数所调用。根据注释可知，`cons_putc()` 负责将数据打印到终端，那么 `cga_putc()` 则是负责具体实现如何打印到终端。

然后研究 `void* memmove( void* dest, const void* src, std::size_t count );` 从 src 处复制 count 大小的数据到 dest 上。最后分析 `memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));` 其实就是将当前屏幕上的数据向上移动一行。

最后的 for 循环就是将最新写入的部分(crt_pos >= CRT_SIZE)打印出来。

3. 对于下面的问题，你可能希望参考第2讲的注释。这些笔记涵盖了GCC在X86上的调用惯例。

* 逐步跟踪以下代码的执行。

    int x = 1, y = 3, z = 4;
    cprintf("x %d, y %x, z %d\n", x, y, z);

* 在对cprintf()的调用中，fmt指向什么？ap指的是什么？

列出对cons_putc、va_arg和vcprintf的每个调用（按执行顺序）。对于cons_putc，也要列出其参数。对于va_arg，列出调用前后ap所指向的内容。对于vcprintf，列出其两个参数的值。


> 首先研究 fmt ：将上述代码写入 `kern/monitor.c` 中的 `mon_backtrace()` 函数中，然后开始调试。

> 使用 `b mon_backtrace` 打断点，使用 c 执行到这一步，然后使用 `s` 进入 `cprintf()` 函数中，多执行两步后发现 `fmt = "x %d, y %x, z %d\n"` 。

![20220505225217](https://cdn.jsdelivr.net/gh/weijiew/pic/images/20220505225217.png)

> 接下来研究 ap ，经过数次调用 va_arg ，ap 从 1 3 4 变为 3 4 变为 4 再为空。

4. 运行以下代码。

    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);


输出是什么？解释一下这个输出是如何按照前面练习的方式一步步得出的。这里有一个ASCII表，将字节映射到字符。这个输出取决于x86是小端的事实。如果x86是big-endian的，你要把i设置成什么样子才能产生同样的输出？你是否需要将57616改为不同的值？

> 输出 "He110 World" 其中 5760 的二进制形式是 e110 。至于 0x00646c72 为什么显示为 rld ，首先要搞清楚大小端。

> 我认为应当从读取顺序的角度来看，通常人类的读取习惯是从高位向地位阅读，也就是从左向右。但是对于计算机而言优先处理地位显然效率更高，所以数字的地位存储在低地址部分，数据的高位存储在高地址部分，也就是小端。而大端反之，地位存储在高地址部分，小端存储在低地址部分。

    低地址  =====> 高地址
    小端：  72  6c  64 00
    大端：  00  64  6c 72

> 查 ASCII 表可知 0x72 0x6c 0x64 0x00 分别表示 'r' 'l' 'd' '\0' 。如果在大端的系统上输出相同的内容需要改为 0x726c6400 。

5. 在下面的代码中，'y='后面要打印什么？(注意：答案不是一个具体的数值。)为什么会出现这种情况？

    cprintf("x=%d y=%d", 3);

> 输出 x=3 y=-267321544 第一个输出 3 是因为参数就是 3 而第二个输出则是读取了相邻地址中的内容。

6. 假设GCC改变了它的调用惯例，使它按声明顺序把参数推到堆栈上，这样最后一个参数就被推到了。你要如何改变cprintf或它的接口，使它仍然有可能传递可变数量的参数？

> `cprintf` 函数的两个参数交换顺序即可。

7. 挑战，终端打印出彩色文本。

> 写到这里已经花了很长时间了，先跳过。

### The Stack

### Exercise 9. 

确定内核在哪里初始化它的堆栈，以及它的堆栈在内存中的确切位置。内核是如何为其堆栈保留空间的？堆栈指针被初始化为指向这个保留区域的哪个 "末端"？

esp 寄存器指向栈顶，ebp 指向栈底。在 32 位模式下，堆栈只能容纳 32 位的值，esp 总能被 4 整除。当调用 C 函数之时，首先将 esp 中的值复制到 ebp 中，然后 esp 向下生长开辟空间。可以通过 ebp 指针链来回溯堆栈进而确定函数的调用关系。这个功能很有用，例如一个函数断言失败或 panic ，那么可以通过回溯堆栈来确定出现问题函数。

> 1. 在 `kernel/entry.S` 中 77 行的 `movl	$(bootstacktop),%esp` 指令开始初始化栈，该指令设置了栈帧。
> 2. 根据 `.space		KSTKSIZE` 来确定栈大小，KSTKSIZE 在 `inc/memlayout.h` 定义大小为 8*PGSIZE ，而 PGSIZE 为 4096 字节。
> 3. 根据相应的汇编文件 `obj/kern/kernel.asm` 第 58 行可知，栈帧的虚拟地址为 0xf0110000。因为栈是向下生长，所以根据 KSTKSIZE 可以确定栈的末端。

### Exercise 10. 

为了熟悉x86上的C语言调用习惯，在`obj/kern/kernel.asm`中找到`test_backtrace`函数的地址，在那里设置一个断点，并检查内核启动后每次调用该函数时发生的情况。test_backtrace的每个递归嵌套层在堆栈上推多少个32位字，这些字是什么？

实现 `kern/monitor.c` 文件中的 `mon_backtrace()` 函数。`inc/x86.h`中的`read_ebp()`函数很有用。

回溯函数应该以下列格式显示函数调用框架的清单。

        Stack backtrace:
        ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
        ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
        ...

每一行都包含一个 ebp 、 eip 和 args 。ebp 值表示该函数所使用的进入堆栈的基本指针：即刚进入函数后堆栈指针的位置，函数序言代码设置了基本指针。 eip 值是该函数的返回指令指针：当函数返回时，控制将返回到该指令地址。返回指令指针通常指向调用指令之后的指令（为什么呢）。最后，在args后面列出的五个十六进制值是有关函数的前五个参数，这些参数在函数被调用之前会被推到堆栈中。当然，如果函数被调用时的参数少于5个，那么这5个值就不会全部有用。(为什么回溯代码不能检测到实际有多少个参数？如何才能解决这个限制呢？）

打印的第一行反映当前执行的函数，即 mon_backtrace 本身，第二行反映调用  mon_backtrace 的函数，第三行反映调用该函数的函数，以此类推。应该打印所有未完成的堆栈帧。通过研究 `kern/entry.S` ，你会发现有一种简单的方法可以告诉你何时停止。

以下是你在《K&R》第五章中读到的几个具体要点，值得你在下面的练习和今后的实验中记住。

1. 如果int *p = (int*)100，那么(int)p + 1和(int)(p + 1)是不同的数字：第一个是101，但第二个是104。当把一个整数加到一个指针上时，就像第二种情况一样，这个整数隐含地乘以指针所指向的对象的大小。
2. p[i]被定义为与*(p+i)相同，指的是p所指向的内存中的第i个对象。当对象大于一个字节时，上面的加法规则有助于这个定义发挥作用。
3. &p[i]与(p+i)相同，产生p所指向的内存中的第i个对象的地址。

尽管大多数C语言程序不需要在指针和整数之间进行转换，但操作系统经常需要。每当你看到涉及内存地址的加法时，要问自己这到底是整数加法还是指针加法，并确保被加的值被适当地乘以。

> ebp 的初始值为 0 ，可以判断是否为零来停止循环。

```cpp
	uint32_t ebp, eip;
	cprintf("Stack backtrace:\n");
	for (ebp = read_ebp(); ebp != 0; ebp = *((uint32_t *)ebp)) {
		eip = *((uint32_t *)ebp + 1);
		cprintf(" ebp %08x eip %08x args %08x %08x %08x %08x %08x\n",
		ebp, eip, *((uint32_t *)ebp + 2),
		*((uint32_t *)ebp + 3), *((uint32_t *)ebp + 4),
		*((uint32_t *)ebp + 5), *((uint32_t *)ebp + 6));
	}
```

### Exercise 11. 

使用 make grade 验证。

如果使用 read_ebp()，中间变量可能会被优化掉，进而导致跟踪堆栈信息时看不到完整的堆栈信息。

在这一点上，你的回溯函数应该给你堆栈上导致 mon_backtrace() 被执行的函数调用者的地址。然而，在实践中，你经常想知道这些地址所对应的函数名称。例如，你可能想知道哪些函数可能包含一个导致内核崩溃的错误。

为了帮助你实现这一功能，我们提供了函数 debuginfo_eip()，它在符号表中查找eip并返回该地址的调试信息。这个函数在kern/kdebug.c中定义。

在这一点上，你的回溯函数应该给你堆栈上导致 mon_backtrace() 被执行的函数调用者的地址。然而，在实践中，你经常想知道这些地址所对应的函数名称。例如，你可能想知道哪些函数可能包含一个导致内核崩溃的错误。

为了帮助你实现这一功能，我们提供了函数 debuginfo_eip() ，它在符号表中查找 eip 并返回该地址的调试信息。这个函数在kern/kdebug.c中定义。

通过 `debuginfo_eip(addr, info)` 来查看 eip 中更多的信息，具体功能是将地址 addr 处的内容填入 info 中。如果找到信息就返回零，如果没有查到信息就返回负数，

### Exercise 12. 

修改你的堆栈回溯函数，为每个 eip 显示函数名、源文件名和与该 eip 对应的行号。

在 debuginfo_eip 中，__STAB_* 来自哪里？这个问题有一个很长的答案；为了帮助你发现答案，这里有一些你可能想做的事情。

* 在kern/kernel.ld文件中查找__STAB_*。
* 运行 objdump -h obj/kern/kernel
* 运行objdump -G obj/kern/kernel
* 运行gcc -pipe -nostdinc -O2 -fno-builtin -I. -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c，并查看init.s。
* 看看bootloader是否在内存中加载符号表作为加载内核二进制的一部分
* 完成debuginfo_eip的实现，插入对stab_binsearch的调用，以找到地址的行号。


	stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
	info->eip_line = lline > rline ? -1 : stabs[rline].n_desc;

在内核监控器中添加一个回溯命令，并扩展你的mon_backtrace的实现，以调用debuginfo_eip并为每个堆栈帧打印一行。

        Stack backtrace:
        ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
        ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061

    K> backtrace
    Stack backtrace:
    ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
            kern/monitor.c:143: monitor+106
    ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
            kern/init.c:49: i386_init+59
    ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
            kern/entry.S:70: <unknown>+0
    K> 

每一行都给出了stack frame 的 eip 文件名和在该文件中的行数，然后是函数的名称和 eip 与函数第一条指令的偏移量（例如，monitor+106表示返回eip比monitor的开头多106字节）。

请确保将文件和函数名单独打印在一行，以避免混淆 grading 脚本。

提示：printf格式的字符串提供了一种简单但不明显的方法来打印非空尾的字符串，如STABS表中的字符串。 printf("%.*s", length, string)最多能打印出字符串的长度字符。看一下printf手册，了解为什么这样做。

你可能会发现在回溯中缺少一些函数。例如，你可能会看到对monitor()的调用，但没有对runcmd()的调用。这是因为编译器对一些函数的调用进行了内联。其他优化可能导致你看到意外的行数。如果你把GNUMakefile中的-O2去掉，回溯可能会更有意义（但你的内核会运行得更慢）。


```cpp
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	uint32_t ebp, eip;
	struct Eipdebuginfo info;
	cprintf("Stack backtrace:\n");
	for (ebp = read_ebp(); ebp != 0; ebp = *((uint32_t *)ebp)) {
		eip = *((uint32_t *)ebp + 1);
		cprintf(" ebp %08x eip %08x args %08x %08x %08x %08x %08x\n",
		ebp, eip, *((uint32_t *)ebp + 2),
		*((uint32_t *)ebp + 3), *((uint32_t *)ebp + 4),
		*((uint32_t *)ebp + 5), *((uint32_t *)ebp + 6));

		if (!debuginfo_eip(eip,&info)) {
			cprintf("%s:%d: %.*s+%d\n",
			info.eip_file, info.eip_line,info.eip_fn_namelen,
			info.eip_fn_name, eip - info.eip_fn_addr);		
		}
	}
	return 0;
}
```

