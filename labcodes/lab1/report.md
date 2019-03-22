# lab1实验报告

## 练习1

### 1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

Makefile文件中有这么一段代码：

```
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```
其中镜像文件需要依赖```bin/bootblock```和```bin/kernel```两个文件
第五行的命令生成一个全0字符文件，一共拷贝1E4次，一共512E4 Byte
第六行的命令将```bin/bootblock```拷贝到```ucore.img```的开头，且保持原有长度
第七行的命令将```bin/kernel```拷贝到```ucore.img```的```bin/bootblock```之后，由于```bin/bootblock```长度为512 Byte，恰好跳过一个block

### 2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

打开```tools/sign.c```，最后一段代码为：

```
	buf[510] = 0x55;
	buf[511] = 0xAA;
	FILE *ofp = fopen(argv[2], "wb+");
	size = fwrite(buf, 1, 512, ofp);
	if (size != 512) {
		fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
		return -1;
	}
	fclose(ofp);
	printf("build 512 bytes boot sector: '%s' success!\n", argv[2]);
	return 0;
```

所以显然要求是一共有512Byte，且最后两Byte为```0x55AA```

## 练习2：使用qemu执行并调试lab1中的软件。

### 1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

按照文档的提示，修改```tools/gdbinit```的内容：

```
set architecture i8086
target remote :1234
```

然后直接运行```make debug```，在gdb调试窗口中，使用```si```命令进行单步调试即可

### 2. 在初始化位置0x7c00设置实地址断点,测试断点正常。

修改文件```tools/gdbinit```的内容：

```
file obj/bootblock.o
set architecture i8086
target remote :1234
b *0x7c00
continue
```

即可跳转到bootloader入口处执行代码

### 3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

三者的汇编代码都相同

### 4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

选在进入保护模式的位置设置断点，修改```tools/gdbinit```的内容：

```
file obj/bootblock.o
set architecture i8086
target remote :1234
b protcseg
continue
```

即可使程序在运行到该位置时暂停

## 练习3：分析bootloader进入保护模式的过程。

**初始化寄存器**

```
    cli                                             # IF设成0，此时无视中断
    cld                                             # DF设成0，表示字符从低地址处理到高地址

    # 将段寄存器设置成0
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```

**为了访问4G内存，需要将A20打开**

```
seta20.1:
    inb $0x64, %al                                  # 读status寄存器，直到input寄存器无数据
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64, 表示要开始写数据
    outb %al, $0x64                                 

seta20.2:											# 等待input寄存器为空
    inb $0x64, %al                       
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60, 打开A20
    outb %al, $0x60                                 
```

**加载gdt并转换到保护模式**

```
    lgdt gdtdesc
    movl %cr0, %eax 								#将CR0的PE设置成1，开启保护模式
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
    
    ljmp $PROT_MODE_CSEG, $protcseg 				#通过长跳转更新CS寄存器
```

**设置段寄存器，建立堆栈，跳转到主函数**

```
		movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
	    movl $0x0, %ebp
	    movl $start, %esp
	    
	    call bootmain
```

## 练习4：分析bootloader加载ELF格式的OS的过程。

### 1. bootloader是如何读取硬盘扇区的？

在bootmain.c中，readsect函数用来读取硬盘扇区。

* 首先，一直查询，直道扇区不是忙状态
* 0x1F2地址输入需要读取的扇区数目
* 0x1F3～0x1F6输入的均为LBA的数值
* 0x1F7输入读取的命令
* 等待扇区准备好
* 从扇区中读出512Byte

### 2. bootloader是如何加载ELF格式的OS？

* 先读取elf文件头
* 验证文件头中的magic number
* 读取程序头部，即一个program header结构的数组，从中获得程序各个的信息，且读取到指定的地址处
* 跳转执行该文件

下面的C语句表示将改地址强制转换成一个没有参数的函数并跳转执行
```
  ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))()
```

## 练习5：实现函数调用堆栈跟踪函数。

### 实现函数调用堆栈跟踪函数

函数实现见```kdebug.c```的```print_stackframe()```函数，实现过程按照给定的要求来就可以，获取当前ebp和eip，输出当前栈帧内容，再获取前一个ebp和eip，直道所有栈帧都被访问。

在bash on windows+Xming环境下，运行指令```env DISPLAY=:0 sudo make qemu testdisplay```，得到输出如下：
```
Kernel executable memory footprint: 64KB
ebp:00007b38 eip:00100a4c args:00010094 00010094 00007b68 00100084
    kern/debug/kdebug.c:307: print_stackframe+21
ebp:00007b48 eip:00100e0f args:00000000 00000000 00000000 00007bb8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:00007b68 eip:00100084 args:00000000 00007b90 ffff0000 00007b94
    kern/init/init.c:48: grade_backtrace2+19
ebp:00007b88 eip:001000a6 args:00000000 ffff0000 00007bb4 00000029
    kern/init/init.c:53: grade_backtrace1+27
ebp:00007ba8 eip:001000c3 args:00000000 00100000 ffff0000 00100043
    kern/init/init.c:58: grade_backtrace0+19
ebp:00007bc8 eip:001000e4 args:00000000 00000000 00000000 00103600
    kern/init/init.c:63: grade_backtrace+26
ebp:00007be8 eip:00100050 args:00000000 00000000 00000000 00007c4f
    kern/init/init.c:28: kern_init+79
ebp:00007bf8 eip:00007d6e args:c031fcfa c08ed88e 64e4d08e fa7502a8
    <unknow>: -- 0x00007d6d --
```

输出的最后一行表示的意思是：
* 该eip指向的是bootblock.asm中的0x7d6e地址的语句，0x7d6c的语句为call语句，表示跳转到kern_init运行
* ebp的初始值在0x7c45处被设置成0x7c00，然后跳转到bootmain运行
* 在bootmain中进行了一些压栈操作，所以后面的参数就是在bootmain函数中压栈的参数

## 练习6：完善中断初始化和处理。

### 1. 中断描述符表的一个表项占多少字节？其中哪几位表示中断处理程序的入口？

一个表项占8个字节，其中0、1位和6、7位合起来表示地址，2、3位为段选择子；段选择子从GDT或者LDT中取得基址，相加后得到程序入口

### 2. 完善trap.c中的idt_init函数

按照要求，在外面声明__vectors，用来存储终端代码的offset
将idt中每一项用SETGATE来设置
调用lidt()函数

### 3. 完善trap.c中的trap函数

设置一个counter，每次累计到100 ticks时打印并清空。

## Challenge 1

### 增加syscall功能，用户态和内核态转换

修改了```trap.c```的```trap_dispatch()```函数，添加了```kdebug.c```的```print_trapStackframe()```函数
其中```print_trapStackframe()```函数实现基本和```print_stackframe()```一样，重点是转换过程：
* 调用int指令，跳转到IDT，选择中断处理服务并执行
* 然后在```trap_dispatch()```函数中，将保存的trapframe的寄存器和基质都修改
* 在内核态调用int指令时，没有保存%esp, %ss寄存器，所以要将%esp减8用于保存
* 同样的在用户态调用int指令时， 也要将%esp加8
