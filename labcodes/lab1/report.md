#OS lab 01

##[练习1]

[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)
```
bin/ucore.img
| 生成ucore.img的相关代码为
| $(UCOREIMG): $(kernel) $(bootblock)
|	$(V)dd if=/dev/zero of=$@ count=10000
|	$(V)dd if=$(bootblock) of=$@ conv=notrunc
|	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
|
| 为了生成ucore.img，首先需要生成bootblock、kernel
|
|>	bin/bootblock
|	| 生成bootblock的相关代码为
|	| $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
|	|	@echo + ld $@
|	|	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ \
|	|		-o $(call toobj,bootblock)
|	|	@$(OBJDUMP) -S $(call objfile,bootblock) > \
|	|		$(call asmfile,bootblock)
|	|	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) \
|	|		$(call outfile,bootblock)
|	|	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
|	|
|	| 为了生成bootblock，首先需要生成bootasm.o、bootmain.o、sign
|	|
|	|>	obj/boot/bootasm.o, obj/boot/bootmain.o
|	|	| 生成bootasm.o,bootmain.o的相关makefile代码为
|	|	| bootfiles = $(call listf_cc,boot) 
|	|	| $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),\
|	|	|	$(CFLAGS) -Os -nostdinc))
|	|	| 实际代码由宏批量生成
|	|	| 
|	|	| 生成bootasm.o需要bootasm.S
|	|	| 实际命令为
|	|	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \
|	|	| 	-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \
|	|	| 	-c boot/bootasm.S -o obj/boot/bootasm.o
|	|	| 其中关键的参数为
|	|	| 	-ggdb  生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore。
|	|	|	-m32  生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位的软件。
|	|	| 	-gstabs  生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息
|	|	| 	-nostdinc  不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的，所以所有的服务要自给自足。
|	|	|	-fno-stack-protector  不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核，ucore内核好像还用不到此功能。
|	|	| 	-Os  为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootloader的最终大小不能大于510字节。
|	|	| 	-I<dir>  添加搜索头文件的路径
|	|	| 
|	|	| 生成bootmain.o需要bootmain.c
|	|	| 实际命令为
|	|	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \
|	|	| 	-fno-stack-protector -Ilibs/ -Os -nostdinc \
|	|	| 	-c boot/bootmain.c -o obj/boot/bootmain.o
|	|	| 新出现的关键参数有
|	|	| 	-fno-builtin  除非用__builtin_前缀，
|	|	|	              否则不进行builtin函数的优化
|	|
|	|>	bin/sign
|	|	| 生成sign工具的makefile代码为
|	|	| $(call add_files_host,tools/sign.c,sign,sign)
|	|	| $(call create_target_host,sign,sign)
|	|	| 
|	|	| 实际命令为
|	|	| gcc -Itools/ -g -Wall -O2 -c tools/sign.c \
|	|	| 	-o obj/sign/tools/sign.o
|	|	| gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
|	|
|	| 首先生成bootblock.o
|	| ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
|	|	obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
|	| 其中关键的参数为
|	|	-m <emulation>  模拟为i386上的连接器
|	|	-nostdlib  不使用标准库
|	|	-N  设置代码段和数据段均可读写
|	|	-e <entry>  指定入口
|	|	-Ttext  制定代码段开始位置
|	|
|	| 拷贝二进制代码bootblock.o到bootblock.out
|	| objcopy -S -O binary obj/bootblock.o obj/bootblock.out
|	| 其中关键的参数为
|	|	-S  移除所有符号和重定位信息
|	|	-O <bfdname>  指定输出格式
|	|
|	| 使用sign工具处理bootblock.out，生成bootblock
|	| bin/sign obj/bootblock.out bin/bootblock
|
|>	bin/kernel
|	| 生成kernel的相关代码为
|	| $(kernel): tools/kernel.ld
|	| $(kernel): $(KOBJS)
|	| 	@echo + ld $@
|	| 	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
|	| 	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
|	| 	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; \
|	| 		/^$$/d' > $(call symfile,kernel)
|	| 
|	| 为了生成kernel，首先需要 kernel.ld init.o readline.o stdio.o kdebug.o
|	|	kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o
|	|	trapentry.o vectors.o pmm.o  printfmt.o string.o
|	| kernel.ld已存在
|	|
|	|>	obj/kern/*/*.o 
|	|	| 生成这些.o文件的相关makefile代码为
|	|	| $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,\
|	|	|	$(KCFLAGS))
|	|	| 这些.o生成方式和参数均类似，仅举init.o为例，其余不赘述
|	|>	obj/kern/init/init.o
|	|	| 编译需要init.c
|	|	| 实际命令为
|	|	|	gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 \
|	|	|		-gstabs -nostdinc  -fno-stack-protector \
|	|	|		-Ilibs/ -Ikern/debug/ -Ikern/driver/ \
|	|	|		-Ikern/trap/ -Ikern/mm/ -c kern/init/init.c \
|	|	|		-o obj/kern/init/init.o
|	| 
|	| 生成kernel时，makefile的几条指令中有@前缀的都不必需
|	| 必需的命令只有
|	| ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
|	| 	obj/kern/init/init.o obj/kern/libs/readline.o \
|	| 	obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
|	| 	obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
|	| 	obj/kern/driver/clock.o obj/kern/driver/console.o \
|	| 	obj/kern/driver/intr.o obj/kern/driver/picirq.o \
|	| 	obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
|	| 	obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
|	| 	obj/libs/printfmt.o obj/libs/string.o
|	| 其中新出现的关键参数为
|	|	-T <scriptfile>  让连接器使用指定的脚本
|
| 生成一个有10000个块的文件，每个块默认512字节，用0填充
| dd if=/dev/zero of=bin/ucore.img count=10000
|
| 把bootblock中的内容写到第一个块
| dd if=bin/bootblock of=bin/ucore.img conv=notrunc
|
| 从第二个块开始写kernel中的内容
| dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征使什么?

答:磁盘主引导扇区只有512字节。且第510个字节为0x55，第511个（倒数第一个）字节为0xAA。

##[练习2]

[练习2.1]从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

单步跟踪，方法如下：
1 修改 lab1/tools/gdbinit为：
```
set architecture i8086
target remote :1234
```

2 在lab1 下执行
```
make debug
```

3 在gdb调试界面输入以下命令
```
layout prev
si
```
即可单步跟踪BIOS的执行。

[练习2.2] 在初始化未做0x7c00设置实地址断点，测试断点正常。

在tools/gdbinit 结尾加上
```
    set architecture i8086
    layout prev
    b *0x7C00
    c
    set architecture i386
