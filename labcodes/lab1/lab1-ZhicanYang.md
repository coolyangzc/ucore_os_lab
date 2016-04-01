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

	大意为启动qemu，等待2s，然后开启另一个终端调用gdb，gdb直接执行`tools/gdbinit`中的内容

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

## [练习3] 分析bootloader进入保护模式的过程 ##

#### 1. 为何开启A20，以及如何开启A20 ####

早期（80286之前）的系统总线仅20根，只能访问2<sup>20</sup> = 1MB内存空间。A20门意为第21根总线（从0计数），是IBM使用键盘控制器上剩余的一些输出线来管理的这第21根地址线，若A20门被禁止就能兼容之前的机器。刚加电时A20被禁止，为了启用所有系统总线，需要打开A20门。

开启方式：

	seta20.1:
	    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
	    testb $0x2, %al
	    jnz seta20.1
	
	    movb $0xd1, %al                                 # 0xd1 -> port 0x64
	    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
	
	seta20.2:
	    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
	    testb $0x2, %al
	    jnz seta20.2
	
	    movb $0xdf, %al                                 # 0xdf -> port 0x60
	    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1

#### 2. 如何初始化GDT表 ####

通过`lgdt(Load GDT)`语句载入GDT表：

	lgdt gdtdesc

`gdtdesc`的定义是：

	# Bootstrap GDT
	.p2align 2                                          # force 4 byte alignment
	gdt:
	    SEG_NULLASM                                     # null seg
	    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
	    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel
	
	gdtdesc:
	    .word 0x17                                      # sizeof(gdt) - 1
	    .long gdt                                       # address gdt	

#### 3. 如何使能和进入保护模式 ####

1. 进入保护模式
	
	开CR0寄存器PE位（置1）：

		movl %cr0, %eax
		orl $CR0_PE_ON, %eax
		movl %eax, %cr0

2. 切换32位模式

	其中`ljmp`的格式是`ljmp 段选择子 段内偏移`，这个长跳转相当于把原本20位的pc扩展为32位

		    # Jump to next instruction, but in 32-bit code segment.
		    # Switches processor into 32-bit mode.
		    ljmp $PROT_MODE_CSEG, $protcseg
		
		.code32                                             # Assemble for 32-bit mode
		protcseg:

3. 设置数据段寄存器

		.code32                                             # Assemble for 32-bit mode
		protcseg:
		    # Set up the protected-mode data segment registers
		    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
		    movw %ax, %ds                                   # -> DS: Data Segment
		    movw %ax, %es                                   # -> ES: Extra Segment
		    movw %ax, %fs                                   # -> FS
		    movw %ax, %gs                                   # -> GS
		    movw %ax, %ss                                   # -> SS: Stack Segment

4. 建立堆栈

		    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
		    movl $0x0, %ebp
		    movl $start, %esp

5. 进入保护模式完成，调用`boot/bootmain.c`中的`bootmain`方法

		call bootmain

## [练习4] 分析bootloader加载ELF格式的OS的过程 ##

#### 1. bootloader如何读取硬盘扇区的？ ####

`/boot/bootmain.c`中的`readsect`过程：

	//将secno扇区读取到dst指针所指的位置
	static void
	readsect(void *dst, uint32_t secno) {
	    // wait for disk to be ready
	    waitdisk();
	
	    outb(0x1F2, 1);                         // count = 1
	    outb(0x1F3, secno & 0xFF);
	    outb(0x1F4, (secno >> 8) & 0xFF);
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors
	
	    // wait for disk to be ready
	    waitdisk();
	
	    // read a sector
	    insl(0x1F0, dst, SECTSIZE / 4);
	}

其中`waitdisk`通过调用inb不断读取直到成功：

	static void
	waitdisk(void) {
	    while ((inb(0x1F7) & 0xC0) != 0x40)
	        /* do nothing */;
	}

#### 2. bootloader是如何加载ELF格式的OS？ ####
	
`/boot/bootmain.c`中的`bootmain`过程：

	/* bootmain - the entry of bootloader */
	void
	bootmain(void) {
	    // read the 1st page off disk 读取ELF文件的第一页
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // is this a valid ELF? 通过约定常数检查文件类型
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // load each program segment (ignores ph flags)
	    //ph是每一个Program Header的指针
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    //eph等于ph加上Program Header的数目，相当于最后一项后的位置
	    eph = ph + ELFHDR->e_phnum;
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	
	    // call the entry point from the ELF header 跳转到ucore，转交控制权
	    // note: does not return
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	
	    /* do nothing */
	    while (1);
	}

## [练习5] 实现函数调用堆栈跟踪函数 ##

