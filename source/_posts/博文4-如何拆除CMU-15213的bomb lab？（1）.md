---
title: '如何拆除CMU-15213的bomb lab？（1）'
date: 2019-09-15
tags: 
- 汇编
- GDB
---


###  phase 1：

用gdb调试器打开bomb程序：

```shell
gdb bomb
```

  <!--more-->

显示汇编窗口以方便调试：

```c
layout asm
```

此时我们可以看到bomb程序的反汇编代码，并在main函数中找到phase_1至phase_6这6个函数的调用，这应该对应着炸弹的六个关卡：

```c
0x400e3a <main+154>     callq  0x400ee0 <phase_1>                                         
0x400e3f <main+159>     callq  0x4015c4 <phase_defused>
......
0x400ec6 <main+294>     callq  0x4010f4 <phase_6>                                        
0x400ecb <main+299>     callq  0x4015c4 <phase_defused> 
```

执行以下操作：

```c
break * 0x400e3a 	# 在phase_1函数调用处加断点
run					# 运行程序
(输入任意字符串)
stepi				# 单步执行，进入phase_1函数
```

进入phase_1函数之后我们可以看到对strings_not_equal函数的调用：

```c
0x400ee9 <phase_1+9>    callq  0x401338 <strings_not_equal>  
0x400eee <phase_1+14>   test   %eax,%eax                                                  
0x400ef0 <phase_1+16>   je     0x400ef7 <phase_1+23>   
0x400ef2 <phase_1+18>   callq  0x40143a <explode_bomb>  
```

当strings_not_equal的返回值为0时，将跳过explode_bomb，否则将执行explode_bomb，炸弹就会爆炸。

而在strings_not_equal函数调用之前，我们可以看到一条指令：

```c
0x400ee4 <phase_1+4>    mov    $0x402400,%esi      
```

这条指令将一个即时数传入了寄存器%esi，或许这是strings_not_equal函数的参数之一，我们看看这个地址里有什么，输入：

```shell
x/s 0x402400
```

得到结果：

```c
0x402400:       "Border relations with Canada have never been better."
```

显然，这是phase 1用来与我们输入字符串比对的密码字符串。至此，phase 1通过。



### phase 2：

执行以下操作：

```c
break * 0x400e56	# 在phase_2函数调用处加断点
......
(输入任意字符串)
stepi 				# 单步执行，进入phase_2函数
```

进入phase_2函数之后我们可以看到：

```c
0x400efe <phase_2+2>    sub    $0x28,%rsp  
0x400f02 <phase_2+6>    mov    %rsp,%rsi 
```

程序在栈上分配了40个字节的空间，并把栈顶地址存入了寄存器%rsi，我们把这个地址称为s1。

接着，调用了read_six_numbers函数：

```c
0x400f05 <phase_2+9>    callq  0x40145c <read_six_numbers> 
```

进入read_six_numbers函数之后，我们可以看到：

```c
0x40145c <read_six_numbers>     sub    $0x18,%rsp        # 在栈上分配了24字节空间
0x401460 <read_six_numbers+4>   mov    %rsi,%rdx         # %rdx值设为s1  
0x401463 <read_six_numbers+7>   lea    0x4(%rsi),%rcx    # %rcx值设为s1+4          
0x401467 <read_six_numbers+11>  lea    0x14(%rsi),%rax                                
0x40146b <read_six_numbers+15>  mov    %rax,0x8(%rsp)    # 0x8(%rsp)值设为s1+20             
0x401470 <read_six_numbers+20>  lea    0x10(%rsi),%rax                                 
0x401474 <read_six_numbers+24>  mov    %rax,(%rsp)       # (%rsp)值设为s1+16
0x401478 <read_six_numbers+28>  lea    0xc(%rsi),%r9     # %rcx值设为s1+12                 
0x40147c <read_six_numbers+32>  lea    0x8(%rsi),%r8	 # %rcx值设为s1+8 
```

紧接其后再次出现了含有即时数的操作：

```c
0x401480 <read_six_numbers+36>  mov    $0x4025c3,%esi
```

看看这个地址里有什么，输入：

```shell
x/s 0x4025c3
```

得到返回值：

```c
0x4025c3:       "%d %d %d %d %d %d"
```

看到这里感觉和字符串格式化有关，再往下看：

```c
0x40148a <read_six_numbers+46>  callq  0x400bf0 <__isoc99_sscanf@plt>                     
0x40148f <read_six_numbers+51>  cmp    $0x5,%eax                                           
0x401492 <read_six_numbers+54>  jg     0x401499 <read_six_numbers+61>                     
0x401494 <read_six_numbers+56>  callq  0x40143a <explode_bomb>
```

看起来可能是调用了sscanf函数(从一个字符串中读进相应格式数据)，并且返回值小于等于5会触发炸弹，结合前面存储的6个地址可推测出，应该是从我们输入的字符串中读入了6个整数，即我们输入的字符串要包含大于等于6个被空格隔开的整数。

返回到phase_2函数，继续看：

```c
0x400f0a <phase_2+14>   cmpl   $0x1,(%rsp)                                                 
0x400f0e <phase_2+18>   je     0x400f30 <phase_2+52>                                   
0x400f10 <phase_2+20>   callq  0x40143a <explode_bomb>                                 
0x400f15 <phase_2+25>   jmp    0x400f30 <phase_2+52>                                   
0x400f17 <phase_2+27>   mov    -0x4(%rbx),%eax      
0x400f1a <phase_2+30>   add    %eax,%eax 
0x400f1c <phase_2+32>   cmp    %eax,(%rbx)                                             
0x400f1e <phase_2+34>   je     0x400f25 <phase_2+41>                                   
0x400f20 <phase_2+36>   callq  0x40143a <explode_bomb>                                 
0x400f25 <phase_2+41>   add    $0x4,%rbx                                               
0x400f29 <phase_2+45>   cmp    %rbp,%rbx                                               
0x400f2c <phase_2+48>   jne    0x400f17 <phase_2+27>                                   
0x400f2e <phase_2+50>   jmp    0x400f3c <phase_2+64>                                   
0x400f30 <phase_2+52>   lea    0x4(%rsp),%rbx                                         
0x400f35 <phase_2+57>   lea    0x18(%rsp),%rbp                                         
0x400f3a <phase_2+62>   jmp    0x400f17 <phase_2+27>
```

仔细看这段代码，我们可以看出这是一个循环，其功能是将s1,s1+4,s1+8,s1+12,s1+16,s1+20这6个地址内的值与1,2,4,8,16,32比对，于是便可以得出结论，关卡2的密码字符串开头应是“1 2 4 8 16 32 ”，后续如何并无影响，至此，phase 2通过。

