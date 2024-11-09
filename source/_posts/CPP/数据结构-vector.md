---
title: 数据结构 【vector】
tags: cpp
abbrlink: 54319
date: 2024-11-10 01:14:59
categories: cpp
---

# 1 STL 简介

> STL是Standard Template Library的简称，中文名标准模板库
STL可分为
容器(containers)、
迭代器(iterators)、
空间配置器(allocator)、
配接器(adapters)、
算法(algorithms)、
仿函数(functors)六个部分。


选自百度词条[STL百度词条](https://baike.baidu.com/item/STL/70103?fr=aladdin)
C++标准中，STL组件被组织命名为以下13个头文件

>< algorithm>
< deque>
< functional>
< iterator>
< vector>
< list>
< map>
< memory.h> 
< numeric>
< queue> 
< set> 
< stack>
< utility>
# 2 容器 vector
向量(vector) 连续存储的元素< vector>;
vector是一个能够存放任意类型的动态数组，能够增加和压缩数据。
为了更好理解，不用基础类型，自行定义一个MyInt结构体

```cpp
struct MyInt
{
	string name;	//用作标识符
	int Int;        //实际存储类型
};
```
写一个打印函数如下
```cpp
/*
功能: 	- vector类打印
		@param
		@param
		@param
描述	:
示例	:
*/
void printVector(vector<MyInt>& v)
{
	for (vector<MyInt>::iterator it = v.begin(); it != v.end(); it++)
	{
		cout << "标识符为:" << it->name << " ";
		cout << "数据为:"<< it->Int << endl;
	}
	cout << endl;
}
```

## 2.1 构造
vector类构造demo
```cpp
void test01()
{
	vector<MyInt> v1; //无参构造
	MyInt* myInt = new MyInt[10];
	for (int i = 0; i < 10; i++)
	{
		myInt[i].name = '0' + i;
		myInt[i].Int = i;
		v1.push_back(myInt[i]);
	}
	printVector(v1);
	vector<MyInt> v2(v1.begin(), v1.end());
	printVector(v2);
	vector<MyInt>::iterator it = v1.begin();
	vector<MyInt> v3(it+1, v1.end());
	printVector(v3);
	delete[] myInt;
}
```
运行结果如下，因为用了迭代器来访问v1.begin() + 1的位置 ，(非基础类型不能用v1[1]访问),所以打印v3容器输出的结果是1到9
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cd43e6bd2b8d53298c83d9d6dc3ff2b2.jpeg)
## 2.2 成员函数
基础访问操作:（注意v3.at[int index]适合访问基础类型,int,char等，自定义的类型还是用迭代器访问）
vector不支持头插(push_front)
```cpp
v3.push_back(elem)  //在尾部插入一个elem数据。
v3.pop_back()       //删除末尾的数据。
v3.at(int index)	//传回索引为index的数据,如果index越界
					//抛出out_of_range异常。
					//非基础类型需要通过迭代器访问
```
### 2.2.1 assgin
v3.assign(beg,end)将[beg,end)一个左闭右开区间的数据赋值给v3。
```cpp
	v3.assign(v1.begin(), v1.end());
	printVector(v3);
```
v3.assign (n,elem)将n个elem的拷贝赋值给v3。
```cpp
	v3.assign(2, myInt[2]);
	printVector(v3);
```
输出结果如下:
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8f4409bb6edf35adcb6ebecb83746ac9.jpeg)
### 2.2.2 数据大小操作
#### 2.2.1.empty
v1.empty() //  判空操作 
先看empty()定义
```cpp
    _NODISCARD bool empty() const noexcept {
        auto& _My_data = _Mypair._Myval2;
        return _My_data._Myfirst == _My_data._Mylast;
    }
```
测试容器空的状况:
```cpp
	vector<MyInt> v1; //无参构造
	cout << v1.empty() << endl;
```
空返回1
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d1d724b0d342e5c785976127f9cb4a2f.jpeg)
非空返回 0
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f681f1fda108aa3696c04aa80a2a0b76.jpeg)
了解empty后修改打印函数(暂时不抛出异常)
```cpp
void printVector(vector<MyInt>& v)
{
	if (v.empty())
	{
		cout << "打印Vector为空";
		return;
	}
	else
	{
		for (vector<MyInt>::iterator it = v.begin(); it != v.end(); it++)
		{
			cout << "标识符为:" << it->name << " ";
			cout << "数据为:" << it->Int << endl;
		}
	}
	cout << endl;
}
```
#### 2.2.2  容量
参考链接[capacity用法](https://blog.csdn.net/JIEJINQUANIL/article/details/51166154)
|函数|  功能|
|--|--|
|capacity() |  容器能存储 数据的个数(真实大小)|
|size() |目前存在的元素个数|
|  max_size  |    最大容量    |
|resize()     | 重新指定大小 ，若指定的更小，超出部分元素被删除|
|reserve |	 预留空间|


```cpp
	cout << "v1真实的大小 = " << v1.capacity() << endl;		//真实的大小
	cout << "v1的大小 = " << v1.size() << endl;
	cout << "v1最大容量 = " << v1.max_size() << endl;
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/67b826a0fe159edebbc23f08616daa89.jpeg)
resize()使用，
重新指定大小10的v1容器为12，多指定的空间重新插入内容myInt[2]；

```cpp
	v1.resize(12, myInt[2]);
	printVector(v1);
```
打印输出结果:
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5fa6feb23652dc4f316bd766deac09f6.jpeg)
重新指定v1打下，删除超过索引的内容
```cpp
	v1.resize(2);
	printVector(v1);
```
输出结果
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8b63a542688e35cfd61689810353b8e5.jpeg)
### 2.2.3 删除与插入操作
|操作|说明  |
|--|--|
|  pop_back()	|  尾删|
|	insert（）|	插入操作|
|  	erase()  | 擦除    |
|	|	|
测试代码如下:

```cpp
	vector<MyInt> v1; //无参构造
	MyInt* myInt = new MyInt[3];
	for (int i = 0; i < 3; i++)
	{
		myInt[i].name = '0' + i;
		myInt[i].Int = i;
		v1.push_back(myInt[i]);
	}
	printVector(v1);
	//尾删
	v1.pop_back();
	printVector(v1);
	//插入
	vector<MyInt>::iterator it = v1.begin();

	v1.insert(it+1, myInt[1]);
	cout << "" << endl;
	printVector(v1);
	v1.insert(v1.begin(), 2, myInt[2]);
	printVector(v1);
	////删除
	v1.erase(v1.begin());
	cout << "擦除头:" << endl;
	printVector(v1);
	////清空
	//v1.erase(v1.begin(), v1.end());
	//printVector(v1);
	v1.clear();
	printVector(v1);
	delete[] myInt;
```
打印输出结果

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/70b367434aee7da1b47c36dced4fab0f.jpeg)
### 2.2.4 swap
速览其定义:
```cpp
    void swap(vector& _Right) noexcept /* strengthened */ {
        if (this != _STD addressof(_Right)) {
            _Pocs(_Getal(), _Right._Getal());
            _Mypair._Myval2._Swap_val(_Right._Mypair._Myval2);
        }
```

swap交换，直接看代码吧:两容器大小需要一致，否在会进入异常
```cpp
	vector<MyInt> v1; 
	vector<MyInt> v2;
	MyInt* myInt = new MyInt[3];
	for (int i = 0; i < 3; i++)
	{
		myInt[i].name = '0' + i;
		myInt[i].Int = i;
		v1.push_back(myInt[i]);
	}
	for (int i = 0; i < 3; i++)
	{
		myInt[i].name = '0' + i;
		myInt[i].Int = 3 - i;
		v2.push_back(myInt[i]);
	}
	cout << " v1 =" << endl;
	printVector(v1);
	cout << " v2 =" << endl;
	printVector(v2);
	//互换容器
	cout << "互换后" << endl;
	v1.swap(v2);
	cout << " v1 =" << endl;
	printVector(v1);
	cout << " v2 =" << endl;
	printVector(v2);
```
输出打印结果如下:
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/76d01921df3a4cba71f040a84ae42ae5.jpeg)
在实际应用中swap常常用于与自身交换来达到收缩内存的效果，这里不加累述
[swap收缩内存参考链接](https://blog.csdn.net/liyazhen2011/article/details/103179974)
#  3 vector总结
|成员函数	|  	说明	|
|--|--|
| 	push_back | 	尾插	 |
|	assign|	[assgin用法](https://blog.csdn.net/qq844352155/article/details/38583529)|
|	resize	|	重构大小	|
|	capacity|	 容量(真实大小)|
|	size|	当前大小 |
| max_size| 最大容量 |
| pop_back	|	尾删	 |
|  insert	|	插入	|
 |erase 	|	删除	|
 |		clear	|	清空|
 |	swap		|		交换|
 |reserve  |		预留空间|
 



