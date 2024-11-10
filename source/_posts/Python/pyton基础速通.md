---
title: pyton基础速通
tags: python
categories: python
abbrlink: 14466
date: 2024-11-10 14:43:26
---
# 1 基础

## 1.1 输出

```python
age = 10
print("helloworld")
print("我的名字是%s,我的国籍是%s"%("JoeNero","中国"))
print("a=%d"%age)
```

## 1.2 输入

```python
password = input("请输入密码")
print("您刚才输入的密码是",password)
```

## 1.3 注释

```python
'''
多行注释
'''

#单行注释
```
## 1.4 流程控制
```python
if 条件:
  执行语句
elif 条件:
  执行语句
else:
 执行语句
```

```python
a = 1
while a < 10
  a++
  print(a)
```

```python
for i in range(5):
    print(i)
    
  a = ["aa","bb","cc","dd"]
 for  i in range(len(a)):
        print(i,a[i])
```
# 2 字符串

```python
my_str = "I said \"I like you \"" #   \"转义输出"
print(my_str)
```
输出打印字符串的片段
```python
str = "chengdu"
print(str[0:5])
print(str[:5])
print(str[1:7:2])
print(str[6:])
print(str+"123") #字符串链接
```

输出

```shell
cheng
cheng
hnd
u
```
```python
print("hello\n")  #\n转义字符
print(r"hello\n") #加r使转义字符失效
```

# 3 元组

元组是不允许修改的

```python
tup1 = () #创建空的元组
print(type(tup1))

tup2 =(50,60,70)
print(type(tup2))
```

# 4 函数

```python
def add(a,b):
    print(a+b)

add(1,3)
```

# 5 文件操作

## 5.1 打开

```shell
f = open("test.txt","w") #打开文件.w模式，写模式

f.write("hello world JoeNero")  #将字符串写入文件中哦

f = open("test.txt","r") #打开文件.r模式，
content = f.read(5) #读取五个字符
print(content)
f.close() #关闭文件
```

| 模式 |                                                              |
| ---- | ------------------------------------------------------------ |
| r    | 以只读方式打开文件.文件的指针会放在文件的开头。这是默认模式  |
| w    | 打开一个文件只用于写入.如果该文件已存在则将其覆盖.如果该文件不存在.则创建新文件 |
| a    | 打开一个文件用于追加.如果该文件已经存在.文件指针会放在文件的结尾<br>如果该文件不存在,创建新文件进行写入 |
| rb   | 以二进制格式打开一个文件用于写入.文件指针会放在文件的开头.这是默认模式 |
| wb   | 以二进制的格式打开一个文件只用于写入.如果该文件存在，则会将其覆盖.如果文件不存在，创建新文件 |
| ab   | 以二进制格式打开一个文件用于追加.<br>如果该文件存在，文件指针将会放在文件的结尾.也就是说新的内容会被写入到以有内容之后.如果文件不存在,创建新文件进行写入 |
| r+   | 打开一个文件用于读写.文件指针将会放在文件的开头              |
| w+   | 打开一个文件用于读写.如果该文件已经存在，则覆盖.如果该文件不存在.创建新文件 |
| a+   | 打开一个文件用于读写.如果该文件已经存在.文件指针将会放在文件的结尾.<br>文件打开时会追加模式.如果该文件不存在,创建新文件用于读写 |
| rb+  | 以二进制格式打开一个文件用于读写.文件指针将会放在文件的开头  |
## 5.2 读取

```python
f = open("test.txt","r")
content = f.readlines() #一次性读取全部文件

#print(content)

i = 1
for temp in content:
    print("%d:%s"%(i,temp))
    i +=1
```

```python
import os

os.rename("test.txt","test1.txt") #重命名
```

# 6 错误和异常

```python
try:

    
except IOErrpr:
    pass
```

```python
try:

    
except IOErrpr:
    pass
finally:
    f.close()
    print("文件关闭")
```


