```

运行"make debug"即可得
```
:call <SNR>107_SparkupNext()
Breakpoint 2, 0x00007c00 in ?? ()

=> 0x7c00:      cli    
   0x7c01:      cld    
   0x7c02:      xor    %eax,%eax
   0x7c04:      mov    %eax,%ds
   0x7c06:      mov    %eax,%es
   0x7c08:      mov    %eax,%ss 
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al

```

[练习2.3]
从0x7C00开始跟踪代码运行，将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较。

```
----------------
IN: 
    0x00007c00:  cli    
    0x00007c01:  cld    
    0x00007c02:  xor    %ax,%ax
    0x00007c04:  mov    %ax,%ds
    0x00007c06:  mov    %ax,%es
    0x00007c08:  mov    %ax,%ss

----------------
IN: 
0x00007c0a:  in     $0x64,%al

----------------
IN: 
0x00007c0c:  test   $0x2,%al
0x00007c0e:  jne    0x7c0a

----------------
IN: 
0x00007c10:  mov    $0xd1,%al
0x00007c12:  out    %al,$0x：wq64
0x00007c14:  in     $0x64,%al
0x00007c16:  test   $0x2,%al
0x00007c18:  jne    0x7c14

----------------
IN: 
0x00007c1a:  mov    $0xdf,%al
0x00007c1c:  out    %al,$0x60
0x00007c1e:  lgdtw  0x7c6c
0x00007c23:  mov    %cr0,%eax
0x00007c26:  or     $0x1,%eax
0x00007c2a:  mov    %eax,%cr0

