---
title: 'Blanca-OS设计之内存管理'
date: 2020-04-12
tags: 
- 操作系统
---

### 内存分布

Blanca-OS假定使用者的内存大于1G，并将1G的内核空间永远映射于地址的3/4高位之上（3GB-4GB），而超出1G内存的部分则被分配为用户空间并映射于地址的低位（0-3GB）。内核空间参考linux的设计分为了3个zone：DMA ZONE（0-1MB），NORMAL ZONE（1-896MB），HIGHMEM ZONE（896MB+）。Blanca-OS目前的内存管理模块主要聚焦于对NORMAL ZONE的管理，其他zone的管理与使用需等待后期的开发。



### 页管理结构

Blanca-OS对页的管理主要使用了以下的结构：

```c
typedef struct{
	list_node list;
	atomic_t count;
	uint32_t order;
	uint32_t flag;
}page_t;
```

页管理结构体page_t（见`inc/pmm.h`）：

每一个页管理结构体存储了对应的一个或多个页的信息，包括：list（用于添加至空闲页表以标记每一个或一组空闲页），count（当前页引用计数），order（当前页管理结构体管理的页数为2的order次幂），flag（当前页状态）。

```c
static page_t* p_pages = (page_t*)(uint32_t)KERNEL_END + KERNEL_BASE;
```

指向页管理结构体数组的指针p_pages（见`mm/pmm.c`）:该页管理结构体数组中记录了NORMAL ZONE中所有页对应的页管理结构体。

```c
static list_node free_list[MAX_ORDER + 1];
```

空闲页表free_list（见`mm/buddy.c`）：空闲页表记录了每一个order的空闲页集合所对应的链表头，可根据表头查找出一组所需大小空闲页对应的list值并反向确定该组起始空闲页的位置。



### 页管理算法

选用了伙伴算法：
（updating...）