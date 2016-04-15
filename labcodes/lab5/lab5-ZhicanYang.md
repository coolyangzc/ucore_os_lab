# Lab5 Report#
---
## [练习0] 填写已有实验 ##

使用`meld`工具辅助人工检查完成。

## [练习1] 加载应用程序并执行 ##

#### 1. 简要说明你的设计实现过程 ####

`kern/process/proc.c`中的`load_icode`函数内：

	/* LAB5:EXERCISE1 2013011377
	     * should set tf_cs,tf_ds,tf_es,tf_ss,tf_esp,tf_eip,tf_eflags
	     * NOTICE: If we set trapframe correctly, then the user level process can return to USER MODE from kernel. So
	     *          tf_cs should be USER_CS segment (see memlayout.h)
	     *          tf_ds=tf_es=tf_ss should be USER_DS segment
	     *          tf_esp should be the top addr of user stack (USTACKTOP)
	     *          tf_eip should be the entry point of this binary program (elf->e_entry)
	     *          tf_eflags should be set to enable computer to produce Interrupt
	     */
	    tf->tf_cs = USER_CS;
	    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
	    tf->tf_esp = USTACKTOP;
	    tf->tf_eip = elf->e_entry;
	    tf->tf_eflags =  FL_IF;

基本是按照要求设置段寄存器/栈指针/指令寄存器等即可。

#### 2. 当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的 ####

首先是调用`proc_run`执行某个进程：

	void
	proc_run(struct proc_struct *proc) {
	    if (proc != current) {
	        bool intr_flag;
	        struct proc_struct *prev = current, *next = proc;
	        local_intr_save(intr_flag);
	        {
	            current = proc;
	            load_esp0(next->kstack + KSTACKSIZE);
	            lcr3(next->cr3);
	            switch_to(&(prev->context), &(next->context));
	        }
	        local_intr_restore(intr_flag);
	    }
	}

最终`switch_to`处切换前后进程的上下文之后，会返回到`forkret`：

	// forkret -- the first kernel entry point of a new thread/process
	// NOTE: the addr of forkret is setted in copy_thread function
	//       after switch_to, the current proc will execute here.
	static void
	forkret(void) {
	    forkrets(current->tf);
	}

`forkret`进一步调用`forkrets`，将栈帧tf传入参数：

	.globl __trapret
	__trapret:
	    # restore registers from stack
	    popal
	
	    # restore %ds, %es, %fs and %gs
	    popl %gs
	    popl %fs
	    popl %es
	    popl %ds
	
	    # get rid of the trap number and error code
	    addl $0x8, %esp
	    iret
	
	.globl forkrets
	forkrets:
	    # set stack to this new process's trapframe
	    movl 4(%esp), %esp
	    jmp __trapret

在`iret`之后，会返回tf中设定的状态，也即在`load_icode`中设置的：

    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;

此时进程进入用户态，并开始执行应用程序的第一条指令。

## [练习2] 父进程复制自己的内存空间给子进程 ##

