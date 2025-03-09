---
title: nRF52832开发入门【二】模块化
tags: nodic
categories: nodic
abbrlink: 37752
date: 2025-03-09 17:22:17
---
# 1. 介绍
我们实际开发过程中往往会很复杂，为了更好的管理代码，我们需要模块化。模块化的好处有很多，比如：

1. 降低耦合度：模块化可以降低模块间的耦合度，使得代码更容易维护和修改。
2. 复用性：模块化可以提高代码的复用性，可以节省开发时间。
3. 降低成本：模块化可以降低开发成本，可以节省开发成本。

# 2.准备工作
VCode安装插件:
1.nRF Connect for VS Code
2.CMake
先创建一个空的工程文件
![](https://s3.bmp.ovh/imgs/2025/03/09/b636b2557bf0f419.png)

创建完,默认会创建一些最基础的配置文件

CMakeLists.txt和prj.conf

然后给这个app添加build配置

![](https://s3.bmp.ovh/imgs/2025/03/09/c14f1a613b73d198.png)

除了板子是你对应的手上的板子，其他一路默认即可

![](https://s3.bmp.ovh/imgs/2025/03/09/8cf7a733fee00a46.png)

然后开始build即可.

![](https://s3.bmp.ovh/imgs/2025/03/09/0ceb7b5e3a3cc9fb.png)

插上对应的板子 烧录build flash即可

![](https://s3.bmp.ovh/imgs/2025/03/09/e4904e2310b08942.png)

提供的默认空工程没啥东西，我这边按照我的自己的习惯对齐进行分模块，有的人习惯是把c文件放一块,h文件放一块,也可以每个模块都单独一个文件夹

我是习惯后者.前者是可以省去CMakeList.txt文件添加的麻烦，但是我觉得这就不算真正意义上的模块化了.

将src文件中的main.c挪出来，删掉src文件，并且添加自己想要添加的模块内容如下示例:

![](https://s3.bmp.ovh/imgs/2025/03/09/230c1b8e5b9dbd41.png)

另外再CMakeLists.txt中添加相关的编译说明:

```
aux_source_directory (led/ led_path)
aux_source_directory (button/ button_path)
aux_source_directory (bluetooth/ bluetooth_path)

target_sources(app PRIVATE main.c
                    ${led_path}
                    ${button_path}
                    ${bluetooth_path})
```

这样就能把每个模块单独分开,互相解藕

# 3 输出

输出一般单片或者嵌入式都是以led作为参考的示例.

非常的简单操作就是对led的节点 做dts检查后初始化,然后就可以输出高低电平了

```c
gpio_is_ready_dt(&led1);
gpio_pin_configure_dt(&led0, GPIO_OUTPUT);
gpio_pin_set_dt(&led0, 1);//高电平
gpio_pin_set_dt(&led0, 0);//低电平
```

更详细gpio定制化dts可以参考这个工程**custom_dts_binding**

# 4 输入

输出拿按钮button举例:

输入和输出相似 也是先做ready_dt check 然后配置成输入

```c
gpio_is_ready_dt(&button0);
gpio_pin_configure_dt(&button0, GPIO_INPUT);
//设置中断配置
gpio_pin_interrupt_configure_dt(&button,GPIO_INT_EDGE_TO_ACTIVE);
```



