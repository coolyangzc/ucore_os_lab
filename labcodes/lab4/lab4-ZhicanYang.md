# Lab4 Report#
---
## [练习0] 填写已有实验 ##

使用`meld`工具辅助人工检查完成。

## [练习1] 分配并初始化一个进程控制块 ##

#### 1. 简要说明你的设计实现过程 ####

`kern/process/proc.c`中的`alloc_proc`函数用于分配并初始化一个`proc_struct`：

	// alloc_proc - alloc a proc_struct and init all fields of proc_struct
	static struct proc_struct *
	alloc_proc(void) {
	    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
	    if (proc != NULL) {
	    //LAB4:EXERCISE1 2013011377
	    /*
	     * below fields in proc_struct need to be initialized
	     *       enum proc_state state;                      // Process state
	     *       int pid;                                    // Process ID
	     *       int runs;                                   // the running times of Proces
	     *       uintptr_t kstack;                           // Process kernel stack
	     *       volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
	     *       struct proc_struct *parent;                 // the parent process
	     *       struct mm_struct *mm;                       // Process's memory management field
	     *       struct context context;                     // Switch here to run process
	     *       struct trapframe *tf;                       // Trap frame for current interrupt
	     *       uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
	     *       uint32_t flags;                             // Process flag
	     *       char name[PROC_NAME_LEN + 1];               // Process name
	     */
	        memset(proc, 0, sizeof(struct proc_struct));
	        proc->state = PROC_UNINIT;
	        proc->pid = -1;
	        proc->cr3 = boot_cr3;
	    }
	    return proc;
	}


#### 2. 请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用 ####

`struct context context`是进程或线程的上下文，用于保存进程/线程的运行状态，包含eip,esp以及通用寄存器（除了不需保存的eax）：
	
	struct context {
	    uint32_t eip;
	    uint32_t esp;
	    uint32_t ebx;
	    uint32_t ecx;
	    uint32_t edx;
	    uint32_t esi;
	    uint32_t edi;
	    uint32_t ebp;
	};

在内核态中进行进程切换时，不需要改变段寄存器，故只需切换上下文即切换这些寄存器的内容即可。

`struct trapframe *tf`的定义在`kern/trap/trap.h`中：

	struct trapframe {
	    struct pushregs tf_regs;
	    uint16_t tf_gs;
	    uint16_t tf_padding0;
	    uint16_t tf_fs;
	    uint16_t tf_padding1;
	    uint16_t tf_es;
	    uint16_t tf_padding2;
	    uint16_t tf_ds;
	    uint16_t tf_padding3;
	    uint32_t tf_trapno;
	    /* below here defined by x86 hardware */
	    uint32_t tf_err;
	    uintptr_t tf_eip;
	    uint16_t tf_cs;
	    uint16_t tf_padding4;
	    uint32_t tf_eflags;
	    /* below here only when crossing rings, such as from user to kernel */
	    uintptr_t tf_esp;
	    uint16_t tf_ss;
	    uint16_t tf_padding5;
	} __attribute__((packed));

其中`struct pushregs tf_regs`为

	/* registers as pushed by pushal */
	struct pushregs {
	    uint32_t reg_edi;
	    uint32_t reg_esi;
	    uint32_t reg_ebp;
	    uint32_t reg_oesp;          /* Useless */
	    uint32_t reg_ebx;
	    uint32_t reg_edx;
	    uint32_t reg_ecx;
	    uint32_t reg_eax;
	};

`trapframe`是中断帧，保存被中断或异常打断时的信息。tf_err, tf_eip, tf_cs, tf_eflags是中断产生时，硬件压入内核堆栈中的。由于执行中断服务例程时，可能会破坏相应的段寄存器或通用寄存器，所以需要保存这些寄存器用以恢复。当发生特权级切换时，保存esp和ss用于恢复堆栈信息。

## [练习2] 为新创建的内核线程分配资源 ##

#### 1. 简要说明你的设计实现过程 ####

`kern/process/proc.c`中的`do_fork`函数：

	/* do_fork -     parent process for a new child process
	 * @clone_flags: used to guide how to clone the child process
	 * @stack:       the parent's user stack pointer. if stack==0, It means to fork a kernel thread.
	 * @tf:          the trapframe info, which will be copied to child process's proc->tf
	 */
	int
	do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
	    int ret = -E_NO_FREE_PROC;
	    struct proc_struct *proc;
	    if (nr_process >= MAX_PROCESS) {
	        goto fork_out;
	    }
	    ret = -E_NO_MEM;
	    //LAB4:EXERCISE2 2013011377
	    /*
	     * Some Useful MACROs, Functions and DEFINEs, you can use them in below implementation.
	     * MACROs or Functions:
	     *   alloc_proc:   create a proc struct and init fields (lab4:exercise1)
	     *   setup_kstack: alloc pages with size KSTACKPAGE as process kernel stack
	     *   copy_mm:      process "proc" duplicate OR share process "current"'s mm according clone_flags
	     *                 if clone_flags & CLONE_VM, then "share" ; else "duplicate"
	     *   copy_thread:  setup the trapframe on the  process's kernel stack top and
	     *                 setup the kernel entry point and stack of process
	     *   hash_proc:    add proc into proc hash_list
	     *   get_pid:      alloc a unique pid for process
	     *   wakeup_proc:  set proc->state = PROC_RUNNABLE
	     * VARIABLES:
	     *   proc_list:    the process set's list
	     *   nr_process:   the number of process set
	     */
	
	    //    1. call alloc_proc to allocate a proc_struct
	    proc = alloc_proc();
	    if (proc == NULL)
	        goto fork_out;
	    //    2. call setup_kstack to allocate a kernel stack for child process
	    if (setup_kstack(proc)) 
	        goto bad_fork_cleanup_proc;
	    //    3. call copy_mm to dup OR share mm according clone_flag
	    if (copy_mm(clone_flags, proc)) 
	        goto bad_fork_cleanup_kstack;
	    //    4. call copy_thread to setup tf & context in proc_struct
	    copy_thread(proc, stack, tf);
	    //    5. insert proc_struct into hash_list && proc_list
	    bool intr_flag;
	    local_intr_save(intr_flag);
	    proc->pid = get_pid();
	    proc->parent = current;
	    hash_proc(proc);
	    list_add(&proc_list, &proc->list_link);
	    nr_process++;
	    local_intr_restore(intr_flag);
	    //    6. call wakup_proc to make the new child process RUNNABLE
	    wakeup_proc(proc);
	    //    7. set ret vaule using child proc's pid
	    ret = proc->pid;
	    
	fork_out:
	    return ret;
	
	bad_fork_cleanup_kstack:
	    put_kstack(proc);
	    
	bad_fork_cleanup_proc:
	    kfree(proc);
	    goto fork_out;
	}