#### 1. 补充`copy_range`的实现 ####

	/* copy_range - copy content of memory (start, end) of one process A to another process B
	 * @to:    the addr of process B's Page Directory
	 * @from:  the addr of process A's Page Directory
	 * @share: flags to indicate to dup OR share. We just use dup method, so it didn't be used.
	 *
	 * CALL GRAPH: copy_mm-->dup_mmap-->copy_range
	 */
	int
	copy_range(pde_t *to, pde_t *from, uintptr_t start, uintptr_t end, bool share) {
	    assert(start % PGSIZE == 0 && end % PGSIZE == 0);
	    assert(USER_ACCESS(start, end));
	    // copy content by page unit.
	    do {
	        //call get_pte to find process A's pte according to the addr start
	        pte_t *ptep = get_pte(from, start, 0), *nptep;
	        if (ptep == NULL) {
	            start = ROUNDDOWN(start + PTSIZE, PTSIZE);
	            continue ;
	        }
	        //call get_pte to find process B's pte according to the addr start. If pte is NULL, just alloc a PT
	        if (*ptep & PTE_P) {
	            if ((nptep = get_pte(to, start, 1)) == NULL) {
	                return -E_NO_MEM;
	            }
	        uint32_t perm = (*ptep & PTE_USER);
	        //get page from ptep
	        struct Page *page = pte2page(*ptep);
	        // alloc a page for process B
	        struct Page *npage=alloc_page();
	        assert(page!=NULL);
	        assert(npage!=NULL);
	        int ret=0;
	        /* LAB5:EXERCISE2 2013011377
	         * replicate content of page to npage, build the map of phy addr of nage with the linear addr start
	         *
	         * Some Useful MACROs and DEFINEs, you can use them in below implementation.
	         * MACROs or Functions:
	         *    page2kva(struct Page *page): return the kernel vritual addr of memory which page managed (SEE pmm.h)
	         *    page_insert: build the map of phy addr of an Page with the linear addr la
	         *    memcpy: typical memory copy function
	         *
	         * (1) find src_kvaddr: the kernel virtual address of page
	         * (2) find dst_kvaddr: the kernel virtual address of npage
	         * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
	         * (4) build the map of phy addr of  nage with the linear addr start
	         */
	        void *src_kvaddr = page2kva(page);
	        void *dst_kvaddr = page2kva(npage);
	        memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
	        ret = page_insert(to, npage, start, perm);
	        assert(ret == 0);
	        }
	        start += PGSIZE;
	    } while (start != 0 && start < end);
	    return 0;
	}

#### 2. 如何设计实现"Copy on Write 机制" ####

给进程/线程使用的虚拟空间添加一个域，记录该虚拟空间被多少进程/线程分享(引用)，在ucore中，已有`mm_struct`下的`mm_count`域。

在fork时，子进程先不申请新的内存空间，而是直接持有父进程的`mm_struct`指针，将对应的`mm_count`+1并将其属性设置为只读。此时若所有进程都只有读内存操作，则所有进程就只需共享这一份内存空间即可。

若有某个进程进行写操作，此时会触发`Page Fault`，检查后发现页本身是可读写的，系统就知道是由于共享VMA导致的。此时将该内存空间的`mm_count`-1，若`mm_count`变为1，则将其属性设置为可读写，最后复制一份给该进程使用，标记为可读写。

## [练习3] 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现 ##

#### 1. fork/exec/wait/exit在实现中是如何影响进程的执行状态的 ####

* fork: 创建子进程，使子进程进入就绪态。
* exec：尝试使该进程转到用户态并执行用户程序，若加载失败，则退出。
* wait：若存在符合条件(限制pid)的处于僵尸态的子进程，回收其中一个并将其返回值传递给父进程；若满足条件的子进程没有处于僵尸态的，则使父进程进入等待态等待子进程唤醒。
* exit：回收大部分资源，使进程进入僵尸态并尝试唤醒父进程

#### 2. 给出ucore中一个用户态进程的执行状态生命周期图 ####

	  alloc_proc                                        RUNNING（运行）
	      +                                          +--<----<--+
	      +                                          + proc_run +
	      V                                          +-->---->--+ 
	PROC_UNINIT(创建) -- proc_init/wakeup_proc --> PROC_RUNNABLE(就绪) -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING（等待） --
	                                           ^      +                                                                            +
	                                           |      +--- do_exit --> PROC_ZOMBIE(僵尸)                                            +  
	                                           +                                                                                   + 
	                                           ---------------------------------------wakeup_proc-----------------------------------

## 我的实现与参考答案的区别 ##

在此前的Lab1和Lab4已经考虑过相对应的边界情况，所以不需要做改动。

在`print_ticks`中，下文注释处会导致任何触发了时钟中断的用户进程(spin/waitkill)无法通过：
	
	static void print_ticks() {
	    cprintf("%d ticks\n",TICK_NUM);
	#ifdef DEBUG_GRADE
	    cprintf("End of Test.\n");
	    //panic("EOT: kernel seems ok.");
	#endif
	}

## 重要知识点 ##
1. 内核栈与用户栈的含义与区别
2. 用户态与内核态切换的过程与方式
3. 进程/线程的运行状态与生命周期