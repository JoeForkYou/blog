---
title: anaconda3管理python包环境说明
tags: python
abbrlink: 10736
date: 2024-11-10 02:38:18
---

@[toc]
# 1 概述

在实际项目开发的时候,我们往往需要不同的python包版本和环境。

pycharm对此就有很好的器包环境。

今天要介绍的是anaconda3 这个环境包管理。

Conda as a package manager helps you find and install packages. If you need a package that requires a different version of Python, you do not need to switch to a different environment manager, because conda is also an environment manager. With just a few commands, you can set up a totally separate environment to run that different version of Python, while continuing to run your usual version of Python in your normal environment.

----Conda官网

anaconda相当于一个包的管理者，去管理这些不同的环境，你可以本地建立多个虚拟环境，并且互相不影响。

# 2 安装配置

下载官网http://continuum.io/downloads

下载下来一路回车,配置好对应的环境变量

```shell
sudo chmod 777 Anaconda3-2021.11-Linux-x86_64.sh
./Anaconda3-2021.11-Linux-x86_64.sh
```

添加conda环境变量,根据本地下载的实际地方路径

```shell
vim ~/.bashrc
export PATH=~/anaconda3/bin:$PATH
```

由于我本地需要一个测试ITS的环境,并且在R版本和S版本的架构中.python的版本要求也不同,所以我同时也需要本地有python2和python3

新建一个环境，设定的python版本为python2.7，然后会跳出给你安装的相关配置，一路回车。

```
conda create -n py2 python=2.7
```

激活环境命令

```
conda activate py2
```

退出当前环境命令

```
conda deactivate
```

需要在当前环境下安装对应的pip包之前需要先激活对应的环境，然后pip安装即可

以下是我ITS S版本的python环境。

```shell
conda create -n py3 python=3.7.9 #这表示创建python版本为3.7.9 ,名字为py3的虚拟环境。不加python=版本 默认是2.7版本
conda activate py3 #激活并进入py3。
conda install opencv=3.4.2 //安装3.4.2版本的opencv 遇见选择Y/N 选择Y 下面都一样
conda install numpy=1.19.2 //安装1.19.2版本的numpy
conda install matplotlib=3.3.2 //安装3.3.2版本的matplotlib
conda install scipy=1.5.2 //安装1.5.2版本的scipy
conda install pyserial=3.5 //安装3.5版本的pyserial
conda install pillow=8.1.0 //安装8.1.0版本的pillow
conda install pyyaml=5.3.1 //安装5.3.1版本的pyyaml
pip install mobly //安装mobly
```

# 3 使用

这边搭建一个AI环境为例子

```shell
conda create -n pyAI python=3.7.9
conda activate pyAI #激活并进入环境
pip list 			#查看当前pip 的包
pip install pandas -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install scikit-learn -i https://pypi.tuna.tsinghua.edu.cn/simple 
pip install scipy -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install jupyter -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install nltk -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install jiaba -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install tensorflow -i https://pypi.tuna.tsinghua.edu.cn/simple
pip install tensorflow_addons  -i https://pypi.tuna.tsinghua.edu.cn/simple
```

AI安装好，运行如下命令打开对应的界面

```
jupyter notebook
```

# 4 使用命令

```
conda info --envs 		 #查看存在的环境
conda activate 环境 		#激活对应的环境
```

