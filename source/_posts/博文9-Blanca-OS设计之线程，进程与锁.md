---
title: 'Blanca-OS设计之线程，进程与锁'
date: 2020-04-22
tags: 
- 操作系统
---

### 线程与进程
线程与进程都是程序执行流，其区别在于进程拥有独立的地址空间与一些资源，而线程则需与其所属进程中的其他的线程共享地址空间与一些资源。Blanca-OS中没有像早期的linux一样将所有程序执行流统一为任务（Task）的概念，而是保留了线程与进程之分，在实现上首先完成了线程模块，并依托于此完成了进程模块。


### 程序控制块（PCB）
Blanca-OS管理线程与线程时统一使用了程序控制块（PCB），程序控制块结构体task_struct设计如下（见`inc/thread.h`）：

```c
typedef struct{
	uint8_t* context;
	uint8_t status;
	uint8_t priority;
	uint32_t run_time;
	list_node ready_list_node;
	list_node all_list_node;
	char name[16];
	uint32_t* pgdir;
	uint32_t stack_boundary;
}task_struct;
```

（待完善）