在分配进程的`pid`时，为了防止多线程下出错（详见下一问题），需要关闭中断使能

#### 2. 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。 ####

ucore有给每个新fork的线程唯一的id，通过在`do_fork`中调用`get_pid`分配`pid`：

	// get_pid - alloc a unique pid for process
	static int
	get_pid(void) {
	    static_assert(MAX_PID > MAX_PROCESS);
	    struct proc_struct *proc;
	    list_entry_t *list = &proc_list, *le;
	    static int next_safe = MAX_PID, last_pid = MAX_PID;
	    if (++ last_pid >= MAX_PID) {
	        last_pid = 1;
	        goto inside;
	    }
	    if (last_pid >= next_safe) {
	    inside:
	        next_safe = MAX_PID;
	    repeat:
	        le = list;
	        while ((le = list_next(le)) != list) {
	            proc = le2proc(le, list_link);
	            if (proc->pid == last_pid) {
	                if (++ last_pid >= next_safe) {
	                    if (last_pid >= MAX_PID) {
	                        last_pid = 1;
	                    }
	                    next_safe = MAX_PID;
	                    goto repeat;
	                }
	            }
	            else if (proc->pid > last_pid && next_safe > proc->pid) {
	                next_safe = proc->pid;
	            }
	        }
	    }
	    return last_pid;
	}

可以发现`get_pid`中，每次循环进程链表维护一个`[last_pid, next_safe)`区间，保证该区间内没有已分配的id。若区间长度缩小为0或负数，则向右移动左区间`last_pid`，进行下一次遍历进程链表循环。总而言之，`get_pid`会返回最小的未分配的id。

其次，`get_pid`寻找未分配的id时，需要进程链表保持不变，所以它是不支持多线程安全的，所以在上一小问分配id和加入进程链表时，需要暂时关闭中断。

## [练习3] 阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的 ##

#### 1. 对proc_run函数的分析 ####

	// proc_run - make process "proc" running on cpu
	// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
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

首先关闭中断，然后把`proc`赋值给全局变量`current`（当前进程）。通过`load_esp0`和`lcr3`加载新的`esp`和`cr3`，实现内核栈和页表的切换。最后通过`switch_to`切换进程的上下文：

	.text
	.globl switch_to
	switch_to:                      # switch_to(from, to)
	
	    # save from's registers
	    movl 4(%esp), %eax          # eax points to from
	    popl 0(%eax)                # save eip !popl
	    movl %esp, 4(%eax)
	    movl %ebx, 8(%eax)
	    movl %ecx, 12(%eax)
	    movl %edx, 16(%eax)
	    movl %esi, 20(%eax)
	    movl %edi, 24(%eax)
	    movl %ebp, 28(%eax)
	
	    # restore to's registers
	    movl 4(%esp), %eax          # not 8(%esp): popped return address already
	                                # eax now points to to
	    movl 28(%eax), %ebp
	    movl 24(%eax), %edi
	    movl 20(%eax), %esi
	    movl 16(%eax), %edx
	    movl 12(%eax), %ecx
	    movl 8(%eax), %ebx
	    movl 4(%eax), %esp
	
	    pushl 0(%eax)               # push eip
	
	    ret

最终通过将新进程应执行的代码位置`eip`压入栈中当做`Return Address`，通过调用`ret`切换到对应代码处。

#### 2. 在本实验的执行过程中，创建且运行了几个内核线程？ ####

两个内核线程，分别是`idleproc`和`initproc`。

#### 3. 语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`在这里有何作用?请说明理由 ####

两语句分别是关中断和开中断。比如在保存上下文时若被中断打断，则保存的上下文只有一部分是正确的，就会导致错误。

## 我的实现与参考答案的区别 ##

具体实现上有些许区别，但实现的功能大体相同：

如`alloc_proc`中我通过`memset(proc, 0, sizeof(struct proc_struct))`清空PCB，实现简单快速。

## 重要知识点 ##
1. `proc_struct(PCB/TCB)`各成员变量的含义与作用
2. 进程/线程的运行状态与生命周期
3. 多线程安全与暂关中断的必要性