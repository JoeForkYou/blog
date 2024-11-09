---
title: meta工具acdk获取camera
tags:
  - hal
  - camera
  - meta
  - acdk
abbrlink: 8765
date: 2024-11-10 04:29:50
categories: [hal,camera,meta,acdk]
---

# 1 背景

meta工具需要获取到cameraModuleInfo,然后把这个信息给到工具，工具需要字符串匹配检查.(**meta这个工具自检就应该被废弃，太没有用了**)

对于我们camera驱动来说，要完成的工作无非两点，一个就是创建节点，给予hal和上层ap的权限(selinux),其二就是把这个信息存好给返回给工具.

对于这个需求的难点.如果对我来说就是camerahalserver要在关机状态下先自启一遍，保证把相关的Camera info先拷贝给节点.因为关机状态下的cameraProvider是不启动的.

关机下init.rc 去启动cameraProvider的流程，我到现在都懒得去整理，没有好一点的项目练手这部分，找机会再说.

# 2 流程

节点的创建，我这边不想再说了，这边做的节点就是一坨屎.

老子要是有空就全部删了 重做.

下面主要描述下meta的工作流程和camera的资源获取.

meta的路径:

vendor/mediatek/proprietary/hardware/meta

我这边主要先关注这个文件**FtModule.cpp**

vendor/mediatek/proprietary/hardware/meta/common/src/FtModule.cpp

获取节点信息的代码非常简单，打开这个节点，获取copy过来就可以了.

```cpp
#define CAM_INFO_PATH  "/proc/cameraModuleInfo"
static unsigned char read_cam_info(char* peer_buf)
{
	char buf[255];
	ssize_t byte_read = 0;
	int inode = 0;

	if (peer_buf == NULL)
		return META_FAILED;

	inode = open(CAM_INFO_PATH, O_RDONLY);
	if (inode < 0) {
		META_LOG("[Meta][FT] read_cam_info open %s fail!", CAM_INFO_PATH);
		return META_FAILED;
	}
	memset(buf, 0, sizeof(buf));
	byte_read = read(inode, buf, sizeof(buf) - 1);

	META_LOG("[Meta][FT] read_cam_info: %s", buf);
	close(inode);

	sprintf(peer_buf, "%s", buf);
	return META_SUCCESS;
}
```

然后在这个函数执行对应cmd

```c++
void FtModCustomer::exec(Frame **pFrm*)
    ...
    switch(req->cmd.m_u1Dummy)
        ...
            case 4:
			META_LOG("[Meta][FT] read cam info");
			if(!acdkIFInit()){ //跟踪这个 
			    ft_cnf.status = read_cam_info(peer_buf); //获取节点信息
			}else{
			    ft_cnf.status = META_FAILED;
			}
```

acdkIFInit 中使用了ioctrl 我自定义的ACDK\_CMD\_SET\_HAL\_INIT 来完成camerahal 的信息自启动获取.

```cpp
static int acdkIFInit()
{
    ACDK_FEATURE_INFO_STRUCT rAcdkFeatureInfo;
    bool bRet;
    unsigned int u4RetLen;
    int srcDev = 1;
    //====== Create ACDK Object ======
    if(Mdk_Open() == false)
    {
        META_LOG("Mdk_Open() Fail");
        return -1;
    }
    
    //====== Select Camera Sensor ======
    rAcdkFeatureInfo.puParaIn = (MUINT8 *)&srcDev;
    rAcdkFeatureInfo.u4ParaInLen = sizeof(int);
    rAcdkFeatureInfo.puParaOut = NULL;
    rAcdkFeatureInfo.u4ParaOutLen = 0;
    rAcdkFeatureInfo.pu4RealParaOutLen = &u4RetLen;

    META_LOG("%s : srcDev:%d\n",__FUNCTION__,srcDev);
    bRet = Mdk_IOControl(ACDK_CMD_SET_SRC_DEV, &rAcdkFeatureInfo);
    if (!bRet)
    {
        META_LOG("ACDK_FEATURE_SET_SRC_DEV Fail: %d\n",srcDev);
        return -1;
    }

    bRet = Mdk_IOControl(ACDK_CMD_SET_HAL_INIT, &rAcdkFeatureInfo); //主要是跟踪这个位置的代码
    if (!bRet)
    {
        META_LOG("ACDK_CMD_SET_HAL_INIT Fail: %d\n",srcDev);
        return -1;
    }
    else
    {
        META_LOG("ACDK_CMD_SET_HAL_INIT ok: srcDev%d\n",srcDev);
    }
    
    return 0;

}
```

在这个文件中新添加对应的cmd(**ACDK\_COMMAND\_END**)

vendor/mediatek/proprietary/hardware/mtkcam/include/mtkcam/main/acdk/AcdkCommon.h

```c++
    ++ACDK_CMD_SET_HAL_INIT,
    ACDK_COMMAND_END
}eACDK_COMMAND;
```

好了 到这部基本就可以使用这个cmd.

接下来就是通过这个cmd 做对应的操作了

这个ioctrl 的cmd的

