---
title: 'Blanca-OS设计之中断'
date: 2020-04-28
tags: 
- 操作系统
---

## 中断例程

Blanca-OS中的所有中断处理程序都要经过一段汇编实现的例程`c_isr_stub`（见`kernel/idt_asm.asm`），该例程将中断将中断发生时的程序信息（主要是寄存器）快照压栈，该快照在C语言中以如下结构体表示（见inc/idt.h）：

```c
typedef struct{
	/*段寄存器,16bit*/
	uint32_t fs;
	uint32_t gs;
	uint32_t es;
	uint32_t ds;
	
	/*pusha保存的寄存器*/
	uint32_t edi;
	uint32_t esi;
	uint32_t ebp;
	uint32_t esp;
	uint32_t ebx;
	uint32_t edx;
	uint32_t ecx;
	uint32_t eax;

	/*中断号*/
	uint32_t intr_num;

	/*错误代码*/
	uint32_t err_code;

	uint32_t eip;
	uint32_t cs; //16bit
	uint32_t eflags;

	/*特权级切换时才会压入ss及esp*/
	uint32_t u_esp;
	uint32_t ss; //16bit
}regs_t;
```

