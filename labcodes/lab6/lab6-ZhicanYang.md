# Lab6 Report#
---
## [练习0] 填写已有实验 ##

使用`meld`工具辅助人工检查完成。

## [练习1] 使用 Round Robin 调度算法 ##

#### 1. 理解并分析sched_class中各个函数指针的用法，并结合Round Robin调度算法分析ucore的调度执行过程 ####

`kern/schedule/sched.h`中`sched_class`的定义：
	
	// The introduction of scheduling classes is borrrowed from Linux, and makes the 
	// core scheduler quite extensible. These classes (the scheduler modules) encapsulate 
	// the scheduling policies. 
	struct sched_class {
	    // the name of sched_class
	    const char *name;
	    // Init the run queue
	    void (*init)(struct run_queue *rq);
	    // put the proc into runqueue, and this function must be called with rq_lock
	    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
	    // get the proc out runqueue, and this function must be called with rq_lock
	    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
	    // choose the next runnable task
	    struct proc_struct *(*pick_next)(struct run_queue *rq);
	    // dealer of the time-tick
	    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
	    /* for SMP support in the future
	     *  load_balance
	     *     void (*load_balance)(struct rq* rq);
	     *  get some proc from this rq, used in load_balance,
	     *  return value is the num of gotten proc
	     *  int (*get_proc)(struct rq* rq, struct proc* procs_moved[]);
	     */
	};

函数指针在`kern/schedule/default_sched.c`中配置对应的具体实现：

	struct sched_class default_sched_class = {
	    .name = "RR_scheduler",
	    .init = RR_init,
	    .enqueue = RR_enqueue,
	    .dequeue = RR_dequeue,
	    .pick_next = RR_pick_next,
	    .proc_tick = RR_proc_tick,
	};

* `init`在`sched_init`时调用，即在创建进程之前
* `enqueue`用于将进程加入就绪队列
* `dequeue`用于将进程从就绪队列中删除
* `pick_next`用于从就绪队列中选择下一个将执行的进程并返回，在进程切换时调用
* `proc_tick`在产生时钟中断时调用，如在`Round Robin`算法中会更新当前进程的剩余时间片数量，若为零则其置为需要调度`proc->need_resched = 1`

#### 2. 简要说明如何设计实现”多级反馈队列调度算法“ ####

多级反馈队列算法(MLFQ)是一种进程可在不同队列间移动的多级队列算法，其目的是使得CPU密集型进程的时间片更大，使I/O密集型进程的时间片较小，以提高CPU的利用效率。

具体实现中，给同一种优先级的进程维护一个进程队列，在每一个进程队列中，可以使用不同的调度算法，比如`Round Robin`、`Stride`等。然后维护一个队列管理这些进程队列。

设优先级最高(1)的进程的时间片大小为`t0`，则优先级为p的进程的时间片大小为`p*t0`。

如果一个进程在当前的时间片结束时没有完成，则将其降一个优先级。

这样一来，I/O密集型进程经常会在时间片用完之前就会触发中断，会停留在较高的优先级；而CPU密集型进程常常会使用完时间片，而降级分到更大的时间片，进而减少调度次数。



## [练习2] 实现 Stride Scheduling 调度算法 ##

#### 1. 简要说明你的设计实现过程 ####

依据宏`USE_SKEW_HEAP`的定义分别实现了使用队列以及使用斜堆的`Stride`调度算法

	/* You should define the BigStride constant here*/
	/* LAB6: 2013011377 */
	#define BIG_STRIDE 0x7FFFFFFF   /* you should give a value, and is ??? */

在ucore中使用如下语句进行步长值大小的判断，其中`lab6_stride`是步长值：

	if ((int32_t)(p->lab6_stride - q->lab6_stride) > 0)
		p = q;

在步长值可能溢出的情况下，上式仍然能够正确判断。原因如下：

首先，任意两个步长值的差距不会超过`BIG_STRIDE`，因为当前步长值最小的进程会被调度，而最大的步进值就是`BIG_STRIDE`：

	#define BIG_STRIDE    0x7FFFFFFF /* ??? */

`BIG_STRIDE`是`2^32 / 2 - 1`，也即无符号32位整型的一半减一。