添加代码如下：
	
	void
	print_stackframe(void) {
	     /* LAB1 2013011377 : STEP 1 */
	     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
	      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
	      * (3) from 0 .. STACKFRAME_DEPTH
	      *    (3.1) printf value of ebp, eip
	      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
	      *    (3.3) cprintf("\n");
	      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
	      *    (3.5) popup a calling stackframe
	      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
	      *                   the calling funciton's ebp = ss:[ebp]
	      */
	    uint32_t ebp = read_ebp();
	    uint32_t eip = read_eip();
	    while (ebp > 0) {
	        cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
	        uint32_t *stk = (uint32_t*) ebp + 2;
	        int i;
	        for (i = 0; i < 4 ; i++)
	            cprintf("0x%08x ", *(stk+i));
	        cprintf("\n");
	        print_debuginfo(eip - 1);
	        eip = *((uint32_t*) ebp + 1);
	        ebp = *((uint32_t*) ebp);
	    }
	}

输出结果如下，与题面中的要求大致相同：

	ebp:0x00007b08 eip:0x001009a6 args:0x00010094 0x00000000 0x00007b38 0x00100092 
	    kern/debug/kdebug.c:306: print_stackframe+21
	ebp:0x00007b18 eip:0x00100c84 args:0x00000000 0x00000000 0x00000000 0x00007b88 
	    kern/debug/kmonitor.c:125: mon_backtrace+10
	ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64 
	    kern/init/init.c:48: grade_backtrace2+33
	ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029 
	    kern/init/init.c:53: grade_backtrace1+38
	ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d 
	    kern/init/init.c:58: grade_backtrace0+23
	ebp:0x00007b98 eip:0x001000fe args:0x001032dc 0x001032c0 0x0000130a 0x00000000 
	    kern/init/init.c:63: grade_backtrace+34
	ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094 
	    kern/init/init.c:28: kern_init+84
	ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
	    <unknow>: -- 0x00007d67 --

最后一行是第一个使用堆栈（调用函数）的函数，即`bootmain`，其`ebp`为`0x00007bf8`，与`boot/bootasm.S`中设置的堆栈起始位置`0x7c00`相符：

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain

函数调用时，先将调用的参数压入栈中，然后保存eip的值（即Return Address），最后保存ebp的值。此时被调用的函数的ebp指向该处，就构成了一个函数调用栈。

## [练习6] 完善中断初始化和处理 ##

#### 1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？ ####

	/* Gate descriptors for interrupts and traps */
	struct gatedesc {
	    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
	    unsigned gd_ss : 16;            // segment selector
	    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
	    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
	    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
	    unsigned gd_s : 1;                // must be 0 (system)
	    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
	    unsigned gd_p : 1;                // Present
	    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
	};

一个表项占8个字节，其中前16位为低位和后16位为高位相接后为中断处理代码的入口

#### 2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。 ####

	/* idt_init - initialize IDT to each of the entry points in kern/trap/vectors.S */
	void
	idt_init(void) {
	     /* LAB1 2013011377 : STEP 2 */
	     /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
	      *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
	      *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
	      *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
	      *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
	      * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
	      *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
	      * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
	      *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
	      *     Notice: the argument of lidt is idt_pd. try to find it!
	      */
	
	    extern uintptr_t __vectors[];
	    int i;
	    for (i = 0; i < 256; i++)
	        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
	    SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
	    lidt(&idt_pd);
	}

#### 3. 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。 ####

	case IRQ_OFFSET + IRQ_TIMER:
	        /* LAB1 2013011377 : STEP 3 */
	        /* handle the timer interrupt */
	        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
	         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
	         * (3) Too Simple? Yes, I think so!
	         */
	        ticks++;
	        if (ticks == TICK_NUM) {
	            print_ticks();
	            ticks = 0;    
	        }
	        break;

## [扩展练习1] ##
#### 增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务 ####

`lab1_switch_to_user`和`lab1_switch_to_kernel`函数通过触发对应的`T_SWITCH_TOU`和`T_SWITCH_TOK`中断实现。

使用户态获得调用`T_SWITCH_TOK`中断的权限，修改`idt_init`：

	for (i = 0; i < 256; i++)
	        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
	    SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
	    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);

中断处理通过修改`trapframe`的各个值实现。


## 我的实现与参考答案的区别 ##

在练习6的第2问中，参考答案没有将系统调用中断`T_SYSCALL`设置权限为用户态，也没有使用陷阱门描述符。我添加如下语句：

	SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);

## 重要知识点 ##

1. BIOS->bootloader->ucore的加载顺序
2. bootloader中实模式->保护模式的切换
3. 分段机制，即各种段描述符、描述符表、各种选择子
4. 函数调用栈
5. 中断向量表