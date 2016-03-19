# Lab2 Report#
---
## [练习0] 填写已有实验 ##

使用`meld`工具辅助人工检查完成。

## [练习1] 实现 first-fit 连续物理内存分配算法 ##

#### 1. 简要说明你的设计实现过程 ####

`kern/mm/default_pmm.c`中的`default_alloc_pages`负责找到第一个大等于n页的空闲连续块并分配并切分（如必要）：

	static struct Page *
	default_alloc_pages(size_t n) {
	    assert(n > 0);
	    if (n > nr_free) {
	        return NULL;
	    }
	    struct Page *page = NULL;
	    list_entry_t *le = &free_list;
	    while ((le = list_next(le)) != &free_list) {
	        struct Page *p = le2page(le, page_link);
	        if (p->property >= n) {
	            page = p;
	            break;
	        }
	    }
	    if (page != NULL) {
	        struct Page *p = page;
	        for (; p != page + n; p++) {
	            ClearPageProperty(p);
	            SetPageReserved(p);
	        }
	        if (page->property > n) {
	            struct Page *newPage = page + n;
	            newPage->property = page->property - n;
	            page->property = n;
	            list_add(&page->page_link, &(newPage->page_link));
	        }
	        nr_free -= n;
	        list_del(&(page->page_link));
	    }
	    return page;
	}


`kern/mm/default_pmm.c`中的`default_free_pages`负责将base页开始的n页加回双向循环链表并合并相邻空闲块（如必要）：

	static void
	default_free_pages(struct Page *base, size_t n) {
	    assert(n > 0);
	    struct Page *p = base;
	    for (; p != base + n; p ++) {
	        p->flags = p->property = 0;
	        set_page_ref(p, 0);
	    }
	    base->property = n;
	    SetPageProperty(base);
	
	    list_entry_t *le = list_next(&free_list);
	    while (le != &free_list) {
	        if (le2page(le, page_link) > base)
	            break;
	        le = list_next(le);
	    }
	    list_add_before(le, &base->page_link);
	
	    le = list_next(&base->page_link);
	    p = le2page(le, page_link);
	    if (le != &free_list && base + base->property == p) {
	        base->property += p->property;
	        ClearPageProperty(p);
	        list_del((&p->page_link));
	    }
	
	    le = list_prev(&base->page_link);
	    p = le2page(le, page_link);
	    if (le != &free_list && p + p->property == base) {
	        p->property += base->property;
	        ClearPageProperty(base);
	        list_del((&base->page_link));
	    }
	
	    nr_free += n;
	}

在我的实现中，不区分初始化和释放操作，所以我删掉了`default_init_memmap`将`default_pmm_manager`中的函数指针`.init_memmap`与`.free_pages`一同指向`default_free_pages`：

	const struct pmm_manager default_pmm_manager = {
	    .name = "default_pmm_manager",
	    .init = default_init,
	    .init_memmap = default_free_pages,
	    .alloc_pages = default_alloc_pages,
	    .free_pages = default_free_pages,
	    .nr_free_pages = default_nr_free_pages,
	    .check = default_check,
	};


#### 2. 你的first fit算法是否有进一步的改进空间 ####

有。可以使用平衡树或分块策略代替循环双向循环链表维护空闲页块，加速分配和释放时的定位操作。

## [练习2] 实现寻找虚拟地址对应的页表项 ##