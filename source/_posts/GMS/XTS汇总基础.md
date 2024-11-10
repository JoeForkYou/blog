---
title: XTS基础汇总
tags:
  - cts
  - Android
  - GMS
abbrlink: 5338
date: 2024-11-10 14:01:39
categories: [Android, GMS,cts]
---
参考谷歌官网：
https://source.android.google.cn/compatibility/tests/development
# 1 XTS 概述
GMS全称为GoogleMobile Service，即谷歌移动服务。我们常说的XTS其实就谷歌认证
GMS是Google所提供的一系列移动服务，包括开发用的一系列服务和用户所用的Google Apps。
Maps与Location：地理位置相关服务，AOSP也包括一个简易的Location服务，这是升级版，有用但并非必要，国内也有百度、高德等提供了类似的API；
Games、Play Services、In-app Billing、Play Distribution：与Google Play相关的服务，毫无疑问这个在国内是用不到的，但如果要在Google Play上发布应用，则非常有用；**(GL和IN做大量测试的原因)** （GL 国外发型的版本: IN :印度发型的版本  有的项目是发往欧美或者东南亚的，都需要经过认证）
Google+、Drive、Cloud Platform、Cloud Messaging：与Google的社交网络和云平台相关的服务，前三个在国内也基本上用不到，第四个是推送服务对开发者非常有用，但国内有很多类似的第三方服务可替代；
Cast、Wallet、Ads：这里是Google推出的与Android平台关系不大的服务，Ads广告对开发者有用，但国内也有很多的移动广告平台和服务。
这些服务不是构建一个Android App所必需的，也可以使用其他的服务替代，因此，没有GMS对国内手机厂商影响没有想象那么大。**(CN少测的原因)**
Google Apps则包括Gmail、Google Maps等Google官方应用，这些系统应用对于一个完善的Android设备是很重要的，但是手机厂商也可以使用自己的或者第三方应用替代。
整个Android平台可以看成是：AOSP+GMS，AOSP（安卓开源项目）是所有手机厂商可以免费获得的开源代码，但GMS则需要Google同意授权才行。
Google给GMS认证设置了比较高的门槛。首先要通过CTS兼容性测试（Compatible Test Suite），一般而言所有的Android厂商都必须通过这个认证，否则会出现兼容性问题。这个认证一般由手机厂商自己做，然后提交结果给Google。
**AOSP是工具，GMS则是服务**

## 1.1 CTS
谷歌官网:
https://source.android.google.cn/compatibility/cts

CTS是Compatibility Test Suite的缩写,即兼容测试，是Google为Android设备制造商免费提供的兼容性测试套件。

CTS 是一个自动化测试套件，包括两个主要的软件组件：

1.CTS Trade Federation

自动化测试框架会在桌面设备上运行，并管理测试执行情况。此框架可实现对多个被测设备 (DUT) 进行分片测试。您还可以利用套件重试功能仅重试失败的测试而不是完整的套件，从而大大减少重新运行所花的时间。

2.单独的测试用例会在 DUT 上执行。

测试用例采用 Java 语言编写为 JUnit 测试，并打包为 Android .apk 文件，以在实际目标设备上运行。

APP层跟Framework层在设计上是分开的，但通过CTS测试，确保了APP无Android Framework之间有一致的调用接口（API），这使得APP开发者编写的同一款程序可以运行在不同系统版本（向前兼容）、不同硬件平台、不同产商制造的不同设备上。**如这个示例图.**



