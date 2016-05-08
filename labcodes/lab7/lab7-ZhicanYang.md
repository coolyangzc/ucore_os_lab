# Lab7 Report#
---
## [练习0] 填写已有实验 ##

使用`meld`工具辅助人工检查完成。

## [练习1] 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题 ##

#### 1. 给出内核级信号量的设计描述，并说明其大致执行流程 ####

内核级信号量`semaphore_t`的定义在`kern/sync/sem.h`中：

	typedef struct {
	    int value;
	    wait_queue_t wait_queue;
	} semaphore_t;

其中`value`是信号量用于计数的整数值，`wait_queue`是进程等待队列，所有等待此信号量的进程会挂在该等待队列上。

`ucore`中提供与信号量相关的四种操作：

	void sem_init(semaphore_t *sem, int value);
	void up(semaphore_t *sem);
	void down(semaphore_t *sem);
	bool try_down(semaphore_t *sem);

* `sem_init`是初始化操作，设置信号量`sem`的初始值并新建一个空的等待队列
* `up`即V操作。首先关中断进行保护，如果该信号量对应的等待队列为空，将信号量的`value`加一后即可开中断返回。如果等待队列非空，则调用`wakeup_wait`函数将等待队列中的第一个进程从队列中删除，然后唤醒，最后开中断返回。
* `down`即P操作。首先关中断进行保护。如果当前信号量的`value`大于0，表明可以获得该信号量，此时将`value`值减一，并开中断返回。若当前信号量的`value`小等于0，则表明无法获得该信号量，将当前进程加入到该信号量的等待队列中，并开中断，调用`schedule`函数让调度器选择其他进程执行。此后，若该进程被其他进程的`up(V)`操作唤醒，把自身关联的`wait`从等待队列中删除。
* `try_down`是非阻塞的P操作，即尝试直接将信号量的`value`值减一。如果信号量大于0，则成功减一并返回true；如果信号量小等于0，则返回false。

#### 2. 给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同 ####

用户态的进程或线程想要使用信号量机制，则必须需要中断的开关保护或者相对应的原子操作。无论哪种方式都需要特权指令，所以通过系统调用来实现是最自然且合理的。

增加一个与信号量相关的系统调用`SYS_SEM`，该系统调用的第一个参数指定操作类型：初始化/P/V/非阻塞P操作；第二个参数指定操作的信号量的地址；若为初始化操作，则还有第三个参数，指定需初始化信号量的`value`初值。若为非阻塞P操作，该系统调用的返回值是有意义的：`false/0`表示操作失败；`true/1`表示操作成功。

用户态进程/线程信号量机制与内核级信号量机制的异同：

* 相同：实现机制相同，提供的操作相同，甚至于可以设计相似的接口。
* 不同：用户态使用信号量机制需要借助系统调用实现，涉及用户态和核心态之间的转换；而内核态不需要，两者的执行效率有差别。

## [练习2] 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题 ##

#### 1. 给出内核级条件变量的设计描述，并说明其大致执行流程 ####

内核级条件变量`condvar_t`的定义在`kern/sync/monitor.h`中：

	typedef struct condvar{
	    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
	    int count;              // the number of waiters on condvar
	    monitor_t * owner;      // the owner(monitor) of this condvar
	} condvar_t;

其中`sem`是信号量，用于复用等待队列等；`count`是等待该条件变量的进程数；`owner`是该条件变量所属于的管程。

`ucore`中提供的与条件变量相关的两个方法：

	// Unlock one of threads waiting on the condition variable. 
	void     cond_signal (condvar_t *cvp);
	// Suspend calling thread on a condition variable waiting for condition atomically unlock mutex in monitor,
	// and suspends calling thread on conditional variable after waking up locks mutex.
	void     cond_wait (condvar_t *cvp);

* `cond_signal`操作使等待条件变量`cvp`的某一个进程唤醒，即在`cvp`的成员变量`sem`中的等待队列中取出一个正在等待该条件变量的进程唤醒。
* `cond_wait`操作使当前进程等待该条件变量`cvp`，根据条件变量`cvp`的情况直接运行或进入等待队列并被其他进程/线程唤醒。

#### 2. 给出用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同 ####

同信号量一样，用户态的进程或线程想要使用条件便利那个机制，也必须需要中断的开关保护或者相对应的原子操作。无论哪种方式都需要特权指令，所以最终通过系统调用来实现是最自然且合理的。

增加一个与条件变量相关的系统调用`SYS_CV`，该系统调用的第一个参数指定操作类型：signal/wait；第二个参数指定操作的条件变量的地址。

用户态进程/线程条件变量机制与内核级条件变量机制的异同：

* 相同：实现机制相同，提供的操作相同，甚至于可以设计相似的接口。
* 不同：用户态使用条件变量机制需要借助系统调用实现，涉及用户态和核心态之间的转换；而内核态不需要，两者的执行效率有差别。

## 我的实现与参考答案的区别 ##

在此前的Lab1和Lab4初始化中，已经使用`memset`了清零各项，处理了相对应的边界情况，所以在本Lab中不需要做改动。

我使用的是Lab6中是实现的`Stride Scheduling`调度算法，而答案使用的仍是`Round Robin`调度算法。这样在触发时间中断时，需要调用`Stride Scheduling`算法中的`sched_proc_tick()`。

## 重要知识点 ##

1. 同步与互斥的异同点
2. 信号量/条件变量的原理与实现
3. 管程与条件变量的关系

未在原理中提到的：

1. 条件变量中的等待队列复用了信号量中的实现，所以条件变量的定义中有一个信号量。 
2. `try_down`方法，也即非阻塞的P操作，可以减少信号量的值。这样可以更灵活地使用信号量，比如某个打印机损坏的话，就直接将打印操作有关的信号量直接减一。