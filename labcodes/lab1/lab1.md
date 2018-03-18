# Lab1 report

## [练习1]

###[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)（简直反人类！

* 生成ucore.img的相关代码为
    ```
     $(UCOREIMG): $(kernel) $(bootblock)
        $(V)dd if=/dev/zero of=$@ count=10000
        $(V)dd if=$(bootblock) of=$@ conv=notrunc
    	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
    ```
* 可以看出其依赖于kernel和bootblock
* 生成bootblock的相关代码为
    ```
     $(bootblock): $(call toobj,$(bootfiles)) * $(call totarget,sign)
        @echo + ld $@
	    $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	    @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	    @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	    @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
    ```
	* bootblock依赖于bootasm.o、bootmain.o、sign
	* 生成bootasm.o,bootmain.o的相关makefile代码为
        ```
	 bootfiles = $(call listf_cc,boot) 
	 $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC), $(CFLAGS) -Os -nostdinc))
        ```
	* 实际代码由宏批量生成

	    * bootasm.o依赖于bootasm.S, 实际命令为
        ```
 gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc 
 -c boot/bootasm.S -o obj/boot/bootasm.o
        ```
            * 其中关键的参数为
            * 	-ggdb  生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore。
            *	-m32  生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位的软件。
    	    * 	-gstabs  生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息
	        * 	-nostdinc  不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的，所以所有的服务要自给自足。
	        *	-fno-stack-protector  不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核，ucore内核好像还用不到此功能。
	        * 	-Os  为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootloader的最终大小不能大于510字节。
	        * 	-I<```dir```>  添加搜索头文件的路径
 
	    * bootmain.o依赖于bootmain.c,  实际命令为
            ```
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
            ```
        	* 新出现的关键参数有
        	* 	-fno-builtin  除非用__builtin_前缀，否则不进行builtin函数的优化
    	* 生成sign工具的makefile代码为
            ```
 $(call add_files_host,tools/sign.c,sign,sign)
 $(call create_target_host,sign,sign)
            ```
    	* 实际命令为
            ```
 gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
 gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
            ```
    * 最后bootblock生成过程为：首先生成bootblock.o
```
 ld -m      elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```
    	* 其中关键的参数为
    	*	-m <emulation>  模拟为i386上的连接器
    	*	-nostdlib  不使用标准库
    	*	-N  设置代码段和数据段均可读写
    	*	-e <entry>  指定入口
	    *	-Ttext  制定代码段开始位置

    * 拷贝二进制代码bootblock.o到bootblock.out
```
 objcopy -S -O binary obj/bootblock.o obj/bootblock.out
```
    	* 其中关键的参数为
    	*	-S  移除所有符号和重定位信息
	    *	-O <bfdname>  指定输出格式
    
    * 使用sign工具处理bootblock.out，生成bootblock
```
 bin/sign obj/bootblock.out bin/bootblock
```

* 生成kernel的相关代码为
``` 
 $(kernel): tools/kernel.ld
 $(kernel): $(KOBJS)
 	@echo + ld $@
 	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
 	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
 	@$(OBJDUMP) -t $@ * $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; \
 		/^$$/d' > $(call symfile,kernel)
``` 
	* 为了生成kernel，首先需要 kernel.ld init.o readline.o stdio.o kdebug.o
	kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o
	trapentry.o vectors.o pmm.o  printfmt.o string.o
	* kernel.ld已存在
    *   生成这些.o文件的相关makefile代码为
```
 $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel, $(KCFLAGS))
```
	* 这些.o生成方式和参数均类似，仅举init.o为例，其余不赘述
	*	obj/kern/init/init.o
    	* 编译需要init.c
    	* 实际命令为
```
 gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
``` 
	* 生成kernel时，makefile的几条指令中有@前缀的都不必需, 必需的命令只有
```
 ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
 	obj/kern/init/init.o obj/kern/libs/readline.o \
 	obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
 	obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
 	obj/kern/driver/clock.o obj/kern/driver/console.o \
 	obj/kern/driver/intr.o obj/kern/driver/picirq.o \
 	obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
 	obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
 	obj/libs/printfmt.o obj/libs/string.o
```
    	* 其中新出现的关键参数为
            * -T <scriptfile>  让连接器使用指定的脚本
            * 生成一个有10000个块的文件，每个块默认512字节，用0填充
            ```
            dd if=/dev/zero of=bin/ucore.img count=10000
            ```
            * 把bootblock中的内容写到第一个块
            ```
            dd if=bin/bootblock of=bin/ucore.img conv=notrunc
            ```
            * 从第二个块开始写kernel中的内容
            ```
            dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
            ```

###[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

*   一个磁盘主引导扇区只有512字节, 且第510个和511个字节分别是0x55，0xAA。

## [练习2]

###[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。
* 将gdbinit文件内容修改为
```
set arch i8086
target remote :1234
define hook-stop
x/i $pc
end
```
* 执行make debug可在gdb中从第一条指令开始单步调试并看到汇编结果

###[练习2.2~2.3] 在初始化位置0x7c00 设置实地址断点,测试断点正常，并把跟踪时反汇编得到的代码与bootasm.S和bootblock.asm比较。
* 在gdbinit文件末尾加上
```
b *0x7c00
c
set architecture i386
```
* 执行make debug可观察到结果与bootblock.asm中一致，与bootasm.S的结果存在实模式和保护模式的对应

## [练习3]
从`%cs=0 $pc=0x7c00`，进入后

首先将flag和段寄存器置0


然后将键盘控制器上的A20线置于高电位来开启A20


从引导区载入已有GDT表

将cr0寄存器PE位置1以开启保护模式


最后更新cs基地址并设置段寄存器以建立堆栈

## [练习4]
bootloader通过readsect函数读取扇区。

    readsect函数首先调用waitdisk()函数等待磁盘准备好。

    然后设置硬盘的IO地址寄存器0x1f0-0x1f7来设置读取的扇区并发出读取命令。

    之后再次调用waitdisk()函数等待磁盘准备好

    最后将数据读取至指定位置

bootloader加载ELF格式的OS

    首先读取ELF头部用以判断是否合法（判断方式为magic number判等法）

    之后根据头部中的描述表把各部分数据加载到对应内存中，并获得内核入口地址

## [练习5]
得到如下输出
```
ebp:0x00007b08 eip:0x001009a6 args:0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:306: print_stackframe+21
ebp:0x00007b18 eip:0x00100c98 args:0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args:0x0010349c 0x00103480 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --
```
最后一行、对应的是第一个使用堆栈的函数，即bootmain.c中的bootmain。

bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。
call指令压栈，所以bootmain中ebp为0x7bf8。

## [练习6]
###[练习6.1] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，
段选择子选择的段的基址加上偏移就是入口地址。
###[练习6.2]请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。
思路就是翻译注释，详见代码
###[练习6.3]请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数
同上
