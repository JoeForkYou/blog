---
title: linux常用基础命令
tags: linux
categories: linux
abbrlink: 20298
date: 2024-11-10 14:39:06
---

这个文件为系统apt 管理软件包的文件.图形化界面操作多了,就差不多忘记终端的以下基础.

知道这个文件就好,可以更改,也可以图形界面更改.

我们所使用的ubuntu系统是有自带的系统终端的. 我们平时操作都是在其中的桌面终端上操作的.一般是GNOME和KDA这种.

    /etc/apt/sources.list

# 1 内存

    df -h #查看系统各个磁盘的占用情况

du 是disk usage 的简称 用来显示目录或文件的大小,查找文件和目录的磁盘使用情况的命令.

    du -sh 查看当前文件所占用的空间
    du -sh * 查看当前文件夹下所有文件夹所占用的空间

# 2 adb

adb (Android Debug Bridge)是一种允许模拟器或已经连接的Android设备进行通信的命令行共军,它可以为各种设备操作提供便利.如安装和调试应用.

查询已经连接的设备

```shell
adb devices
```

adb 调佣图片命令。 前提要在此路径下存在对应的图. 不然会调用起损坏的图片

    adb shell am start -a android.intent.action.VIEW -t image/png -d file://mnt/sdcard/Download/scene1_1.png

拍照,拍照时间的keyevent 为27,所以 输入以下的命令就可以实现拍照.

    adb shell input keyevent 27

adb 查看当前包名和activity.  这个可以配合调apk来使用.我们一般要先确定调用的是哪个apk和activity

    adb shell dumpsys window |grep mCurrentFocus

输出打出以下的信息

      mCurrentFocus=Window{90dd2d3 u0 com.sec.android.app.camera/com.sec.android.app.camera.Camera}

那么adb 启动apk的方式

    adb shell am start -n com.sec.android.app.camera/com.sec.android.app.camera.Camera

adb 回到home

    adb shell input keyevent 3

查看设备安装的第三方应用

    adb shell pm list packages -3

查看系统安装的应用

    adb shell pm list packages -s

adb install

    -l 将应用安装到保护目录/mnt/asec
    -r 允许覆盖安装
    -t 允许安装AndroidManifest.xml里application 指定android:testOnly="true"的应用
    -s 将应用安装到sdcard
    -d 允许降级覆盖安装

adb install 实际分三步完成:
1.push apk 文件到/data/local/tmp
2.调用pm install 安装
3.删除/data/local/tmp 下的对应apk文件

与install 相反的是uninstall

adb uninstall  -k package-name

清楚应用缓存

adb shell pm clear \<package-name>

查看应用安装路径

adb shell pm path \<package-name>

```shell
adb shell input keyevent 26 #控制电源键，一般来控制息屏和亮屏
adb shell input keyevent 82 #菜单键,用户版本才用到
adb shell input keyevent 4 #返回键
adb shell input keyevent 24 #增加音量
adb shell input keyevent 25 #降低音量
adb shell input keyevent 164 #静音
adb shell input keyevent 224 #亮屏
adb shell input keyevent 223 #熄屏
```

查看屏幕分辨率

    adb shell wm size

查看屏幕设备密度

    adb shell wm density

# 3 CP

cp 是拷贝命令.

要是要拷贝文件只要加cp -r 即可

# 4 VIM

vim 是一个比较好用的文本编辑器

正常调用vim 就即可. vim 后面接对应的文件,并且vim 打开文件时会在本地创建一个bak文件,用于奔溃的时候的备份.

而且vim 可以更改到系统级别的文件.意味着 什么文件都能改,

vim 的模式有很多种

一般我们用到插入(可编辑)模式和命令模式

输入i 就进去插入模式,可以进行文本编辑

输入o 插入到当前光标下行并且进入到插入模式.

输入esc就退出当前的模式，回到命令模式

在命令模式下直接输入,就会查找对应的文本. 按n 即可查找下一个  ,以下是一些常用的命令

```shell
/文本内容
:wq  #保存并退出
:w!  #强制保存
:q   #退出
:q!  #强制退出
:w ! sudo tee # 保存只读文件
:u  #撤掉当前的修改
:行数 #跳转到对应的行数
dd #删除当前行的内容
```