* 若p和q均未溢出，判断式自然正确
* 若p溢出，q未溢出，此时有p > q，且p最高位为0，q最高位为1(p和q差值至多为32位一半减一)，此时p-q的最高位为0，强转为32位有符号整型的结果为正数，判断出p > q。
* 若p未溢出，q溢出，此时有p < q，且p最高位为1，q最高位为0(p和q差值至多为32位一半减一)，此时p-q的最高位为1，强转为32位有符号整型的结果为负数，判断出p < q。
* 若p和q均溢出，此时两者的溢出次数要么相同，要么相差一(p和q差值至多为32位一半减一)，相当于两者模2^32，还原成前3种情况，能正确判断出p,q的大小关系。

各函数的实现：

	static void
	stride_init(struct run_queue *rq) {
	     /* LAB6: 2013011377
	      * (1) init the ready process list: rq->run_list
	      * (2) init the run pool: rq->lab6_run_pool
	      * (3) set number of process: rq->proc_num to 0       
	      */
	#if USE_SKEW_HEAP
	    rq->lab6_run_pool = NULL;
	#else
	    list_init(&rq->run_list);
	#endif
	    rq->proc_num = 0;   
	}

	static void
	stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
	     /* LAB6: 2013011377 
	      * (1) insert the proc into rq correctly
	      * NOTICE: you can use skew_heap or list. Important functions
	      *         skew_heap_insert: insert a entry into skew_heap
	      *         list_add_before: insert  a entry into the last of list   
	      * (2) recalculate proc->time_slice
	      * (3) set proc->rq pointer to rq
	      * (4) increase rq->proc_num
	      */
	#if USE_SKEW_HEAP
	    rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
	#else
	    list_add_before(&rq->run_list, &proc->run_link);
	#endif
	    if (proc->time_slice <= 0 || proc->time_slice > rq->max_time_slice)
	        proc->time_slice = rq->max_time_slice;
	    proc->rq = rq;
	    rq->proc_num++;
	}

	static void
	stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
	     /* LAB6: 2013011377 
	      * (1) remove the proc from rq correctly
	      * NOTICE: you can use skew_heap or list. Important functions
	      *         skew_heap_remove: remove a entry from skew_heap
	      *         list_del_init: remove a entry from the  list
	      */
	#if USE_SKEW_HEAP
	    rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
	#else
	    list_del(&proc->run_link);
	#endif
	    rq->proc_num--;
	}

	static struct proc_struct *
	stride_pick_next(struct run_queue *rq) {
	     /* LAB6: 2013011377 
	      * (1) get a  proc_struct pointer p  with the minimum value of stride
	             (1.1) If using skew_heap, we can use le2proc get the p from rq->lab6_run_poll
	             (1.2) If using list, we have to search list to find the p with minimum stride value
	      * (2) update p;s stride value: p->lab6_stride
	      * (3) return p
	      */
	    struct proc_struct *p = NULL;
	#if USE_SKEW_HEAP
	    if(!rq->lab6_run_pool) return NULL;
	    p = le2proc(rq->lab6_run_pool, lab6_run_pool);
	#else
	    list_entry_t *le = list_next(&rq->run_list);
	    while ((le = list_next(le)) != &rq->run_list) {
	        struct proc_struct *cnt = le2proc(le, run_link);
	        if (!p || proc_stride_comp_f(&p->lab6_run_pool, &cur->lab6_run_pool))
	            p = cur;
	    }
	#endif
	    if (p)
	        p->lab6_stride += BIG_STRIDE / (p->lab6_priority == 0? 1: p->lab6_priority);
	    return p;
	}

	static void
	stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
	     /* LAB6: 2013011377 */
	    if (--proc->time_slice <= 0)
	        proc->need_resched = 1;
	}



## 我的实现与参考答案的区别 ##

在此前的Lab1和Lab4已经使用`memset`初始化了各项，处理了相对应的边界情况，所以在Lab6不需要做改动。

若进程的优先级为0，答案会将其置为1。而我的实现是保留其优先级为0，在做除法时将其按照1来计算，更贴近原设计。

在参考答案的链表实现中，没有复用`proc_stride_comp_f`而是又写了一份比较的代码，属于冗余，也不易于进行修改。

在`stride_proc_tick`中，不需要判断`proc->time_slice > 0`，我的实现更简洁效率。

## 重要知识点 ##
1. `Round Robin`调度算法与`Stride`调度算法
2. 函数指针的作用与实现
3. 进程/线程优先级的作用
4. 多级反馈队列调度算法