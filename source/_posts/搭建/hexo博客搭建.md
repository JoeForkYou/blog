---
title: hexo博客搭建
tags: hexo
categories: hexo
abbrlink: 31415
date: 2024-11-10 14:58:32
---
# 准备工作
hexo 博客的搭建要依赖nodejs的组件
直接去官网nodejs.org下载即可
在浏览器中输入https://nodejs.org/zh-cn
下载添加到环境变量中即可
win10 用户的环境变量和系统变量都添加然后重启就行.
因为我也不知道怎么让win10的环境变量生效,对于linux来说只要bash一下就可以了.

下载完验证
```shell
node -v
v22.11.0
npm -v
10.9.0
```
打印出来node的版本信息,然后更换镜像源
```shell
npm install -g cnpm --registry=http://registry.npm.taobao.org
```
用cnpm代替npm安装hexo架构
```
cnpm install -g hexo-cli  
```
查看hexo的版本
```shell
hexo -v
```
# 搭建
准备工作做的差不多了,该开始搭建了
创建一个你自己的文件，我这边是直接创建了blog文件
然后进入到这个文件中做初始化即可,这个要等等,不一定能拉全或者拉下来，可能还要改host文件.忘了以后再说
```shell
hexo init
```
基础操作
启动默认端口是本地的4000端口
```shell
hexo s
```
访问即可http://localhost:4000/
hexo init的时候回创建一个默认的markdown文件.
最常用的就是hexo g 生成静态文件
在hexo g之前先hexo clean一下,清除静态文件
然后hexo s查看改动后的效果
# 托管
github pages 托管
自检一个githu仓库，然后命名为username.github.io
hexo d 部署
要用hexo d部署的话,需要配置_config.yml文件,并且
要安装hexo-deployer-git
cnpm install --save hexo-deployer-git #在blog目录下安装git部署插件
这样子后我们再去配置_config.yml文件
在如下的位置添加自己部署的仓库和分支,然后hexo d部署即可
```yml
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: 'git'
  repo:  https://github.com/JoeForkYou/JoeForkYou.github.io.git
  branch: master
```
hexo d部署后,需要等待一段时间才能生效.
要访问的话直接输入类似我这种格式:
https://joeforkyou.github.io/
自此一个简单的静态博客就搭建好了
搭建完后续的工作就是建立分类和搜索索引,这个我打算单独写一篇文章