![1.png](https://i.loli.net/2021/11/13/UbfqzP8gvRhTpOy.png)


CTS定义了众多Android设备必须满足的技术指标，以确保每台通过CTS认证的设备，都可以顺利运行Google Play中出售的软件。（并不是每个软件都可以在所有Android设备上运行，Google Play仅显示可以运行在该Android设备上的应用，并且还受到当地法律法规的限制。）
**CTS的目的就是让Android设备开发商能够开发出兼容性更好的Android设备。**

通过以上概述可以知道这些认证的本意是：

1.让APP提供更好的用户体验。用户可以选择更多的适合自己设备的APP。让APP更稳定。

2.让开发者设计更高质量的APP。

## 1.2 GTS
GTS的全称是Google Mobile Services Test Suite，所谓的Google Mobile Services即谷歌移动服务
谷歌移动服务提供了Search、 Search by Voice、Gmail、Contact Sync、 Calendar Sync、Talk、 Maps、 Steet View、 YouTube、 Android Market (Play store)等服务，当用户使用谷歌时，谷歌可以把各种广告嵌入到谷歌的服务中。
这些服务依赖于网络.

## 1.3 VTS
Android 目前有一个比较明显的缺点是**设备升级到新版本系统所要花费的时间太长（比如从 Android 6.0 升级到 Android 7.0）**。通常在由 Google 发布新版本的 AOSP 之后，还需要 SoC 厂商对 HAL 进行升级，以及 OEM 厂商对 HAL 和 Framework  进行升级后，用户才能在设备上收到 OTA 升级包的推送。低端一点的产品甚至在出厂后就不会再进行系统升级了。用户对此抱怨良多。反观竞争对手 iOS 在这方面就做得比较好（但这不代表我支持 iOS)
 **为了解决这个问题，于是 Google 发起了 Project  Treble 项目**2017 年 5 月 12 日，官方在”Developers Blog”上向公众介绍了这一项目并宣布 Android  8.0 中将引入它，但从目前我拿到的描述 Project Treble 的相关文档的修订记录来看，这些文档最早的起草时间可以追溯到 2015 年 10 月 30 日。 
 　　**而 Project Treble 中最重要的就是新增了 Vendor Interface 这一概念，以及相应的 Vendor Test Suite (VTS) 测试。**

------

Project Treble 中引入 Vendor Interface 的目的是将 Android Framework 与 HAL 分开，通过对Vendor Interface进行测试，确保同一个版本的Android Framework可以运行在不同的HAL上，或者Android Framework可以运行在同一个HAL，即保证HAL的向前兼容性。通过这样的Framework/HAL分离设计和接口一致性保证，这就使得8.0版本之后的Android系统在进行升级时，可以直接对Framework进行升级而不用考虑HAL层的改动，从而缩短了用户手上设备得到系统升级OTA推送的时间。

## 1.4 GSI
GSI是在VTS环境下用google img测试CTS用例,从andorid 11版本后面迁移道cts环境下跑测，是替换掉自己framework相关的内容，用谷歌原生的system.img
## 1.5 CTS-V
CTS Verifier是CTS兼容性测试的补充。CTS检查的是可以自动化的API和功能，而CTS Verifier是测试在没有手动输入的静态设备上测试这些API和功能，例如音频质量，触摸屏，加速度计，相机等等.这边可以重点关注FOV和尺寸video相关的测试项.

## 1.6 ITS
Android 相机图像测试套件 (ITS) 是 Android 兼容性测试套件 (CTS) 验证程序的一部分，其中包含用于验证图像内容的测试

`scenes=sensor_fusion` 

在传感器融合测试中，将分别针对 AR 和 VR 应用，测试相机和陀螺仪之间的时间戳差异，因此需要按特定轨迹移动相机。
`REALTIME` 功能标记和 VR/AR 应用要求相机/陀螺仪的定时偏差小于 1 毫秒

ITS测试输出:

PASS：测试通过

FAIL：测试失败，必须修复

SKIP：跳过测试项

FAIL*：测试失败，目前可以不修复，但可能在未来的测试中变为强制性

注意:另外还有STS安全补丁包相关的测试,因为和camera无关,就没有提及,GTS大部分依赖的是服务，camera相关性低，也与我们无关,但是有camera测试项目,就顺带一提.


its测试的图表可以用另外一台平板电脑来提供