# 5 快捷键

快捷键可以自己定义

当然系统默认好用快捷键如下:

    ctrl + shift +c #复制
    ctrl + shift +v #粘贴
    ctrl + shift + t #在当前终端栏边上打开终端  一般我不用ctrl +alt + t  那样打开的终端 很乱 
    ctrl + c #中断终端操作
    ctrl + d #退出当前窗口
    ctrl + q #关
    ctrl + r #查询调用历史输入的命令

# 6 shell

我们这边提到的是**命令行式shell**,不是gnome KDE那种桌面式终端.

shell类似于DOS下的COMMAND.COM和后来的cmd.exe。它接收用户命令，然后调用相应的应用程序.

创建后缀为.sh

Shell 脚本（shell script），是一种为 shell 编写的脚本程序。

业界所说的 shell 通常都是指 shell 脚本，但要知道，shell 和 shell script 是两个不同的概念。

由于习惯的原因，简洁起见，都是指 shell 脚本编程，不是指开发 shell 自身。

    #!/bin/bash
    echo "Hello World !"

\#!  是一个约定的标记,它告诉系统这个脚本需要用什么解释器来执行,即使用哪一种shell

echo 命令用于向窗口输出文本.

作为可执行程序运行

    chmod +x ./test.sh #使脚本具有执行权限
    ./test.sh #执行脚本

## 6.1 变量

    your_name="somethings"
    echo $your_name
    echo${your_name}

循环打印变量

```shell
#!/bin/bash
for skill in JOJO nONONONO AMAZON java;do
	echo "I am good at ${skill} Script"
done
```

## 6.2 字符

单引号

```shell
str='this is a string'
```

双引号

```shell
your_name="JoeNero"
str="hello ,I know you are \"$your_name\"! \n"
echo -e $str
```

拼接字符串

```shell
#使用双引号拼接
your_name='JoeNero'
greeting="hello ,"$your_name"!"
greeting_1="hello,${your_name}!"
echo $greeting $greeting_1
#使用单引号拼接
greeting_2='hello ,'$your_name'!'
greeting_3='hello,${your_name}!'
echo $greeting_2 $greeting_3
```

获取字符串长度

```shell
string="abcd"
echo${#string}
```

提取子字符串

```shell
string="runoob is a great site"
echo ${string:1:4}#输出unoo
#注意第一个字符的索引值为0
```

查找字字符串

查找字符i或o的位置(哪个字母先出现就计算哪个)

    string="runoob is a great site"
    echo `expr index "$string" io`

## 6.3 数组

用括号来表示数组,数组元素用空格符号来分割.

```shell
array_name=(1 2 3 4 5 6)
array_name1=(
1
2
3
4
5
6
)
array_name[0]=0
array_name[1]=1
array_name[2]=2
```

读取数组的格式

```shell
${数组名[下标]}
echo ${array_name[@]}

# 取得数组元素的个数
length=${#array_name[@]}
# 或者
length=${#array_name[*]}
# 取得数组单个元素的长度
lengthn=${#array_name[n]}
```

## 6.4 流控制

```shell
num1=100
num2=100
if test $[num1] -eq $[num2]
then
    echo '两个数相等！'
else
    echo '两个数不相等！'
fi
```

```shell
if condition1
then
    command1
elif condition2 
then 
    command2
else
    commandN
fi
```

**for 循环**

```shell
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done
```

实例

```shell
for loop in 1 2 3 4 5
do
    echo "The value is: $loop"
done
```

while语句

```shell
while condition
do
    command
done
```

```shell
#!/bin/bash
int=1
while(( $int<=5 ))
do
    echo $int
    let "int++"
done
```

无线循环

```shell
while:
do 
	command
done
```

    while true
    do 
    	command
    done

输入输出重定向文件在此不赘述

# 7 LS

ls 是list files的缩写

```shell
ll -t # 按照时间排序呈现当前目录下的文件内容
ll -a # 显示隐藏文件
ll -Sh # 按照文件大小排序显示当前目录下的文件
```

