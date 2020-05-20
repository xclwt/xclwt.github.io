---
title: 'Blanca-OS设计之中断'
date: 2020-04-28
tags: 
- 操作系统
---

## 中断

中断，即CPU接收到一定的信号，打断了当前的程序控制流以处理其他事务的机制。
中断主要可分为两类：同步中断与异步中断。
其中，同步中断又称异常，可分为故障（fault），陷阱（trap），终止（abort）三类。异步中断则简称为中断。

| 类别 |      |      |
| ---- | ---- | ---- |
| 中断 |      |      |
| 故障 |      |      |
| 陷阱 |      |      |
| 终止 |      |      |



## 中断例程

Blanca-OS中的所有中断处理程序都要经过一段汇编实现的例程asm_isr_stub（见`kernel/idt_asm.asm`），该例程将中断将中断发生时的程序信息（主要是寄存器）快照压栈并调用中断号对应的中断处理程序，该快照在C语言中以如下结构体表示（见inc/idt.h）：

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

 需要注意的是，并非所有的中断都会压入error code（错误代码），故中断发生时的程序快照结构体本应有两种，但此处为了方便后续程序的编写进行了统一，即在汇编macro生成的不压入错误代码的相关中断的处理程序中(见`kernel/idt_s.asm`)自动在相应位置压入32字节数据，该数据值统一为0。