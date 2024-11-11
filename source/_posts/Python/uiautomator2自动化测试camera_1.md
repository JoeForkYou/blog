---
title: '''uiautomator2自动化测试camera【一】'''
tags: python
categories: python
abbrlink: 63775
date: 2024-11-11 22:46:49
---
# 1 概述
本文档是自己写andorid camera自动化测试的随笔
测试机器为
OPPO Find x7
测试Apk为oppo的系统相机.
# 2 准备工作
我电脑是有装conda环境的,所以我直接用conda创建虚拟环境专门用于相关的测试.
官网下的巨慢，直接去清华大学的镜像源下载速度快很多.
https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/
下载对应的系统版本即可
linux对应sh文件.
windows直接双击exe文件安装即可.
我不是很喜欢直接破坏本机电脑的python环境,所以我创建了一个新的虚拟环境. 这种包管理更为安全，pip炸了就炸了 打不了删了重新建一个
```shell
conda create -n py3 python=3.7  #创建python3.7的虚拟环境
conda activate  py3             #激活虚拟环境
#conda deactivate               #退出虚拟环境
conda info --envs               #查看虚拟环境
# conda environments:
#
py2                      C:\Users\Admin\.conda\envs\py2
py3                      C:\Users\Admin\.conda\envs\py3
base                     D:\conda
```
激活后会显示当前所在的环境名字，比如我这边是py3.
```shell
(py3) D:\Pr\demo>
```
安装需要的包
```
pip install uiautomator2
pip install pyyaml
```
电脑要提前安装好相关的adb.直接去谷歌官网下就行，linux下直接apt-get install adb就行.
如果adb 版本有问题，可以到https://developer.android.com/studio/releases/platform-tools下载最新版.然后添加到
我需要先获得测试apk的包名,本质上我都去调用一些api接口.
这个包名可以替换的.
清下log,然后开始抓log.开始抓log后打开你所用的camera app.
adb logcat -c
adb logcat -G 20M
adb logcat -b all >main.txt
打开main.txt log
过滤log 关键字connect call
看到我这边打印的一个
```shell
11-11 21:42:43.435  1458  8995 I CameraService: CameraService::connect call (PID 6159 "com.oplus.camera", camera ID 5) and Camera API version 2
```
com.oplus.camera 就是我用的测试apk的包名,对应的camera ID 5 就是我打开的相机的ID.
话说为什么是5,我记得后摄一般项目都是做成0.
一般remosaic的相机ID是会做别的映射，我打了好多不同模式的，没明白他的映射id是怎么做的.
等有机会我自己写个apk，给这个手机hal的信息慢慢剖出来看下人家产品是怎么做的.
11-11 21:53:24.108  1458 10185 I CameraService: CameraService::connect call (PID 6159 "com.oplus.camera", camera ID 5) and Camera API version 2
11-11 21:53:27.274  1458  9223 I CameraService: CameraService::connect call (PID 6159 "com.oplus.camera", camera ID 1) and Camera API version 2
11-11 21:53:46.899  1458  2400 I CameraService: CameraService::connect call (PID 6159 "com.oplus.camera", camera ID 2) and Camera API version 2
扯远了.
# 3 写个demo
新建一个python文件,名字为oppoCam.py
写个简单的demo
```python
# -*- coding: utf-8 -*-

import uiautomator2 as u2
import yaml                         # 引入yaml模块 预留我后续用这个做基本配置文件
import time                         # 引入time模块 预留我后续用这个做延时
if __name__ == '__main__':
    package = "com.mediatek.camera" # 设置需要运行的包名

    sn = 'YD9HVGXGZLA6ZHCQ'         # 设置手机序列号 adb devices -l 获取

    d = u2.connect(sn)              # 连接手机
    d.app_start(package)            # 启动app
    print(d.info)                   # 打印手机信息
```
第一次运行好像还会从github上下载ATX和uiautomator2的包,下载完后就可以运行了.
```shell
python oppoCam.py
```
第二次运行就很快了.
我这边打印出来了一些信息
```
(py3) D:\Pr\demo\py>python oppoCam.py
{'currentPackageName': 'com.android.launcher', 'displayHeight': 2256, 'displayRotation': 0, 'displaySizeDpX': 360, 'displaySizeDpY': 792, 'displayWidth': 1080, 'productName': 'PHZ110', 'screenOn': True, 'sdkInt': 34, 'naturalOrientation': True}
```
自此相关的准备工作都已经完成可以做后续的拍照/切换/录像等操作了.
剩余部分另外整理


