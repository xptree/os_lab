#操作系统 Lab1


##练习1: 理解通过makefile生成执行文件的过程
### ucore.img是如何一步一步生成的
make的默认目标为makefile中TARGETS中定义内容，另外makefile在开始阶段检查gccprefix以及qemu，同时定义了一系列变量，包括文件后缀，目录，编译选项，连接选项等内容，并使用include引入了tools/function.mk定义的内容。
使用make "V="命令查看make执行的命令，生成ucore.img的命令包括:

* 使用gcc编译器由*.c文件生成obj文件
	```cc kern/init/init.c```,使用命令:
	
	```
	gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
	```
参数说明:

	```
	-Ikern/init/ -Ilibs : 将目录kern/init/等添加到头文件的搜索路径
	-fno-builtin : 禁用不是两个下划线开头的内建函数
	-Wall : 提示所有warning
	-ggdb : 为gdb生成调试信息
	-m32 : 32位编译
	-gstabs : 产生stabs格式的调式信息
	-nostdinc : 不在标准系统目录中搜索头文件，即只搜索-I指定的目录
	-fno-stack-protector : 禁用堆栈保护
	-c : 指定源文件名
	-o : 指定输出文件名
	```

	以上编译选项均在makefile中定义的CFLAGS变量中，相似指令包括:

	```
	cc kern/libs/readline.c
	cc kern/libs/stdio.c
	cc kern/debug/kdebug.c
	cc kern/debug/kmonitor.c
	cc kern/debug/panic.c
	cc kern/driver/clock.c
	cc kern/driver/console.c
	cc kern/driver/intr.c
	cc kern/driver/picirq.c
	cc kern/trap/trap.c
	cc kern/trap/trapentry.S
	cc kern/trap/vectors.S
	cc kern/mm/pmm.c
	cc libs/printfmt.c
	cc libs/string.c
	cc boot/bootasm.S
	cc boot/bootmain.c
```

* 使用ld命令连接obj文件

	* ```ld bin/kernel```，连接生成的obj文件到bin/kernel，使用命令:
	```
	ld -m elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
	```
参数说明:

		```
	-m elf-i386 : 模拟器设置为elf-i386
	-nostdlib : 不使用标准库文件连接，仅搜索命令中指定的库路径
	-T tools/kernel.ld : 指定tools/kernel.ld为主连接脚本
	-o : 指定输出文件名
		```
	* ```ld bin/bootblock```，连接obj/boot/bootasm.o obj/boot/bootmain.o到obj/bootblock.o，使用命令:
	```ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o```
参数说明:

		```
	-N : 把text和data节设置为可读写，同时取消数据节的页对齐
	-e start : 使用符号start作为程序的开始执行点
	-Ttext 0x7C00 : 设置代码起始段地址为0x7C00
	```
* 使用gcc编译器由tools/sign.c生成可执行文件bin/sign
cc tools/sign.c，使用命令:

	```
	gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
	gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
	```
* 使用dd工具创建一个bin/ucore.img空文件
使用命令:

	```
	dd if=/dev/zero of=bin/ucore.img count=10000
	```

* 使用dd工具将文件bin/bootblock写入bin/ucore.img
使用命令:

	```
	dd if=bin/bootblock of=bin/ucore.img conv=notrunc
	```
参数conv=notrunc表示不截断输出文件

* 使用dd工具将文件bin/kernel写入bin/ucore.img起始的1个block后，即bootblock后
使用命令:

	```
	dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
	```
参数seek=1表示从输出文件开头跳过1个block开始写入

###什么是一个符合规范的系统
 一个被系统认为是符合规范的硬盘主引导区的特征是
	大小为512个字节，以55AA结束

## 练习2:使用qemu执行并调试lab1中的软件
### 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行

修改gdb配置文件gdbinit如下:

```		
file bin/kernel
target remote :1234
break *0x7C00
```		
使用make debug命令使qemu进入等待模式，同时gdb与qemu连接并进入调试，在qemu中输入x/i $pc查看指令地址，gdb使用si单步执行指令，得到的前十条指令依次为:

```
0xfffffff0:	ljmp	$0xf000,$0xeo5b
0x000fe05b:	cmpl	$0x0,%cs:-0x2f2c
0x000fe062:	jne	0xfc792
0x000fe066:	xor	%ax,%ax
0x000fe068:	mov	%ax,%ss
0x000fe06a:	mov	%0x7000,%esp
0x000fe070:	mov	$0xf0060,%edx
0x000fe076:	jmp	0xfc641
0x000fc641: mov %eax,%ecx
0x000fc644:	cli
```
###在初始化位置 0x7c00 设置实地址断点,测试断点正常
gdbinit不变，gdb执行continue命令，得到结果:
		
```
Continuing.
=> 0x7c00:	cli
```
断点正常

### 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较
接着使用gdb si命令单步跟踪，取6条指令结果如下:

```
0x00007c00:	cli
0x00007c01:	cld
0x00007c02:	xor	%ax,%ax
0x00007c04:	mov	%ax,%ds
0x00007c06:	mov	%ax,%es
0x00007c08:	mov	%ax,%ss
```
bootasm.S

```
# start address should be 0:7c00, in real mode, the beginning address of the running bootloader
.globl start
start:
.code16                                             
# Assemble for 16-bit mode
	cli                                            
# Disable interrupts
	cld                                             
# String operations increment

# Set up the important data segment registers (DS, ES, SS).
	xorw %ax, %ax                                   
# Segment number zero
	movw %ax, %ds                                   
# -> Data Segment
	movw %ax, %es                                   
# -> Extra Segment
	movw %ax, %ss                                   
# -> Stack Segment
```
bootblock.asm

