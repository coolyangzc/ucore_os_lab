# Lab3 Report#
---
## [练习0] 填写已有实验 ##

使用`meld`工具辅助人工检查完成。

## [练习1] 给未被映射的地址映射上物理页 ##

#### 1. 简要说明你的设计实现过程 ####

`kern/mm/vmm.c`中的`do_pgfault`函数是页访问异常处理函数，供中断服务例程调用：

	int
	do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr) {
	    int ret = -E_INVAL;
	    //try to find a vma which include addr
	    struct vma_struct *vma = find_vma(mm, addr);
	
	    pgfault_num++;
	    //If the addr is in the range of a mm's vma?
	    if (vma == NULL || vma->vm_start > addr) {
	        cprintf("not valid addr %x, and  can not find it in vma\n", addr);
	        goto failed;
	    }
	    //check the error_code
	    switch (error_code & 3) {
	    default:
	            /* error code flag : default is 3 ( W/R=1, P=1): write, present */
	    case 2: /* error code flag : (W/R=1, P=0): write, not present */
	        if (!(vma->vm_flags & VM_WRITE)) {
	            cprintf("do_pgfault failed: error code flag = write AND not present, but the addr's vma cannot write\n");
	            goto failed;
	        }
	        break;
	    case 1: /* error code flag : (W/R=0, P=1): read, present */
	        cprintf("do_pgfault failed: error code flag = read AND present\n");
	        goto failed;
	    case 0: /* error code flag : (W/R=0, P=0): read, not present */
	        if (!(vma->vm_flags & (VM_READ | VM_EXEC))) {
	            cprintf("do_pgfault failed: error code flag = read AND not present, but the addr's vma cannot read or exec\n");
	            goto failed;
	        }
	    }
	    /* IF (write an existed addr ) OR
	     *    (write an non_existed addr && addr is writable) OR
	     *    (read  an non_existed addr && addr is readable)
	     * THEN
	     *    continue process
	     */
	    uint32_t perm = PTE_U;
	    if (vma->vm_flags & VM_WRITE) {
	        perm |= PTE_W;
	    }
	    addr = ROUNDDOWN(addr, PGSIZE);
	
	    ret = -E_NO_MEM;
	
	    pte_t *ptep=NULL;
	    /*LAB3 EXERCISE 1: 2013011377
	    * Maybe you want help comment, BELOW comments can help you finish the code
	    *
	    * Some Useful MACROs and DEFINEs, you can use them in below implementation.
	    * MACROs or Functions:
	    *   get_pte : get an pte and return the kernel virtual address of this pte for la
	    *             if the PT contians this pte didn't exist, alloc a page for PT (notice the 3th parameter '1')
	    *   pgdir_alloc_page : call alloc_page & page_insert functions to allocate a page size memory & setup
	    *             an addr map pa<--->la with linear address la and the PDT pgdir
	    * DEFINES:
	    *   VM_WRITE  : If vma->vm_flags & VM_WRITE == 1/0, then the vma is writable/non writable
	    *   PTE_W           0x002                   // page table/directory entry flags bit : Writeable
	    *   PTE_U           0x004                   // page table/directory entry flags bit : User can access
	    * VARIABLES:
	    *   mm->pgdir : the PDT of these vma
	    *
	    */
	    /*LAB3 EXERCISE 1: 2013011377*/
	    ptep = get_pte(mm->pgdir, addr, 1);              //(1) try to find a pte, if pte's PT(Page Table) isn't existed, then create a PT.
	    if (!ptep)
	        goto failed;
	    if (*ptep == 0) {
	                                                     //(2) if the phy addr isn't exist, then alloc a page & map the phy addr with logical addr
	        struct Page *page = pgdir_alloc_page(mm->pgdir, addr, perm);
	        if (!page)
	            goto failed;
	    }
	    else {
	    /*LAB3 EXERCISE 2: YOUR CODE
	    * Now we think this pte is a  swap entry, we should load data from disk to a page with phy addr,
	    * and map the phy addr with logical addr, trigger swap manager to record the access situation of this page.
	    *
	    *  Some Useful MACROs and DEFINEs, you can use them in below implementation.
	    *  MACROs or Functions:
	    *    swap_in(mm, addr, &page) : alloc a memory page, then according to the swap entry in PTE for addr,
	    *                               find the addr of disk page, read the content of disk page into this memory page
	    *    page_insert ： build the map of phy addr of an Page with the linear addr la
	    *    swap_map_swappable ： set the page swappable
	    */
	        if(swap_init_ok) {
	            struct Page *page=NULL;
	                                    //(1) According to the mm AND addr, try to load the content of right disk page
	                                    //    into the memory which page managed.
	                                    //(2) According to the mm, addr AND page, setup the map of phy addr <---> logical addr
	                                    //(3) make the page swappable.
	            ret = swap_in(mm, addr, &page);
	            if (ret != 0)
	                goto failed;
	            page_insert(mm->pgdir, page, addr, perm);
	            swap_map_swappable(mm, addr, page, 1);
	            page->pra_vaddr = addr;
	        }
	        else {
	            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
	            goto failed;
	        }
	   }
	   ret = 0;
	failed:
	    return ret;
	}


