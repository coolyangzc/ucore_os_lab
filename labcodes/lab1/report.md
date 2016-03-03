# Lab1 Report#
---
## [练习1] 了解通过make生成执行文件的过程 ##

#### 1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果) ####

分析`labcodes/lab1/`下的`Makefile文件`：

	ALLOBJS	:=
	ALLDEPS	:=
	TARGETS	:=
	
	include tools/function.mk

TARGETS在`tools/function.mk`中通过如下语句定义：

	TARGETS += $$(__temp_target__)

在`Makefile`中，每种命令都依赖`$(UCOREIMG)`，如：

	qemu-mon: $(UCOREIMG)
		$(V)$(QEMU) -monitor stdio -hda $< -serial null
	qemu: $(UCOREIMG)
		$(V)$(QEMU) -parallel stdio -hda $< -serial null

`$(UCOREIMG)`即`ucore.img`，依赖`$(kernel)`和`$(bootblock)`：

	# create ucore.img
	UCOREIMG	:= $(call totarget,ucore.img)
		#totarget在function.mk中定义为：
		#totarget = $(addprefix $(BINDIR)$(SLASH),$(1))
		#即./bin/
	
	$(UCOREIMG): $(kernel) $(bootblock)
			#dd指用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换，在此处作用是将文件载入虚拟硬盘中。
			#$@是目标文件，在此处即为ucore.img，即虚拟硬盘文件
		$(V)dd if=/dev/zero of=$@ count=10000
			#/dev/zero是ubuntu系统提供的一个伪文件，产生无穷的二进制零流，count=10000即为拷贝10000个块，一块大小默认为512字节，则共拷贝了5.12MB。
		$(V)dd if=$(bootblock) of=$@ conv=notrunc
			#将bootblock拷贝到第一块
		$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
			#seek=1即为跳过第一个块，故从第二块开始拷贝kernel

	$(call create_target,ucore.img)

`$(kernel)`定义如下：

	KOBJS	= $(call read_packet,kernel libs)
	
	# create kernel target
	kernel = $(call totarget,kernel)
	
	$(kernel): tools/kernel.ld
		#$(kernel)即tools/kernel.ld
	
	$(kernel): $(KOBJS)
		@echo + ld $@
		$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
		
		@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
		@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
	
	$(call create_target,kernel)

Makefile中指令前加`@`即运行时不输出该语句本身，可通过`make V=`重新输出该语句。

`$(kernel)`依赖的`$(KOBJS)`可通过make时编译的文件简单获得：
	
	+ cc kern/init/init.c
	+ cc kern/libs/readline.c
	+ cc kern/libs/stdio.c
	+ cc kern/debug/kdebug.c
	+ cc kern/debug/kmonitor.c
	+ cc kern/debug/panic.c
	+ cc kern/driver/clock.c
	+ cc kern/driver/console.c
	+ cc kern/driver/intr.c
	+ cc kern/driver/picirq.c
	+ cc kern/trap/trap.c
	+ cc kern/trap/trapentry.S
	+ cc kern/trap/vectors.S
	+ cc kern/mm/pmm.c
	+ cc libs/printfmt.c
	+ cc libs/string.c
	+ ld bin/kernel

`$(bootblock)`的定义如下：

	# create bootblock
	bootfiles = $(call listf_cc,boot)
	$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
	
	bootblock = $(call totarget,bootblock)
	
	$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
		@echo + ld $@
		$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
		@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
		@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
		@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
		@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
	
	$(call create_target,bootblock)

同样，`bootblock`依赖的是：

	+ cc boot/bootasm.S
	+ cc boot/bootmain.c
	+ cc tools/sign.c
	+ ld bin/bootblock

编译c文件使用的参数如下：

	HOSTCC		:= gcc
	HOSTCFLAGS	:= -g -Wall -O2
	CC		:= $(GCCPREFIX)gcc
	CFLAGS	:= -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
	CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null 

#### 2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？ ####

分析`lab1/tools/sign.c`：

1. 主引导扇区大小为512字节：

		char buf[512]; 

2. 主引导扇区以0x55AA结尾：

		buf[510] = 0x55;
		buf[511] = 0xAA;

3. 故装载的文件大小不能超过510字节：

		if (st.st_size > 510) {
	        fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
	        return -1;
	    }

## [练习2] 使用qemu执行并调试lab1中的软件##

#### 1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。 ####

1. 修改`tools/gdbinit`为：

		file bin/kernel
		target remote :1234
		set architecture i8086

2. 在终端执行`make debug`

	`debug`在`makefile`中的定义为：
	
		debug: $(UCOREIMG)
		$(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
		$(V)sleep 2
		$(V)$(TERMINAL)  -e "cgdb -q -x tools/gdbinit"

	大意为启动qemu，等待2s，然后开启另一个终端调用gdb，gdb直接执行tools/gdbinit中的内容

3. 因为BIOS并非使用c语言所写，所以在gdb中暂时只能使用`ni(nexti)`或`si(stepi)`进行汇编指令层级的单步调试

4. 使用`x pc`查看当前运行的汇编语句，或`x /5i $pc`查看5句汇编语句等。

#### 2. 在初始化位置0x7c00设置实地址断点,测试断点正常 ####

1. 修改`tools/gdbinit`为：

		file bin/kernel
		target remote :1234
		set architecture i8086
		b *0x7c00

2. 参照问题1步骤，进入gdb时已设置好断点`0x7c00`，此时pc停留在加电后的第一条指令`0xfff0`
3. 使用`c(continue)`即可连续执行到断点处`0x7c00`

#### 3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较 ####

1. 使用`x /10i $pc`查看汇编指令，显示如下：

		=> 0x7c00:      cli    
		   0x7c01:      cld    
		   0x7c02:      xor    %ax,%ax
		   0x7c04:      mov    %ax,%ds
		   0x7c06:      mov    %ax,%es
		   0x7c08:      mov    %ax,%ss
		   0x7c0a:      in     $0x64,%al
		   0x7c0c:      test   $0x2,%al
		   0x7c0e:      jne    0x7c0a
		   0x7c10:      mov    $0xd1,%al

2. 查看`boot/bootasm.S`的`start`处：

		.globl start
		start:
		.code16                                             # Assemble for 16-bit mode
		    cli                                             # Disable interrupts
		    cld                                             # String operations increment
		
		    # Set up the important data segment registers (DS, ES, SS).
		    xorw %ax, %ax                                   # Segment number zero
		    movw %ax, %ds                                   # -> Data Segment
		    movw %ax, %es                                   # -> Extra Segment
		    movw %ax, %ss                                   # -> Stack Segment

	与单步跟踪反汇编的结果相符。

#### 4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。 ####

1. 修改`tools/gdbinit`为：
		
		file obj/bootblock.o
		target remote :1234
		set architecture i8086
		b bootmain
		c

2. `make debug`，即停在`bootmain.c`中的`bootmain(void)`入口处。