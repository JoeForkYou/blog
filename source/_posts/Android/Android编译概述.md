---
title: Android编译概述
tags: Android
abbrlink: 6664
date: 2024-11-10 02:22:33
---

所有的编译都要先source build/envsetup.sh

然后lunch 对应的产品。

整编就直接make

# 1 概述

- 在Andorid 7.0 之前都是使用GNU make,模块编译脚本使用Android.mk;

- 之后,编译使用ninja,由kati工具把Andorid/mk转换为构建规范文件buildxxx.ninja;

- Android 8.0 开始,引入编译脚本Android.bp,及工具blueprint和soong用于把Android.bp转换为buildxxx.ninja.

  Android 编译脚本主要为Android.mk和Android.bp,在编译过程中都会转换为buildxxx.nija构建文件,加入到ninja构建系统中参与编译.

  buildxxxx.ninja文件生成在out目录中,文件大小比较大,包含了编译中的所有配置信息.

|      |                                               |
| ---- | --------------------------------------------- |
| m    | 编译整个源码,可以不用切换根目录               |
| mm   | 编译当前目录下的源码.不包含他们的依赖模块     |
| mmm  | 编译指定目录下的所有模块,不包含他们的依赖模块 |
| mma  | 编译当前目录的下的源码,包含他们的依赖模块     |
| mmma | 编译指定目录下的所有模块.包含他们的依赖项目   |

编译环境初始化.

由命令source build/envsetup.sh完成

其中envsetup.sh主要做了下面几个事情.

- 定义一些lunch /m /mm /mmm /provision等函数.

- 确定当前的shell 环境.建立shell命令

- 从device/vendor/product等目录遍历搜索vendorsetup.sh, 并source 进来

- 将下面的bash文件导入到当前环境中

  system/core/adb/adb.bash,

  system/core/fastboot/fastboot.bash,

  tools/asuite/asuite.sh

# 1 image

像system/vendor/dtbo/boot 这些

可以直接

```
make systemimage
make bootimage
make dtboimage
make vendorimage
make cts
```

# 2 framework

framework部分内容是很复杂的一块的内容.

关于cameraservice的部分可以用ninja编译 ,jni的部分也可以直接mma或者找到对应的so去编译

下列命令是单编译对应的so.对于所有模块都是可以的.需要注意的是这种编译是不加依赖项的.所以会出现修改的Android.mk不生效. 对应的ninja工具需要在对应的项目内寻找.

```
 ./prebuilts/build-tools/linux-x86/bin/ninja -f out/combined-vnd_xxxx.ninja libcameraservice
```
这个路径下是apk层直接调用的硬件接口.可以用如下的命令直接编译生成framework.jar包.

frameworks/base/core/java/android/hardware/

```
make framework-minus-apex
```

adb push framework.jar system/framework/

同时删除设备中system/framework目录下

oat,arm,arm64的三个文件夹.

然后adb reboot. 不删除以上的三个文件，系统会一直处在开机动画中无法打开.

frameworks/base/Android.bp的相关编译规则如下:

```
java_library {
    name: "framework-minus-apex",
    defaults: ["framework-minus-apex-defaults"],
    installable: true,
    // For backwards compatibility.
    stem: "framework",
    apex_available: ["//apex_available:platform"],
    visibility: [
        "//frameworks/base",
        // TODO(b/147128803) remove the below lines
        "//frameworks/base/apex/appsearch/framework",
        "//frameworks/base/apex/blobstore/framework",
        "//frameworks/base/apex/jobscheduler/framework",
        "//frameworks/base/packages/Tethering/tests/unit",
        "//packages/modules/Connectivity/Tethering/tests/unit",
    ],
    errorprone: {
        javacflags: [
            "-Xep:AndroidFrameworkBinderIdentity:ERROR",
            "-Xep:AndroidFrameworkCompatChange:ERROR",
            "-Xep:AndroidFrameworkUid:ERROR",
        ],
    },
}
```



# 3 selinux
adb shell setenforce 0会解放selinux权限
Android 的selinux权限路径,但是这个是总的. 不同平台的编译本质上是编译这个路径.

system/sepolicy

```shell
make selinux_policy
将编译生成的.cil相关文件push到设备中重启.
adb push vendor/etc/selinux/* vendor/etc/selinux
```