----------------
IN: 
0x00007c2d:  ljmp   $0x8,$0x7c32

----------------
IN: 
0x00007c32:  mov    $0x10,%ax
0x00007c36:  mov    %eax,%ds

----------------
IN: 
0x00007c38:  mov    %eax,%es

----------------
IN: 
0x00007c3a:  mov    %eax,%fs
0x00007c3c:  mov    %eax,%gs
0x00007c3e:  mov    %eax,%ss

----------------
IN: 
0x00007c40:  mov    $0x0,%ebp

----------------
IN: 
0x00007c45:  mov    $0x7c00,%esp
0x00007c4a:  call   0x7cd3

``
可见其与bootasm.S和bootblock.asm中代码相同。


##[练习3]
BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。

#为何开启A20，以及如何开启A20

为何：很显然，在实模式下要访问高端内存区，这个开关必须打开，在保护模式下，由于使用32位地址线，如果A20恒等于0，那么系统只能访问奇数兆的内存，即只能访问0--1M、2-3M、4-5M......，这显然是不行的，所以在保护模式下，这个开关也必须打开。

如何：
```
等待8042 Input buffer为空；
发送Write 8042 Output Port （P2）命令到8042 Input buffer；
等待8042 Input buffer为空；
将8042 Output Port（P2）得到字节的第2位置1，然后写入8042 Input buffer；
```

#如何初始化GDT表

一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可
```
    lgdt gdtdesc
```


进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式
```
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
```

通过长跳转更新cs的基地址
```
	 ljmp $PROT_MODE_CSEG, $protcseg
	.code32
	protcseg:
```

设置段寄存器，并建立堆栈
```
	    movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
	    movl $0x0, %ebp
	    movl $start, %esp
```
转到保护模式完成，进入boot主方法
```
	    call bootmain
```

##[练习4]

[练习4.1]bootloader是如何读取硬盘扇区的？




[练习4.2]bootloader是如何加载ELF格式的OS？

首先看readsect函数，
`readsect`从设备的第secno扇区读取数据到dst位置
```
	static void
	readsect(void *dst, uint32_t secno) {
	    waitdisk();
	
	    outb(0x1F2, 1);                         // 设置读取扇区的数目为1
	    outb(0x1F3, secno & 0xFF);
	    outb(0x1F4, (secno >> 8) & 0xFF);
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	        // 上面四条指令联合制定了扇区号
	        // 在这4个字节线联合构成的32位参数中
	        //   29-31位强制设为1
	        //   28位(=0)表示访问"Disk 0"
	        //   0-27位是28位的偏移量
	    outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
	
	    waitdisk();

	    insl(0x1F0, dst, SECTSIZE / 4);         // 读取到dst位置，
	                                            // 幻数4因为这里以DW为单位
	}
```

readseg简单包装了readsect，可以从设备读取任意长度的内容。
```
	static void
	readseg(uintptr_t va, uint32_t count, uint32_t offset) {
	    uintptr_t end_va = va + count;
	
	    va -= offset % SECTSIZE;
	
	    uint32_t secno = (offset / SECTSIZE) + 1; 
	    // 加1因为0扇区被引导占用
	    // ELF文件从1扇区开始
	
	    for (; va < end_va; va += SECTSIZE, secno ++) {
	        readsect((void *)va, secno);
	    }
	}
```

在bootmain函数中，
```
	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```

##[练习5]
实现函数调用堆栈跟踪函数

```
ebp:0x00007b08 eip:0x00100a67 args:0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:306: print_stackframe+21
ebp:0x00007b18 eip:0x00100d5e args:0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args:0x001032fc 0x001032e0 0x00001308 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d6a args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d69 --
```
可见其与题述一致。

最后一行对应第一个使用堆栈的函数，bootmain.c中的bootmain。
bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。
call指令压栈，故bootmain中ebp为0x7bf8。


##[练习6]