#### 2. 请描述页目录项（Page Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处 ####

主要用处是提供线性地址到物理地址的转换，潜在用处是筛出非法访问的页面。

#### 3. 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？ ####

插入中断处理，根据IDT和GDT再次调用缺页服务例程。

## [练习2] 补充完成基于FIFO的页面替换算法 ##

#### 1. 简要说明你的设计实现过程 ####

`kern/mm/swap_fifo.c`中的`_fifo_map_swappable`和`_fifo_swap_out_victim`：

	static int
	_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
	{
	    list_entry_t *head=(list_entry_t*) mm->sm_priv;
	    list_entry_t *entry=&(page->pra_page_link);
	 
	    assert(entry != NULL && head != NULL);
	    //record the page access situlation
	    /*LAB3 EXERCISE 2: 2013011377*/ 
	    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
	    list_add_after(head, entry);
	    return 0;
	}
	/*
	 *  (4)_fifo_swap_out_victim: According FIFO PRA, we should unlink the  earliest arrival page in front of pra_list_head qeueue,
	 *                            then set the addr of addr of this page to ptr_page.
	 */
	static int
	_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
	{
	     list_entry_t *head=(list_entry_t*) mm->sm_priv;
	         assert(head != NULL);
	     assert(in_tick==0);
	     /* Select the victim */
	     /*LAB3 EXERCISE 2: 2013011377*/ 
	     //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
	     //(2)  set the addr of addr of this page to ptr_page
	     list_entry_t *le = list_prev(head);
	     assert(le != head);
	     *ptr_page = le2page(le, pra_page_link);
	     assert(ptr_page != NULL);
	     list_del(le);
	     return 0;
	}

#### 2. 如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题 ####
* 需要被换出的页的特征是什么？
* 在ucore中如何判断具有这样特征的页？
* 何时进行换入和换出操作？ 

现有的`swap_manager`框架并不足以支持`extended clock`页替换算法，因为只有`_fifo_map_swappable`、`_fifo_swap_out_victim`而并没有供访问页面时调用的函数。需要在此基础上增加访问与修改页面时调用的函数，在该函数中修改访问/修改页面的访问位或修改位。

* 被时钟算法的指针扫过的页会渐渐清掉修改位和访问位，而需要被换出的页是第一个被时钟算法指针扫到的修改位与访问位都为0的页，这样的页大概率是短期内没有被修改或访问过的页。
* 在ucore中维护一个时钟指针并在做`_fifo_swap_out_victim`时进行处理
* 在触发页访问异常是进行换入换出操作，也即被动页替换。

## 我的实现与参考答案的区别 ##

在`kern/mm/swap_fifo.c`的`_fifo_swap_out_victim`中，我得到被替换页的语句是：
	
	list_entry_t *le = list_prev(head);

而参考答案的实现是：

	list_entry_t *le = head->prev;

相对来说我的实现更加抽象安全，也更符合`list_entry_t`作为通用数据结构存在的思路。

## 重要知识点 ##
1. 页替换算法（尤其是改进时钟算法）
2. `mm_struct`和`vma_struct`之间的关系，以及如何使用`list_entry_t`访问它们
3. `Page Fault`异常中的`Error Code`含义以及对应处理方式