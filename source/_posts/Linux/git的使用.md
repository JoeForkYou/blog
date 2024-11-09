---
title: git的使用
tags: git
abbrlink: 42437
date: 2024-11-10 00:50:40
---

# 1.git 下载
最好是更新下镜像源

```powershell
sudo apt-get update
```
下载git
```powershell
sudo apt-get install git
```
![](https://i-blog.csdnimg.cn/blog_migrate/57610ea2389eed7f411f029a43c3c8b5.png)
检测git下载成功的版本

```powershell
git version
```
版本显示正确后执行再执行下一步
![](https://i-blog.csdnimg.cn/blog_migrate/804594cf93c6f0f70dbe0f7e487fed39.png)
# 2 ssh 
ssh具体内容参考百度词条[ssh](https://baike.baidu.com/item/SSH/10407)
执行

```powershell
sudo apt-get install ssh 
```
显示如下
![](https://i-blog.csdnimg.cn/blog_migrate/2d7bb5fce1275523d01140a2a1c01424.png)
查看ssh服务
```powershell
ps -e | grep sshd
```
显示sshd的话表示ssh-server已经启动
![](https://i-blog.csdnimg.cn/blog_migrate/ccac9456c7008cd8439c45c645c8c26e.png)
生成ssh-key
```powershell
ssh-keygen -t rsa -C "你的邮箱@qq.com"
```
生成后默认就行，直接回车生成文件隐藏
用下面命令检测是否在根目录下生存(上面建议默认，是直接在根目录下生成的)

```powershell
 ls -al ~/.ssh
```
![](https://i-blog.csdnimg.cn/blog_migrate/de0994ad8d808f189ed4ec9030615e3b.png)
然后打开显示隐藏文件
![](https://i-blog.csdnimg.cn/blog_migrate/eb47eba16f9a712bd8047524f5f5eb84.png)找到
![](https://i-blog.csdnimg.cn/blog_migrate/bcc065d14a29d13598f4d56647b41ed2.png)
再进入到文件夹下找到以下文件，这个是公钥。
打开这个文件将其内容复制(用记事本或者vim打开都行)
![](https://i-blog.csdnimg.cn/blog_migrate/7c0c0e8b5c356f45ba34f74f38fd475a.png)
![](https://i-blog.csdnimg.cn/blog_migrate/c787fd589d622eef983fe42f02857eff.png)打开你的github帐号，进入你的settings
![](https://i-blog.csdnimg.cn/blog_migrate/c948ad84b69384cd61385fe7347ad056.png)找到ssh
![](https://i-blog.csdnimg.cn/blog_migrate/978cb04f1e14a96e65df43568252c90d.png)
新建一个ssh
![](https://i-blog.csdnimg.cn/blog_migrate/df880886f641c1af1f64f360b189b121.png)将复制的内容粘贴进去后便是如上显示
# 3 git 使用
## 3.1 新建仓库
![](https://i-blog.csdnimg.cn/blog_migrate/d9eb93e958f4078d7cbca14191487f6d.png)
这部分比较简单(其实都不难)，直接看图说话吧。名字和描诉整干净后直接创建仓库

![](https://i-blog.csdnimg.cn/blog_migrate/4eb16a6a5731d31d61aea7dca0a0c4eb.png)然后把地址复制下来
![](https://i-blog.csdnimg.cn/blog_migrate/460b2d3f06233ee1b0a8aaade07a981e.png)
## 3.2 git它！！！git就万事了
在你自己的文件夹下git clone远程仓库
```powershell
git clone 网址
```
然后进入到目录里初始化他
![](https://i-blog.csdnimg.cn/blog_migrate/36735234074ff0312a22d082cb636779.png)
在该目录下创建你自己的文件,这个随便你怎么建立，touch 是创建命令

```cpp
touch README.md
```
git add .  是有个空格后再家一个点。直接该该目录下的文件添加到暂存区域
git add 文件名  是直接将相应的文件添加到暂存区域
```powershell
git add .
#git add README.md
```
提交本次修改
git commit

```powershell
git commit -m "add readme file" #提交本次修改
```
推送到远程仓库
格式为 git push (brash) 我这里是直接推送到origin master
```powershell
git push origin master	#推送到远程仓库
```



![](https://i-blog.csdnimg.cn/blog_migrate/e1c8ad0468251a3f08b6c1b1de785d3e.png)
输入你名字和密码
![](https://i-blog.csdnimg.cn/blog_migrate/524377bf2fdf3cca28cc2a639e873c5a.png)
刷新你的仓库，内容就提交上去了
![](https://i-blog.csdnimg.cn/blog_migrate/d4fb82ae9b4940d3130cff0c98c91b05.png)


更多笔记请访问
[JoeNero私人博客](https://joenero.github.io)
参考链接
[参考链接1](https://blog.csdn.net/wxy540843763/article/details/80197301)
[参考链接2](https://blog.csdn.net/qicheng777/article/details/74724015)
