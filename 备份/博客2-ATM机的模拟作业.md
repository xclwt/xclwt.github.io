---
title: 'ATM界面的模拟作业'
date: 2018-10-01 15:51:55
tags: [作业,python]

---
>关于“ATM机界面的模拟作业”一些优化改进思路，仅供参考

![作业要求][1]


  [1]: http://xclwt-blog-image.oss-cn-hangzhou.aliyuncs.com/18-10-1/71985279.jpg

  <!--more-->
# ***初始方案***
```python
print('1.存款')
print('2.贷款')
print('3.开卡')
print('4.挂失')
print('请输入您要办理的业务')
number=input('请输入数字:') #python3中默认input输出类型为str，python2中则有所区别。
a=number #将输出赋值给a，方便后续调用
print (a)
if int(a) in range (1,5): #用int()将字符串转化为整型
    print('welcome!',a)
else:
    print('sorry!',a) #使用if判断语句判别用户输入数字是否在选项范围内
```
### **输出结果**
```
1.存款
2.贷款
3.开卡
4.挂失
请输入您要办理的业务

请输入数字:4
4
welcome! 4
```
# ***第一次与第二次优化方案整合***
>*使用对供选择的选项的打印代码进行了缩减*
>*将输出内容优化到数字对应的具体选项内容*
>*考虑到ATM机上除数字键外还有“#”“*”“确认”，对可能出现的bug进行了排除*
```python
business={1:'存款',2:'贷款',3:'开卡',4:'挂失'}
for key in business:
    print (str(key)+'.'+business[key])
number=input('请输入您要办理的业务:')
a=number
print (a)
if a=='#'or a=='*' or a=='确认':
    print('请重新输入!')
else:
    if int(a) in range (1,5):
        print('欢迎'+business[int(a)]+'!')
    else:
        print('对不起，无此选项!')
```
### **输出结果**
```
1.存款
2.贷款
3.开卡
4.挂失

请输入您要办理的业务:4
4
欢迎挂失!
```
# ***第三次与第四次优化方案整合***
>*考虑到实际操作中“确认”=“enter”以及用户不输入内容直接确认的情况，取消了代码中“确认”的情况*
>*使用while break语句优化了代码，使程序识别到用户操作不规范后自动重运行*
```python
business={1:'存款',2:'贷款',3:'开卡',4:'挂失'}
for key in business:
    print (str(key)+'.'+business[key])
while True:      #在用户输入不符合要求后重新执行程序
    number=input('请输入您要办理的业务:')
    a=number
    print (a)
    if a=='#'or a=='*' or a=='':      #考虑到atm机上还有“#”和“*”以及用户不输入内容直接确认的情况，确认键等同于“enter”操作
        print('请重新输入!')
    elif int(a) in range (1,5):
        print('欢迎'+business[int(a)]+'!')
        break        #用户选择正确选项时结束循环
    else:
        print('对不起，无此选项!')     #用户输入不在范围内数字时的反馈
```
### **输出结果**
```
1.存款
2.贷款
3.开卡
4.挂失

请输入您要办理的业务:#
#
请重新输入!

请输入您要办理的业务:
```
------
*作者：爱好小姐姐的咸鱼白*

*欢迎小姐姐们扫码加我微信*
![此处输入图片的描述][2]

  [2]: http://xclwt-blog-image.oss-cn-hangzhou.aliyuncs.com/18-10-1/80219838.jpg