![平板要求.png](https://i.loli.net/2021/11/29/JcXSDagpFMtvYfn.png)




# 2 环境搭建

## 2.1 CTS

CTS环境搭建只需要一个jdk和aapt

前者这个比较大问相关人员拿吧. 
或者直接去官网拿对应的jdk:

这里放一个11版本的下载链接

https://www.oracle.com/java/technologies/javase/javase11-archive-downloads.html

andorid 11后面要求的jdk>=11  

然后vim ~/.bashrc 把对应的路径配置写入后:wq保存退出,source ~/.bashrc

```shell
#jdk
export JAVA_HOME=/home/ubuntu/Jdk11
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```

然后用以下命令检查jdk版本

```shell
java -version
```

输出打印

```shell
openjdk version "11" 2018-09-25
OpenJDK Runtime Environment 18.9 (build 11+28)
OpenJDK 64-Bit Server VM 18.9 (build 11+28, mixed mode)
```

说明jdk配置成功

cts包下载路径:

https://source.android.google.cn/compatibility/cts/downloads

CTS会使用aapt工具,如下命令安装aapt
```shell
sudo apt install aapt
```
aapt 是什么？谷歌官网有很详细的介绍.

https://developer.android.google.cn/studio/command-line/aapt2?hl=zh-cn

AAPT2（Android 资源打包工具）是一种构建工具. 这里理解下输出的报告就是通过这个工具构建的,当然不仅仅是报告.

两种皆可以直接命令装.检查是否搭建成功直接看基本操作命令中CTS单跑一项即可


## 2.2 ITS/VTS

its和vts同样依赖python包.具体另外有个XTS基础有单独的操作说明.还有GTS需要连接外网,
VTS 不依赖网络

```shell
sudo apt-get install python-dev
sudo apt-get install python-protobuf
sudo apt-get install protobuf-compiler
sudo apt-get install python-virtualenv
sudo apt-get install python-pip
sudo apt-get install python-numpy
sudo apt-get install python-scipy
sudo apt-get install python-matplotlib
sudo apt-get install python-opencv

#pip install pyserial
#pip install serial #如果这两个命令不行就用下面这两个命令.
sudo apt install python-pyudev
sudo apt install python-serial
```
因为ubuntu16和18以及其他系统默认带的python包不一致，请根据实际情况下载其他相关的python

以上完成后直接看基本操作命令ITS部分

其他测试项不依赖任何环境.

# 3 基本操作命令

谷歌官网：

https://source.android.google.cn/compatibility/cts/command-console-v2#ctsv2_reference

手机开发者选项打开，永不锁屏，充电不休眠这些打开.因为测试过程中长时间未操作会影响测试结果的

## 3.1 CTS

进入到cts的目录

```
cd android-cts/tools/
./cts-tradefed   #进入到cts的终端命令里
```

相关的命令

```shell
run cts -m CtsCameraTestCases #cts整跑三个abi的命令
```

我们一般debug只要跑一个abi,使用的cts-dev,会跑测当前设备对应的abi.

上述命令会直接跑侧当前环境下所有相关的abi.

出报告还是要整跑三个abi的


谷歌原话是:
**On 64-bit devices, run the test against only the 32-bit or 64-bit ABI**

**运行默认的 CTS 计划（即完整的 CTS 调用），但跳过前提条件以缩减运行时间，从而对新测试执行迭代开发。这会绕过对设备配置的验证和设置（例如推送媒体文件或检查 Wi-Fi 连接），就如同使用了 --skip-preconditions 选项。此命令还会跳过设备信息收集和所有系统状态检查工具。它还仅在单个 ABI 上运行测试。对于设备验证，请忽略此优化操作并添加所有前提条件和检查。有关要排除的内容，请参阅 cts-dev.xml。**



在测试过程中，CTS 控制台可以接受其他命令。

如果没有连接任何设备，CTS 台式机（或主机）会等到设备连接后再启动测试。如果连接了多台设备，则 CTS 主机将自动选择一台设备。




它还仅在单个 ABI 上运行测试
cts-dev 是跑测当前设备默认abi.

abi(Application Binary Interface，ABI)：
应用程序二进制接口
默认情况下，CTS 会为设备支持的每个 ABI 运行一次测试。


只跑一个命令如下:
```shell
run cts-dev -m CtsCameraTestCases#单跑当前设备默认Module命令
run cts-dev -m CtsCameraTestCase --shard-count 3 -s sn1 -s sn2 -s sn3 #多台设备跑测一个报告
#Android12 以后建议用subplan的方式创建计划表 去跑测试，别用cts-dev的方式.
```

单跑一个测试项命令.加参数-t -s -m 还有list(l d) 这些相关的命令其他XTS测试是一样的.

```shell
run cts-dev -m CtsCameraTestCases -t
android.hardware.camera2.cts.RecordingTest#testVideoSnapshot
```

如果有多个设备挂跑的时候, 用以下命令 加-s 设备序列号

```shell
run cts-dev -m CtsCameraTestCases -s 141190ce0312
```

```shell
adb device #查看设备序列号
```
```shell
l r #list result查看报告结果状态,如下图能看到session 为O(记住要考)
Pass测试项为569
Fail项目为0 
完成测试Module为1
报告结果时间和名字以及其他一些相关信息.
```
![ITS.png](https://i.loli.net/2021/07/15/ZifKRjmh5V7pa6r.png)

```shell
l d #list device列举设备信息，主要关注Allocation 下设备的状态Available为可以使用的状态.
```
![ld.png](https://i.loli.net/2021/07/15/PQeURF45zWOZxEw.png)

```shell
l c #查看运行的command命令
```
![lc.png](https://i.loli.net/2021/07/15/4NfjRGVbmiqvDtW.png)

```shell
l i #查看当前命令运行的时间
```


![li.png](https://i.loli.net/2021/07/15/G7Pbes18xp3vKqC.png)

一般跑测没有那么快一下子全PASS,很有可能受到环境或者其他因素影响导致Fail,所以需要retry

```shell
run retry --retry <session-number> #retry 格式
run retry --retry 0 #这个就是l r 里面报告里面的session,重跑你fail的报告即可，注意版本和机器设备必须是之前出报告的同一个，不然重跑不起来.
#当然你有了这个报告,也可以在其他电脑上重跑
```

生成的报告路径在android-cts/results

subplan的方式进行跑测:

什么是subplan? 是自己创建的跑测计划,一般用于跑测.

```
add subplan --session 10 -n cts_dev --result-type failed #创建一个名未cts_dev的 session10中跑测失败的计划表
```

	#这是参数说明
	add s[ubplan]: create a subplan from a previous session
		Available Options:
			--session <session_id>: The session used to create a subplan
			--name/-n <subplan_name>: The name of the new subplan
			--result-type <status>: Which results to include in the subplan. One of passed, failed, not_executed. Repeatable

```
run cts --subplan cts_dev -s 81926A8J00072 #这是subplan的使用说明,在上述已经创建了一个名未cts_dev的计划，跑测该计划中的内容
```



打开test_result.html即可,其他测试项除ITS外都是在相似的文件路劲下

一般cts debug都是到这个路劲android-cts/testcases

-t -g 强制下载并且打开相关权限.打开这个可以不用去操作打开apk的权限了.可以直接运行相关的命令 
```shell
adb install -t -g CtsCameraTestCases.apk
```

然后需要进入到setting里面

![01.png](https://i.loli.net/2021/07/15/lEU6eTQmrWdApjN.png)

选择app&notifications

<img src="https://i.loli.net/2021/07/16/CE4vAgwNjIbLpSR.png" alt="01.png" style="zoom: 66%;" />
选择App info
<img src="https://i.loli.net/2021/07/16/UKrLbaGpwHCf5XO.png" alt="01.png" style="zoom:67%;" />

找到我们cts 测试apk,  android.camera.cts
<img src="https://i.loli.net/2021/07/16/iAgW2XIExwnlsmz.png" alt="01.png" style="zoom:67%;" />

把所有的权限都打开

<img src="https://i.loli.net/2021/07/16/d6lYgqXWzbNLv2a.png" alt="01.png" style="zoom:67%;" />



然后在终端上运行以下类似格式的命令就可以了.  

```shell
adb shell am instrument -e class android.hardware.camera2.cts.RecordingTest#testVideoSnapshot --abi arm64-v8a -w android.camera.cts/androidx.test.runner.AndroidJUnitRunner

adb shell am instrument -e class 对应的测试项目 --abi arm64-v8a -w android.camera.cts/androidx.test.runner.AndroidJUnitRunner
```

运行结果如下即便就是PASS,否则为Fail

![01.png](https://i.loli.net/2021/07/16/vtLFJ5Ec73OlX4Z.png)


## 3.2 VTS

vts需要连接外网,再一次强调.
查看gsi版本日期

```shell
strings system.img | grep ro.build.version.security_patch
adb shell getprop ro.build.version.security_patch
```
![补丁包的时间.png](https://i.loli.net/2021/07/15/Srp6vI2cX3RAb1O.png)
预备准备好对应的system.img 和boot-debug.img

要刷入这两个镜像要先解锁设备.一般默认是上锁的无法烧录镜像

```shell
adb reboot bootloader # 进入fastboot模式
#有的项目解锁方式
fastboot oem unlock
#有的项目解锁方式
fastboot flashing unlock
```

解锁完就能烧录对应的镜像,以下命令往下跑就行了

```shell
fastboot reboot fastboot
fastboot flash system system.img # 需要进入fastboot 下烧录.bootloader下没有system的分区
fastboot reboot bootloader
fastboot flash boot boot-debug.img #boot 分区在bootloader模式下.
fastboot reboot
```

进入到vts的tool目录下运行

```
./vts-tradefed
```

vts整跑命令,vts 我们就关注这三个模块的部分

注意，并不是三个都一定存在的项目.需要根据实际项目来看.

比如展讯平台上没有2_5的接口，所以实际无法跑测.

```shell
run vts --include-filter VtsHalCameraProviderV2_4Target --include-filter VtsHalCameraProviderV2_5Target --include-filter VtsHalCameraServiceV2_0Target --skip-preconditions

run vts --skip-preconditions  --include-filter -m VtsHalCameraServiceV2_0TargetTest VtsHalCameraServiceV2_0Target
```

单跑对应的模块

```shell
run vts -m VtsHalCameraProviderV2_4Target -s 1769E47E
run vts -m VtsHalCameraProviderV2_5Target -s 1769E47E
run vts -m VtsHalCameraServiceV2_0Target -s 1769E47E
```

单跑单个测试项目

```shell
run vts -m VtsHalCameraProviderV2_4Target -t CameraHidlTest.processCaptureRequestBurstISO(legacy/0)_64bit
```

其他操作命令和CTS一样



## 3.3 GSI

GSI也是要刷入system.img.但是不用刷boot.img./(andorid11后都要刷.)

一般system分区会做在fastboot 模式下，但是后来有的项目好像是得进bootloader刷system.这个要确定好，不然刷进去就无法
进入系统了.

GSI刷镜像按照如下的操作即可.

```shell
adb reboot bootloader #刷入对应的谷歌镜像system.img
fastboot devices
fastboot flashing unlock
fastboot reboot fastboot
fastboot erase system
fastboot flash system system.img #谷歌官网获取
fastboot -w
fastboot reboot bootloader
fastboot flashing unlock
#按音量上键
fastboot flashing lock 
fastboot flashing unlock
fastboot reboot
```

打开开发者选项 , 打开 stay awake 和 USB debugging
andorid 11 后 gsi 在 cts/vts 的运行环境中跑测。

android10以前以及10版本都是在vts的运行环境中跑测试.11以及11以后谷歌将测试挪到了11中

GSI整跑命令

```shell
run cts-on-gsi -m CtsCameraTestCases
```



## 3.4 ITS

### 1 准备

我们可以编译对应的项目生成CtsVerifier.apk或者直接在ITS包里面把测试apk下载到手机中

```
./mk -p 项目名字 -s -v userdebug -m mma -o cts/apps/CtsVerifier
```

```
adb install CtsVerifier.apk
```

进入到手机apk，所有的权限都打开.

操作步骤如下:

![01.png](https://i.loli.net/2021/07/15/lEU6eTQmrWdApjN.png)

选择app&notifications

<img src="https://i.loli.net/2021/07/16/CE4vAgwNjIbLpSR.png" alt="01.png" style="zoom: 66%;" />

<img src="https://i.loli.net/2021/07/16/UKrLbaGpwHCf5XO.png" alt="01.png" style="zoom:67%;" />

找到CTS Verifer 测试apk,

![01.png](https://i.loli.net/2021/07/16/2BM7Imz5KJV8pH1.png)

打开其所有的权限即可
![01.png](https://i.loli.net/2021/07/16/ZsL7hkT2zERSu1v.png)

### 2 场景说明

| 场景          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| 场景0         | 无任何要求                                                   |
| 场景1         | 相机位于三脚架上, 指向一个静态场景, 其中包含灰色卡和白色背景, 在恒定 (稳定) 相对明亮的光照源下。这是 CTS 验证程序物理设置上面描述的场景。镜头视野中，灰卡大致放在中间，周围为白色背景 |
| 场景 2        | 这是测试人脸检测的场景。相机位于三脚架上, 指向一张包含3人脸的静态图片, 在恒定 (稳定) 相对明亮的照明光源下。 |
| 场景3         | 这是测试图像清晰度的场景。相机位于三脚架上, 指向包含某些边缘的静态图片, 如打印的 ISO 12233 图表。现场应在一个恒定 (稳定) 相对明亮的照明源。 |
| 场景4         | 这是测试纵横比的场景。相机位于三脚架上, 指向一个静态测试页, 其中包含一个黑色圆圈和一个方块。现场应在一个恒定 (稳定) 相对明亮的照明源 |
| 场景5         | 这是测试镜头着色和颜色均匀性的场景。在摄像机前放置一个扩散器(毛玻璃)。相机位于三脚架上，指向恒定的 (稳定) 相对地明亮的照明源。<br/>我们这边就用的一张白色餐巾纸(A4纸也可以，只要白色)代替了扩散器，将镜头对着光源，用餐巾纸挡住镜头即可。 |
| sensor_fusion | 马达灯箱                                                     |

注意，在andorid 11后这些场景会进一步细分和增加，具体看对应的CTSVerfier的包里面,路劲如下,这个路劲下面也有对应场景的图片.pdf

```shell
ubuntu@ubuntu:~/GMS/r4/CameraITS/tests
```



### 3 命令

终端需要source its的环境.只要source一次，终端没有关闭不用再次source.新开终端要重新source

```
source build/envsetup.sh 
```



ITS 整跑命令如下:

```shell
python tools/run_all_tests.py device=008bcdcf0405   camera=1 scenes=5 #更改成对应的camera和场景即可
python tools/run_all_tests.py device=008bcdcf0405   camera=1 scenes=scenes_fusion #如果只有一台机器可以不加device参数
python tools/run_all_tests.py camera=0 scenes=sensor_fusion rot_rig=04d8:fc73:1 #rot_rig后面加马达相关的参数，这个用默认即可不用更改

python tools/run_all_tests.py camera=0 scenes=0 tmp_dir=~/XTS/CtsVerifier_r6/CameraITS/outfile/ 
#设置输出的路径
```

整跑会在/tmp/xxx 目录下创建对应的跑测相关的文件.其中summary.txt文件记录了整跑的结果，如果没有则PASS，如果有fail项目,需要单跑检查

单跑命令,python 运行对应的场景，对应的测试项即可

```shell
python tests/scene1_1/test_ev_compensation_advanced.py camera=0
```
关于sensor_fusion 场景，如果马达不会转动，则需要赋予马达权限

```shell
sudo chmod +x /dev/ttyACM0
```

### 4 创建报告

单跑是空的,显示在终端上。创建报告需要全 pass ,有一项没跑或者 fail, 报告出来都是 fail ,并且没有详
细的信息

```shell
#its 跑测全部PASS后要点击绿色√后往下操作
adb shell settings put global hidden_api_policy 1
adb shell appops set com.android.cts.verifier android:read_device_identifiers allow
adb shell appops set com.android.cts.verifier MANAGE_EXTERNAL_STORAGE 0 (保存报告之前)
在cts界面点击右上角保存报告
adb pull /storage/emulated/0/verifierReports ~/桌面/
```

注意这个报告的结果只能用ie浏览器打开.

## 3.5 GTS

GTS需要电脑和手机都连接外部的网络.

GTS 跑测命令如下


```
run gts -m GtsCameraTestCases -s xxx
run gts -m GtsCameraTestCases -s 008bcdcf0405
```