---
title: 一篇文章入门python去写shell
tags: python
abbrlink: 40801
date: 2024-11-10 02:44:14
---

# 1 基础

有的编程语言要求必须提前将所有源代码一次性转换成二进制指令，也就是生成一个可执行程序（比如 Windows 下的 .exe 文件），比如C语言、[C++](http://c.biancheng.net/cplus/)、Golang、[汇编语言](http://c.biancheng.net/asm/)等，它们都属于**编译型语言**，使用的转换工具称为**编译器**。

有的编程语言可以一边执行一边转换，需要哪些源代码就转换哪些源代码，不会生成可执行程序，比如 [Python](http://c.biancheng.net/python/)、[JavaScript](http://c.biancheng.net/js/)、[PHP](http://c.biancheng.net/php/)、Shell 等，这类编程语言称为**解释型语言**，使用的转换工具称为**解释器**

注意:python使用的是对其方式来区分同一级的逻辑控制。不使用分号(；)所以设置的时候，一定要设置好一个tab对4个空格，不要使用tab,不然换到其他编辑器中，容易报语法错误。tab和空格不要混用。

python 中都是通过import去导入一些系统包或者自己写的包。这个和java的操作很相似,也和c中的#include <某些.h>相似。

毕竟市面上绝大多数的python的底层逻辑都是用c去写的。

加减乘除取余就不赘述了。所有编程的语法都是大同小异的。

python在linux的环境下不需要安装，我们使用的图形化界面都是以python和一些桌面管理为基础的。

一般存在的路径在/usr/bin/python, 查看python 默认的版本,直接python -v

python文件开头解释器说明。我是用python3版本的。编码格式为utf-8

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-
```

函数入口我一般是这样定义的

```python
if __name__ == '__main__':
    main()
```

然后在定义出main函数的运行内容

```python
def main():
	执行语句
```

## 1.1 流程控制

选择控制

```python
def ifTest():
    i = random.randint(1,100)
    print("随机生成的数字为",i)
    if i%2 == 0:
        print("是个偶数")
    else:
        print("是个奇数")
```

选择控制的格式如下:

```python
if 条件:
	执行命令
elif 条件:
	执行命令
else:
	执行命令
```

循环控制 while循环和for循环的示例如下:

```python
def whileTest(i):
    count = 0
    while i > 0 :
        i -=1
        count+=1
        print("循环的第",count,"次")

def forTest(i):
    for count in range(i):
        print("循环的第",count,"次")
    for count in range(0,i,3):
        print("步进循环的第",count,"次")
```

python不提供switch语句。虽然可以自己实现，但是我觉得没啥必要的。

## 1.2 数据类型

python中有6个标准的数据类型:

- Number(数字型号)
- String(字符串型号)
- List(列表)
- Tuple(元组)
- Sets(集合)
- Dictionaries(字典)

### number

int(x) 将x转换为一个整数。

float(x) 将x转换到一个浮点数。

complex(x) 将x转换到一个复数，实数部分为 x，虚数部分为 0。

complex(x, y) 将 x 和 y 转换到一个复数，实数部分为 x，虚数部分为 y。x 和 y 是数字表达式。

### String

#### 索引

字符串直接拿引号包起来就可以了。

索引下标[-1]表示倒数第一个。

```python
    s = '0123456789'
    s1 = s[0]
    print('s[0] = ' + s1)   #s[0] = 0
    print('s[3] = '+ s[3])  #s[3] = 3
    print('倒数第三个数为：' + s[-3])   #倒数第三个数为：6
    print('最后一个数为：' + s[-1])     #最后一个数为：8
```

切片

```python
    s = '0123456789'
    s2 = s[0:3]
    print('s[0:3] = ' + s2)     
    #s[0:3] = 012

    print('整个字符串如下：' + s[:])    
    #整个字符串如下：0123456789

    print('整个字符串如下：' + s[0:])   
    #整个字符串如下：0123456789

    print('前两个字符：' + s[:2])      
    #前两个字符：01
```

跳首

```python
    s = "01234567489"
    print(s[0:6:2]) #行首0,行尾6，间隔2 取 打印出024
    print(s[::2])      #024649
    print(s[4:0:-1])   #倒着取:4321
    print(s[3::-1])    #3210
    print(s[-1::-1])   #98476543210
```

#### 常见的字符串操作

- 大小写操作

  ```python
      s = "adbCDefg"
      print("首字母大写",s.capitalize())
      print("全部大写  ",s.upper())
      print("全部小写  ",s.lower())
      print("大小写互换",s.swapcase())
  ```

- 删除空格操作

  ```python
      s = '  xtt  123'
      s_1 = s.strip()             #删除字符串前后的空格    
      print(s_1)                  #xtt 123
      s_2 = s.strip('%')          #删除字符串前后的空格 
      print(s_2)                  #  xtt  123
      s_3 = s.replace(" ","")     #替换掉所有的空格
      print(s_3)                  #使用join()方法将字符串中所有的空格删除
      s=' This is a demo code'
      print(''.join(s.split()))   #Thisisademo
      #其中，join() 方法用于将序列中的元素以指定的字符连接生成一个新的字符串。
  ```

- 计算字符出现的次数count

  ```python
      s = '  xtt  123 xtt'
      count = s.count("t")
      print(count)  #打印出现的次数为4次
  ```

- 分割。split不加任何参数则默认空格
```python
    s1 = 'I am the king of amazon!!!'
    s2 = 'I am the: king: of amazon:'
    s_1 = s1.split()
    print(s_1)              #['I', 'am', 'the', 'king', 'of', 'amazon!!!']
    s_2 = s2.split(':')
    print(s_2)              #['I am the', ' king', ' of amazon', '']
```

- 格式化输出

  ```python
      s12_1 = '我叫{},今年{}岁，爱好{},再说一下我叫{}'.format('小明',22,'学习','小明')
      print(s12_1)  
      s12_2 = '我叫{0},今年{1}岁，爱好{2},再说一下我叫{0}'.format('小明',22,'学习')
      print(s12_2)
      s12_3 = s1 = '我叫{name},今年{age}岁，爱好{hobby},再说一下我叫{name}'.format(name = '小明',age = 18,hobby = '学习')
      print(s12_3)
  ```

### List

list列表可以存放多个值，创建list列表，使用[ ]，多个值之间用逗号隔开，不限制数据类型

```python
    l1 =["joenero","amazon","joker","father","was"]
    print(l1)								#['joenero', 'amazon', 'joker', 'father', 'was']
    print(l1[0:4:2])						#['joenero', 'joker']
    print(l1[-3:-1])						#['joker', 'father']
```

list相关的方法如下:

| 方法               | 说明                                                       |
| ------------------ | ---------------------------------------------------------- |
| .append(元素)      | 向列表最后追加一个元素                                     |
| .extend(元素)      | 向列表最后追加多个元素                                     |
| .insert(下标,元素) | 向指定的下标位置插入元素                                   |
| .pop(下标)         | 移除下标指定的元素，如果没有指定的下标，则删除最后一个元素 |
| .remove(元素)      | 删除指定元素                                               |
| .clear()           | 清空列表                                                   |
| .index(元素)       | 获取指定元素在list列表中的第一次出现的下标                 |
| .count(元素)       | 统计元素在list列表中出现的次数                             |
| .reverse()         | 反转list列表                                               |
| .sort()            | 排序.默认是升序，降序添加参数:reverse=True                 |

这三比较复杂,会在之后单独拎出来讲解
- Tuple(元组)
- Sets(集合)
- Dictionaries(字典)

# 2 添加help参数

一般我们写一个需要外部输入的参数，我们都需要借用解析器argparse去获取和解析对应的内容

```python
import argparse

def help():
    parser = argparse.ArgumentParser()
    parser.add_argument('-f',help='PSS文件')
    parser.add_argument('-o',help='生成的图片')
    args = parser.parse_args()
    print(args)  				#打印存储的所有输入值
    print(args.f)				#打印存储的-f 之后的值
    print(args.o)				#打印存储的-o 之后的值
```

然后运行这个py文件后面加-h的参数就可以显示对应的help值

例如上面的内容显示如下:

```shell
python plot.py -h
usage: plot.py [-h] [-f F] [-o O]

optional arguments:
  -h, --help  show this help message and exit
  -f F        PSS文件
  -o O        生成的图片
```



# 3 文件操作

## 基础


- open

  ```
  file = open("fileName.txt",mode="r")
  ```
  | 模式 | 描述                                                         |
  | ---- | ------------------------------------------------------------ |
  | ‘r’  | 以「只读」模式打开文件，如果指定文件不存在，则会报错，默认情况下文件指针指向文件开头 |
  | ‘w’  | 以「只写」模式打开文件，如果文件不存在，则根据 filename 创建相应的文件，如果文件已存在，则会覆盖原文件 |
  | ‘a’  | 以「追加」模式打开文件，如果文件已存在，文件指针会指向文件尾部，将内容追加在原文件后面，如果文件不存在，则会新建文件且写入内容 |
  | ‘t’  | 以「文本文件」模式打开文件                                   |
  | ‘b’  | 以「二进制」模式打开文件，主要用于打开图片、音频等非文本文件 |
  | ‘+’  | 打开文件并允许更新（可读可写），也就是说，使用参数 w+、a+ 也是可以读入文件的，在使用的时候，需要注意区别 |
  
  ```python
      file = open("fileName.txt",mode="r")
      num = 1
      for line in file:
          print("第",num,"行内容:",line)
          num +=1
      file.close()
  ```
  
- read(): 直接读取整个文件。

  ```python
      file = open("fileName.txt",mode="r")
      fileCon = file.read()
      print(fileCon)
      
      with open(file=r"fileName.txt",mode="r",encoding="utf-8") as f:
          content = f.read()
          print(content)
  ```

- readline():读一行

- readlines():按行读取所有数据。结果为列表，一行为一个成员。

## 实例

直接举个实际例子如下:

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-
"""
@File    : main.py
@Author  : JoeNero
@Time    : 2022/12/30 16:32
"""
import argparse
import fileinput
import re
import matplotlib.pyplot as plt
from scipy.signal import find_peaks

fileName = ""
outTxt   = ""

class INFO:
    def __init__(self):
        self.username = ""
        self.password = ""

def help():
    global fileName
    global outTxt
    parser = argparse.ArgumentParser()
    parser.add_argument('-f',help='对应的TXT文件')
    parser.add_argument('-o',help='生成的文件')
    args = parser.parse_args()
    fileName = args.f
    outTxt = args.o

#读取解析txt
def readTxt(filename):
    count = 0
    aInfo = INFO()
    for line in fileinput.input(filename):
        if count == 0:
            temp = line.split('username:')[1]
            temp = temp.replace("\n","")
            # print("temp",temp)
            aInfo.username = temp
        else :
            temp = line.split('password:')[1]
            # print("temp",temp)
            temp = temp.replace("\n","")
            aInfo.password = temp
        count+=1
    print("username",aInfo.username,"password",aInfo.password)
        
#主函数入口
def main():
    help()
    if fileName == None or len(fileName):
        print("输入文件不能为空")
    else :
        readTxt(fileName)

if __name__ == '__main__':
    main()
```

我这边是有个外部文件。其格式如下，可以通过以上的内容来读取解析其中的内容。

```
username:xtt
password:123456
```

生成临时文件. 路径为/tmp/tmp06pz62p5 文件名字为随机

```python
import tempfile

def main():
    dir_name = tempfile.mkdtemp()
    print (dir_name)

if __name__ == '__main__':
    main()
```

# 4 shell

python有很多操作shell的方式。需要先import 的包

```python
import os
```

第一种方式是直接用os.system("command")，其中返回0,表示执行命令成功，明显的缺点是无法将返回的值保存下来

```python
devices = os.system("adb devices")
```

第二种方式是用os.popen("command")

os.popen() 返回的是 file read 的对象，对其进行读取 read() 的操作可以看到执行的输出。

```python
devices = os.popen("adb devices")
print("deviecs = ",devices.read())
```

打印输出的结果为

```shell
deviecs =  List of devices attached
L7Z5AABQAILZPBTO        device
```

配合下将其修改成如下的内容，就可以把对应的adb设备获取出来，当前只能获取到最上面的那个，可以自己根据逻辑来完善

```python
devices = os.popen("adb  devices|awk '{print $1}'|sed -n '2p'")
print("deviecs = ",devices.read())
```

注意:popen中主要涉及到文件上的操作，但是一些shell中的sleep和拍照等操作不需要返回值还是用system来操作。

我实际操作拍照的时候，发现存在的这个问题。popen不生效，只有system的才生效。

# 5 类

python中的类和c++的及其相似。

```python
#定义INFO类,用来存储
class INFO:
    def __init__(self):
        self.username = ""
        self.password = ""
        
#使用
aInfo = INFO()
aInfo.username = temp
aInfo.password = temp
```

# 6 import

python import 和包含头文件的用法相似。

在同一个目录下直接import 文件名即可

导入模块的方式有如下几种:

```python
#hello.py
def say ():
    print("Hello,World!")

#say.py
import hello
hello.say()
```
临时添加模块完整路径如下:
```
    import sys
    sys.path.append('D:\\python_module')
```

"from 模块名 import 成员"的形式直接导入指定成员

