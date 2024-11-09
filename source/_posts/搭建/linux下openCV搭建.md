---
title: linux下openCV搭建
tags: linux
abbrlink: 44710
date: 2024-11-10 03:09:18
categories: linux
---

# 1 依赖项
先下载好相关的依赖项目.确保编译成功。
```
sudo apt-get install build-essential
sudo apt-get install cmake git libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev
sudo apt-get install python-dev python-numpy libtbb2 libtbb-dev libjpeg-dev libpng-dev libtiff-dev libdc1394-22-dev
sudo apt-get install libopencv-dev
```
# 2 源码编译
clone 源码,我在gitee上放了一份。但是呢，实际上cmake的时候,部分内容还是会链接到github上。如果某些同学连接不到github的网站,那也没辙。
```
git clone https://gitee.com/joenero/opencv.git
```
进入源码目录,创建一个单独文件build
```shell
cd opencv
mkdir build
cd build
```
cmake 一下到/usr/local/目录下
```shell
cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr/local ..
```
执行编译
```shell
sudo make -j8
```
将make生成的文件下载到系统目录下
```
sudo make install
```
# 3 配置
用vim打开这个文件。因为这些只读文件。有些文本编辑器可能无法强制修改。但是vim是万能的。强制保存命令为 : w ! sudo tee %
```
sudo vim /etc/ld.so.conf
```
在打开的文件添加makefile安装路劲
```
/usr/local/lib
```
再运行
```
sudo ldconfig
```
# 4示例代码
## 4.1 cmake
cmakelist.txt添加如下内容
```
include_directories(/usr/local/include/opencv4/opencv2)
set (OpenCV_LIBS /usr/local/lib)
find_package(OpenCV REQUIRED)
target_link_libraries(helloCV ${OpenCV_LIBS}) #helloCV 工程名字
```
完整的cmakelist.txt如下:
```cmake
cmake_minimum_required(VERSION 3.17)
project(myTool)

set(CMAKE_CXX_STANDARD 14)

include_directories(/usr/local/include/opencv4/opencv2)
set (OpenCV_LIBS /usr/local/lib)
find_package(OpenCV REQUIRED)
add_executable(myTool main.cpp)

target_link_libraries(myTool ${OpenCV_LIBS}) #helloCV 工程名字
```
demon代码如下
```cpp
#include <opencv2/opencv.hpp>
#include <iostream>
#include <string>
using namespace cv;
using namespace std;
int main(int argc,char **argv)
{
    Mat img = imread("../test.jpeg");
 //   cout<<img;
    if(img.empty())
    {
        cout<<"error";
        return -1;
    }
    cout<<"My picture: "<< img.size() <<endl;
    imshow("image",img);
    waitKey();
    return 0;
}
```
如果出现报错： Gtk-Message: 21:57:35.293: Failed to load module "canberra-gtk-module" 则安装
```shell
sudo apt-get install libcanberra-gtk-module
```
## 4.2 Qt使用
在.pro文件中添加如下内容,根据个人情况
```pro
INCLUDEPATH += /usr/local/include \
                /usr/local/include/opencv4 \

LIBS += /usr/local/lib/libopencv*
```
出现这个错误，只需要在对应的文件中添加头文件
#include <opencv2/highgui/highgui_c.h>
```
OpenNCC_View/widget.cpp:330: error: ~~‘cvGetWindowHandle’~~ was not declared in this scope
                 if (!cvGetWindowHandle("OpenNCC"))
                      ^~~~~~~~~~~~~~~~~
```