vendor/mediatek/proprietary/hardware/mtkcam/main/acdk/v4.0/src/acdk/AcdkMain.cpp

**Mdk\_IOControl**的实际实现是通过一下的这个函数.直接在里面加我需要的camera hal的操作即可.

```c++
AcdkMain::sendcommand


+    else if(a_u4Ioctl == ACDK_CMD_SET_HAL_INIT)
+    {
+        ACDK_LOGE("[meta] ioctrl a_u4Ioctl=%d",a_u4Ioctl);
+        err = sensorInit();
+        if (err != ACDK_RETURN_NO_ERROR)
+        {
+            ACDK_LOGE("[meta] Sensor setting Fail. err(0x%x)",err);
+            err = ACDK_RETURN_API_FAIL;
+        }
+        else
+        {
+            ACDK_LOGE("[meta] Sensor setting ok");
+        }
+
+        err = getSensorInfo();
+        if(err != ACDK_RETURN_NO_ERROR)
+        {
+            ACDK_LOGE("[meta] getSensorInfo error(0x%x)",err);
+            err = ACDK_RETURN_API_FAIL;
+        }
+        else
+        {
+            ACDK_LOGE("[meta] getSensorInfo ok");
+        }
+
+        //====== Initialize AcdkMhal ======
+        err = m_pAcdkMhalObj->acdkMhalInit();
+        if(err != ACDK_RETURN_NO_ERROR)
+        {
+            ACDK_LOGE("[meta] acdkMhalInit Fail(0x%x)", err);
+        }
+        else
+        {
+            ACDK_LOGE("[meta] acdkMhalInit ok");
+        }
+
+        err = m_pAcdkMhalObjEng->acdkMhalInit();
+        if(err != ACDK_RETURN_NO_ERROR)
+        {
+            ACDK_LOGE("[meta] acdkMhalInit eng Fail(0x%x)", err);
+        }
+        else
+        {
+            ACDK_LOGE("[meta] acdkMhalInit eng ok");
+        }
+    }
     else if(a_u4Ioctl == ACDK_CMD_SET_OPERATION_MODE)
     {
         eACDK_OPERA_MODE eOpMode = ACDK_OPT_NONE_MODE;
```

# 3 编译debug

## 3.1 meta

对于meta部分:直接看这个Android.mk

vendor/mediatek/proprietary/hardware/meta/common/Android.mk

    LOCAL_MODULE:=meta_tst

目标模块名字是这个.我本来以为是编译出来so或者a文件，结果编译出来一个二进制文件就叫meta\_tst.

直接push到vendor/bin/中就可以. meta的工具是直接通过这个bin 来下cmd交互的.我也不想研究，早该弃用的东西没什么软用.这一套和车机ais\_server和ais\_be\_server的交互倒是很相似的.

    adb push  meta_tst vendor/bin/meta_tst

理论上这个meta\_tst可以直接在手机上运行. 当然是直接运行其main函数.而不是通过工具来操作的.

这个我不想梳理，没啥用. 知道改meta路径下的文件push  meta\_tst到 vendor/bin/即可.

## 3.2 acdk

我们本质上是通过acdk已经写好的函数来完成操作的.

acdk代码路径:

外部接口头文件说明:

vendor/mediatek/proprietary/hardware/mtkcam/include/mtkcam/main/acdk

该so库主要实现代码:

vendor/mediatek/proprietary/hardware/mtkcam/main/acdk

这边主要用到v4.0(**能观察到mtk源码和很多其他的acdk，那些都是针对单独模块做的.我这个是最源生的**.)

然后我们看到对应的mk文件:

vendor/mediatek/proprietary/hardware/mtkcam/main/acdk/v4.0/src/Android.mk

```makefile
LOCAL_MODULE := libacdk
```

从这里就能知道编译生成的文件是libacdk.so

push路径

```shell
adb push libacdk.so vendor/lib64/
```

同理需要杀掉cameraProvider才能生效.

mtk对cameraProvider进行了封装,是camerahalserver.所以杀掉camerahalserver即可

从上述内容基本了解到如何编译，但是怎么debug呢？这是关机的状态.如果一直拿meta工具去debug,要不停地重启和抓log.效率极其低下。因为进meta模式机器经常卡死，要好久才能开机.

于是乎，这边就用到mtk自己编写的测试文件.平台的专业就专业在每个模块都是可以做单独的测试，都写好了对应的接口，push到vendor/bin下运行即可.

对应的test文件路径也在其对应的模块下.

示例如下:

**vendor/mediatek/proprietary/hardware/mtkcam/main/acdk/v4.0/src/test**

main.cpp 函数入口，我自己debug的时候基本删的很干净，一来是方便debug，二来是跑得快.只跑我要的代码块即可.

main中其他的我不关注.只要关注这个

    ret *=* main_testMdk(argc, argv, &(crcResults[loop]));

main\_testMdk在test\_mdk.cpp中实现

这个编译是生成acdk\_camshottest  然后push到vendor/bin/

直接在手机里运行即可

```shell
./vendor/bin/acdk_camshottest  
```

