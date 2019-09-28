---
title: '如何拆除CMU-15213的bomb lab？（2）'
date: 2019-09-17
tags: 
- 汇编
- GDB
---


###  phase 3：

执行以下操作：

```c
break 0x400e72		# 在phase_3函数调用处设置断点
......
（输入任意字符串）
stepi 				# 单步执行，进入phase_3函数
```

  <!--more-->

进入phase_3函数后，我们可以看到程序在栈上分配了24字节空间，我们把现在的栈顶称为s1：

```c
0x400f43 <phase_3>      sub    $0x18,%rsp                             0x400f47 <phase_3+4>    lea    0xc(%rsp),%rcx        # %rcx值设为s1+12 0x400f4c <phase_3+9>    lea    0x8(%rsp),%rdx		# %rdx值设为s1+8
```

接下来又出现了包含即时数的操作：

```c
0x400f51 <phase_3+14>   mov    $0x4025cf,%esi
0x400f56 <phase_3+19>   mov    $0x0,%eax                               
0x400f5b <phase_3+24>   callq  0x400bf0 <__isoc99_sscanf@plt>
0x400f60 <phase_3+29>   cmp    $0x1,%eax                           
0x400f63 <phase_3+32>   jg     0x400f6a <phase_3+39>                   
0x400f65 <phase_3+34>   callq  0x40143a <explode_bomb> 
```

我们来看看这个地址里有什么，输入：

```c
x/s 0x4025cf
```

得到返回值：

```c
0x4025cf:       "%d %d"
```

并且又调用了sscanf操作，并且返回值小于等于1会触发炸弹，所以此处应该是从我们输入的字符串中读入了2个整数。

重新进入phase3（此次的密钥输入两个任意整数），继续往下读：

```c
0x400f6a <phase_3+39>   cmpl   $0x7,0x8(%rsp)
0x400f6f <phase_3+44>   ja     0x400fad <phase_3+106> 
......
0x400fad <phase_3+106>  callq  0x40143a <explode_bomb>
```

此处出现了s1+8存储的值与7的比较，执行以下操作查看s1+8存储的值：

```c
x/x $rsp+8
```

我们会发现该值正是我们输入的第一个整数,若这个数大于7,则会跳转到explode_bomb函数的调用，注意此处的跳转使用的是ja命令，说明上一步执行了对无符号整数的比较，故我们输入的第一个整数范围应当是0-7。

再向下看：

```c
0x400f71 <phase_3+46>   mov    0x8(%rsp),%eax                         
0x400f75 <phase_3+50>   jmpq   *0x402470(,%rax,8)                     
0x400f7c <phase_3+57>   mov    $0xcf,%eax                             
0x400f81 <phase_3+62>   jmp    0x400fbe <phase_3+123>                 
0x400f83 <phase_3+64>   mov    $0x2c3,%eax                             
0x400f88 <phase_3+69>   jmp    0x400fbe <phase_3+123>                 
0x400f8a <phase_3+71>   mov    $0x100,%eax                             
0x400f8f <phase_3+76>   jmp    0x400fbe <phase_3+123>                 
0x400f91 <phase_3+78>   mov    $0x185,%eax                             
0x400f96 <phase_3+83>   jmp    0x400fbe <phase_3+123>                 
0x400f98 <phase_3+85>   mov    $0xce,%eax                             
0x400f9d <phase_3+90>   jmp    0x400fbe <phase_3+123>                 
0x400f9f <phase_3+92>   mov    $0x2aa,%eax                             
0x400fa4 <phase_3+97>   jmp    0x400fbe <phase_3+123>                 
0x400fa6 <phase_3+99>   mov    $0x147,%eax                             
0x400fab <phase_3+104>  jmp    0x400fbe <phase_3+123>                 
0x400fad <phase_3+106>  callq  0x40143a <explode_bomb>                 
0x400fb2 <phase_3+111>  mov    $0x0,%eax                               
0x400fb7 <phase_3+116>  jmp    0x400fbe <phase_3+123>                 
0x400fb9 <phase_3+118>  mov    $0x137,%eax                             
0x400fbe <phase_3+123>  cmp    0xc(%rsp),%eax                         
0x400fc2 <phase_3+127>  je     0x400fc9 <phase_3+134>                 
0x400fc4 <phase_3+129>  callq  0x40143a <explode_bomb>                 
0x400fc9 <phase_3+134>  add    $0x18,%rsp                             
0x400fcd <phase_3+138>  retq
```

我们输入的第一个整数被移入%rax寄存器，并且在下一步操作中直接跳转到了（0x402470+%rax×8）地址中存储的位置，因为我们输入的整数范围0-7,而十六进制整型数据占用4个字节，则%rax×8跨越15个内存地址，所以尝试执行以下指令查看从0x402470往后16个内存地址的内容（每个内存地址4字节）：

