# Lab1 Report#
---
## [练习1] ##

### 1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果) ###

分析`labcodes\lab1\`下的`Makefile文件`：

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