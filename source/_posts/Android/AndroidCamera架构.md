---
title: AndroidCamera架构
tags: camera
categories: camera
abbrlink: 28751
date: 2024-11-10 18:53:19
---
​

分层：将各层的接口和实现分开
Camera架构
# APP
所在位置
    架构最顶层
作用
    负责跟用户交互
流程
    接受到用户上的UI操作
    将UI操作通过request操作下发
    接收到底层返回的信息并反馈给用户
# CameraFramework/Service
CameraFramework
    作用
        以jar包的形式运行在APP进程中
    流程
        暴露接口供app调用
        接收app的请求
        通过调用Camera AIDL跨进程接口将请求发送到camera service进行处理
        将相关的结果返回至app
Camera Service
    作用
        封装Camera AIDL跨进程接口
        独立进程 Android 系统启动初期运行起来
# Provider
内部加载Camera Hal Module
    遵循谷歌制定的标准Camera Hal3接口
    由OEM/ODM实现Module
# Driver
CameraSensor 驱动/AF/otp等相关驱动。用于实现其基本逻辑
# Hardware
camera最底层V4L2
物理实现部分/dts/相关设备树供电

![​​​](https://i-blog.csdnimg.cn/blog_migrate/495b8430ba7fe0618c00d8472771e86e.png)