```c
x/16xw 0x402470
```

得到结果：

```c
0x402470:       0x00400f7c      0x00000000      0x00400fb9      
0x00000000
0x402480:       0x00400f83      0x00000000      0x00400f8a      
0x00000000
0x402490:       0x00400f91      0x00000000      0x00400f98      
0x00000000
0x4024a0:       0x00400f9f      0x00000000      0x00400fa6      
0x00000000
```

其中有8个非0的十六进制数，我们仔细比对可以发现正可以与后面尚未执行的指令中的8个的地址相对应。这8个指令，每个都是执行将一个十六进制整型数传入寄存器%eax的指令，紧接其后都是跳转向了这条指令：

```c
0x400fbe <phase_3+123>  cmp    0xc(%rsp),%eax
```

将s1+12处的数（就是我们输入的第二个整数）与寄存器%eax中的数比较，若相同便不回触发炸弹。

至此，我们可以理清楚了：当输入的第一个整数为0-7时，分别有一个对应的第二个整数可以通过关卡3.

仔细对比指令中的组合，我们最终得到了以下8组整数：

```
0 207
1 311
2 707
3 256
4 389
5 206
6 682
7 327
```

至此，phase 3通过。



### phase 4：

执行以下操作：

```c
break * 0x400e8e		# 在phase_4函数调用处设置断点
......
（输入任意字符串）
stepi					#单步执行进入phase_2函数
```

一如既往的，我们见到了含有立即数的指令并且很容易得出了对我们输入字符串的要求是两个整数。往下看：

```c
0x40102e <phase_4+34>   cmpl   $0xe,0x8(%rsp)           
0x401033 <phase_4+39>   jbe    0x40103a <phase_4+46>  
0x401035 <phase_4+41>   callq  0x40143a <explode_bomb>
```

程序将我们输入的第一个整数与14比较，如果大于14就会触发炸弹，并且由jbe指令我们可以得出输入的第一个整数应该是0-14中的任一无符号整数。

再往后看，调用了函数func4,且返回值不为0会触发炸弹，进入函数func4看看：

```c
0x400fce <func4>        sub    $0x8,%rsp                               
0x400fd2 <func4+4>      mov    %edx,%eax                               
0x400fd4 <func4+6>      sub    %esi,%eax                               
0x400fd6 <func4+8>      mov    %eax,%ecx                               
0x400fd8 <func4+10>     shr    $0x1f,%ecx                             
0x400fdb <func4+13>     add    %ecx,%eax                               
0x400fdd <func4+15>     sar    %eax                                   
0x400fdf <func4+17>     lea    (%rax,%rsi,1),%ecx                     
0x400fe2 <func4+20>     cmp    %edi,%ecx                               
0x400fe4 <func4+22>     jle    0x400ff2 <func4+36>                     
0x400fe6 <func4+24>     lea    -0x1(%rcx),%edx                         
0x400fe9 <func4+27>     callq  0x400fce <func4>                       
0x400fee <func4+32>     add    %eax,%eax                               
0x400ff0 <func4+34>     jmp    0x401007 <func4+57>                     
0x400ff2 <func4+36>     mov    $0x0,%eax                               
0x400ff7 <func4+41>     cmp    %edi,%ecx                               
0x400ff9 <func4+43>     jge    0x401007 <func4+57>                     
0x400ffb <func4+45>     lea    0x1(%rcx),%esi                         
0x400ffe <func4+48>     callq  0x400fce <func4>                       
0x401003 <func4+53>     lea    0x1(%rax,%rax,1),%eax                   
0x401007 <func4+57>     add    $0x8,%rsp                               
0x40100b <func4+61>     retq
```

在func4中，我们可以看到存在调用自身的操作，这应该是个递归函数，逻辑不是很清楚，尝试用逆向工程的方法将其还原为如下c程序：

```c
int func4(int num, int x, int y){				# num为我们输入的第一个整数
    int result = x - y;
    result /= 2;
    int temp = result + y;
    if (temp < num){
        return 2 * func(num, x, temp + 1) + 1;
    }
    else if (temp > num){
        return 2 * func4(num, temp - 1, y);
    }
    else return result;
}
```

根据这段c程序很容易推演出当我们输入的第一个整数为0,1,3,7时，func4返回值为0。

回到phase_4继续看

```c
0x401051 <phase_4+69>   cmpl   $0x0,0xc(%rsp)                         
0x401056 <phase_4+74>   je     0x40105d <phase_4+81>                   
0x401058 <phase_4+76>   callq  0x40143a <explode_bomb>  
```

这段程序将第二个整数与0比较，若不想等就会触发炸弹，所以，第二个整数应该为0。

至此，phase 4通过。