```
# start address should be 0:7c00, in real mode, the beginning address of the running bootloader
.globl start
start:
.code16                                            
# Assemble for 16-bit mode
	cli                                             
# Disable interrupts
	7c00:	fa                   	cli    
	cld                                             
# String operations increment
	7c01:	fc                   	cld    

# Set up the important data segment registers (DS, ES, SS).
	xorw %ax, %ax                                   
# Segment number zero
	7c02:	31 c0                	xor    %eax,%eax
	movw %ax, %ds                                   
# -> Data Segment
	7c04:	8e d8                	mov    %eax,%ds
	movw %ax, %es                                   
# -> Extra Segment
	7c06:	8e c0                	mov    %eax,%es
	movw %ax, %ss                                   
# -> Stack Segment
	7c08:	8e d0                	mov    %eax,%ss
```
可以看到三者是一致的，bootasm.S没有指令地址信息
	
### 自己找一个 bootloader 或内核中的代码位置,设置断点并进行测试。
修改gdb配置文件gdbinit如下:

```
file bin/kernel
target remote :1234
break kern_init
continue
```
执行make debug得到结果```Breakpoint 1, kern_init () at kern/init/init.c:17```，可以看到程序运行到kern_init时成功中断

##练习3: 分析bootloader进入保护模式的过程
* bootloader开始运行在实模式，物理地址为0x7c00,且是16位模式
* bootloader关闭所有中断，方向标志位复位，ds，es，ss段寄存器清零
* 打开A20使之能够使用高位地址线
* 由实模式进入保护模式，使用lgdt指令把GDT描述符表的大小和起始地址存入gdt寄存器，修改寄存器CR0的最低位（orl $CR0_PE_ON, %eax）完成从实模式到保护模式的转换，使用ljmp指令跳转到32位指令模式
* 进入保护模式后，设置ds，es，fs，gs，ss段寄存器，堆栈指针，便可以进入c程序bootmain


##练习4: 分析bootloader加载ELF格式的OS的过程
###bootloader如何读取硬盘扇区的
* bootloader进入保护模式并载入c程序bootmain
* bootmain中readsect函数完成读取磁盘扇区的工作，函数传入一个指针和一个uint_32类型secno，函数将secno对应的扇区内容拷贝至指针处
* 调用waitdisk函数等待地址0x1F7中低8、7位变为0,1，准备好磁盘
* 向0x1F2输出1，表示读1个扇区，0x1F3输出secno低8位，0x1F4输出secno的8~15位，0x1F5输出secno的16~23位，0x1F6输出0xe+secno的24~27位，第四位0表示主盘，第六位1表示LBA模式，0x1F7输出0x20
* 调用waitdisk函数等待磁盘准备好
* 调用insl函数把磁盘扇区数据读到指定内存

### bootloader是如何加载ELF格式的OS
bootloader通过bootmain函数完成ELF格式OS的加载。

* 调用readseg函数从kernel头读取8个扇区得到elfher
* 判断elfher的成员变量magic是否等于ELF_MAGIC，不等则进入bad死循环
* 相等表明是符合格式的ELF文件，循环调用readseg函数加载每一个程序段
* 调用elfher的入口指针进入OS


##练习5: 实现函数调用堆栈跟踪函数
####实现过程
* 使用```read_ebp()，read_eip()```函数获得ebp，eip的值
* 循环：
	1. 输出ebp，eip的值
	2. 输出4个参数的值，其中第一个参数的地址为ebp+8，依次加4得到下一个参数的地址
	3. 更新ebp，eip，其中新的ebp的地址为ebp，新的eip的地址为ebp+4，即返回地址
	4. ebp为0时表明程序返回到了最开始初始化的函数，ebp=0为循环的退出条件

####语句和地址等的对应关系
在输出的eip中设置断点，结果如下：
	
```
kern/debug/kdebug.c:306: print_stackframe+22 对应kern/debug/kdebug.c文件中print_stackframe函数调用完read_ebp的位置
kern/debug/kmonitor.c:125: mon_backtrace+10对应kern/debug/kmonitor.c文件中mon_backtrace函数调用完print_stackframe的位置
kern/init/init.c:48: grade_backtrace2+33 对应kern/init/init.c文件中grade_backtrace2函数调用完mon_backtrace的位置
kern/init/init.c:53: grade_backtrace1+38 对应kern/init/init.c文件中grade_backtrace1函数调用完grade_backtrace2的位置
kern/init/init.c:58: grade_backtrace0+23 对应kern/init/init.c文件中pgrade_backtrace0函数调用完grade_backtrace1的位置
kern/init/init.c:63: grade_backtrace+34 对应kern/init/init.c文件中grade_backtrace函数调用完grade_backtrace0的位置
kern/init/init.c:28: kern_init+86 对应kern/init/init.c文件中pkern_init函数调用完grade_backtrace的位置
<unknow>: -- 0x00007d6e -- 此时执行到了bootmain中的内容
```
##练习6: 完善中断初始化和处理
1. 中断向量表中一个表项占多少字节?其中哪几位代表中断处理代码的入口?

	中断向量表一个表项占8个字节，其中6，7两个字节表示入口指针的高16位，0，1两个字节表示入口指针的低16位，2，3两个字节表示入口指针的段选择子

2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数```idt_init```
	
	使用SETGATE宏设置每一个idt，均使用中断门描述符，权限均为内核态权限，设置T_SYSCALL，使用陷进门描述符，权限为用户权限，最后调用lidt函数

3. 请编程完善trap.c中的中断处理函数trap
	
	使用kern/driver/clock.c中的变量ticks，每次中断时加1，达到```TICK_NUM```次后归零并执行print_ticks

