---
title: 'Blanca-OS设计之线程，进程与锁'
date: 2020-04-22
tags: 
- 操作系统
---

## 线程与进程
线程与进程都是程序执行流，其区别在于进程拥有独立的地址空间与一些资源，而线程则需与其所属进程中的其他的线程共享地址空间与一些资源。Blanca-OS中参考了linux中的实现，采用了”伪“线程的概念，将所有程序执行流（每一个线程/进程都分配一个PCB）统一归为任务（Task）进行调度。


## 程序控制块（PCB）
Blanca-OS管理线程与线程时统一使用了程序控制块（PCB），程序控制块结构体task_struct设计如下（见`inc/thread.h`）：

```c
typedef struct{
	uint8_t* kstack;
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

严格来说，PCB并不仅限于这个结构体，还包含了kstack所指向的内核线程栈，整个PCB的内存结构如下图：

（待添加）

## 程序调度

#### 程序切换

程序切换的操作由汇编实现（见thread/switch.asm），该操作保存了当前程序的上下文并切换到下一个程序去。

```asm
switch_to:
	mov eax,[esp + 4]
	mov [eax],esp
	mov eax,[esp + 8]

	push esi
	push edi
	push ebx
	push ebp

	mov esp,[eax]

	pop ebp
	pop ebx
	pop edi
	pop esi

	ret
```

switch_to函数按照ABI将上下文（esi，edi，ebx，ebp寄存器的内容，esp寄存器值由调用约定保存）保存在当前程序的内核栈中，并切换出下一个程序的上下文，最后通过ret切换到下一个程序。

#### 调度算法

## 锁

现阶段，Blanca-OS仅支持单处理器，故目前对于共享资源只提供了互斥锁（mutex）这一种机制以避免竞争。互斥锁的原理即将抢锁失败的任务阻塞并等待锁的释放。其实现主要采用了`semaphore`与`mutex_lock`两种数据结构（见`inc/sync.h`），锁的获取与释放代码如下:

```c
void acquire_mutex(mutex_lock* mutex){
	if(mutex->holder != cur_thread()){
		sema_busy(&(mutex->sig));		//信号量设为忙碌
		mutex->holder = cur_thread();	//将互斥锁持有者设为当前任务
		mutex->request_num = 1;			//初始化互斥锁申请次数为1
	}else{
		++mutex->request_num;			//互斥锁申请次数加1
	}
}

void release_mutex(mutex_lock* mutex){
	if(mutex->request_num > 1){
		--mutex->request_num;			//互斥锁申请次数减1
		return;
	}else{
		--mutex->reqauest_num;			
		mutex->holder = NULL;			//清空持有者
		sema_free(&(mutex->sig));		//信号量设为空闲
	}
}
```

