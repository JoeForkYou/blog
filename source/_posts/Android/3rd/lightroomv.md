---
title: lightroomv
tags: 3rd
abbrlink: 6941
date: 2024-11-10 03:28:45
categories: 3rd
---

# 1 lightroomv

三方apk lightroomv 拍照. 一次性只能拍照4张，无法生成第5张图片.

很明显能看到异常的时候会有如下的报错log,到这里hal就没有收到任何拍照请求.

```shell
#上一张拍照请求到hal,下述log. pic_req =1 ，拍照请求到hal 查收了
05-13 17:32:18.966   560 12747 D Cam3HWI : 2317, processCaptureRequest: camId=0, bufs_num=2, frame_num=49, cap_intent=2, pic_req=1, first_regular_req=0
#已经报错
05-13 16:53:39.869  8798 12489 W ImageReader_JNI: Unable to acquire a buffer item, very likely client tried to acquire more than maxImages buffers
#下面就无法打印出对应的拍照请求.
```



# 2 JNI
我们camera相关的会涉及如下路径的JNI
frameworks/base/media/jni      (这个是多媒体相关的，主要涉及到拍照的image相关的内容)

编译的so 为libmedia_jni.so

frameworks/base/core/jni

编译的so为libandroid_runtime.so

具体查看编译什么可以去看对应目录文件下Android.bp文件

```makefile
cc_library_shared {
    name: "libandroid_runtime",
    host_supported: true,
    cflags: [
```

```makefile
cc_library_shared {
    name: "libmedia_jni",

```

举个例子
1.android_hardware_Camera.cpp 这个文件中如下的内容:

```cpp
static const JNINativeMethod camMethods[] = {
  { "getNumberOfCameras",
    "()I",
    (void *)android_hardware_Camera_getNumberOfCameras },
```
对于java hardware 中的虚函数  getNumberOfCameras使用的实际方法是通过这个JNI的方法映射过来. 实际使用函数是这个文件中的android_hardware_Camera_getNumberOfCameras

2.

`ImageReader`允许应用程序直接获取渲染到`surface`的图形数据，并转换为图片

更为详细的内容可以查阅https://developer.android.google.cn/reference/android/media/ImageReader.html

frameworks/base/media/java/android/media/ImageReader.java中

```java
private synchronized native int nativeImageSetup(Image i);
```

实际上调用的是如下位置的内容.下述是其映射关系.

frameworks/base/media/jni/android_media_ImageReader.cpp

```c++
static const JNINativeMethod gImageReaderMethods[] = {
    {"nativeClassInit",        "()V",                        (void*)ImageReader_classInit },
    {"nativeInit",             "(Ljava/lang/Object;IIIIJ)V",  (void*)ImageReader_init },
    {"nativeClose",            "()V",                        (void*)ImageReader_close },
    {"nativeReleaseImage",     "(Landroid/media/Image;)V",   (void*)ImageReader_imageRelease },
    {"nativeImageSetup",       "(Landroid/media/Image;)I",   (void*)ImageReader_imageSetup }, 
    {"nativeGetSurface",       "()Landroid/view/Surface;",   (void*)ImageReader_getSurface },
    {"nativeDetachImage",      "(Landroid/media/Image;)I",   (void*)ImageReader_detachImage },
    {"nativeDiscardFreeBuffers", "()V",                      (void*)ImageReader_discardFreeBuffers }
};
```



![JNI.png](https://s2.loli.net/2022/05/13/13qiodPny2csMCW.png)



# 3 追溯原因

报错内容:

```shell
05-13 16:53:39.869  8798 12489 W ImageReader_JNI: Unable to acquire a buffer item, very likely client tried to acquire more than maxImages buffers
```

报错位置:frameworks/base/media/jni/android_media_ImageReader.cpp.我们可以把相关的logv 都改成可以打印出的loge看具体的差异.

```c++
static jint ImageReader_imageSetup(JNIEnv* env, jobject thiz, jobject image) {
    ALOGV("%s:", __FUNCTION__);
    JNIImageReaderContext* ctx = ImageReader_getContext(env, thiz);
    if (ctx == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException",
                "ImageReader is not initialized or was already closed");
        return -1;
    }

    BufferItemConsumer* bufferConsumer = ctx->getBufferConsumer();
    BufferItem* buffer = ctx->getBufferItem();//看这个函数的调用
    if (buffer == NULL) {
        ALOGW("Unable to acquire a buffer item, very likely client tried to acquire more than"
            " maxImages buffers"); //这是报错的位置.我们看buffer 为NULL,打印了这个log
        return ACQUIRE_MAX_IMAGES;
    }
```

```c++
BufferItem* JNIImageReaderContext::getBufferItem() {
    if (mBuffers.empty()) {
        return NULL;  //<==== 这是实际返回的NULL,说明mBuffers.empty()是为空了.然后往下面的内容看构造函数.
    }
    // Return a BufferItem pointer and remove it from the list
    List<BufferItem*>::iterator it = mBuffers.begin();   //用一块，擦一块
    BufferItem* buffer = *it;
    mBuffers.erase(it);
    return buffer;
}
```

```c++
JNIImageReaderContext::JNIImageReaderContext(JNIEnv* env,
        jobject weakThiz, jclass clazz, int maxImages) :
    mWeakThiz(env->NewGlobalRef(weakThiz)),
    mClazz((jclass)env->NewGlobalRef(clazz)),
    mFormat(0),
    mDataSpace(HAL_DATASPACE_UNKNOWN),
    mWidth(-1),
    mHeight(-1) {
    for (int i = 0; i < maxImages; i++) {   //这是构造上述BufferItem的内容,只申请了4块空间.
        BufferItem* buffer = new BufferItem;
        mBuffers.push_back(buffer);
    }
}
```

相关路径:frameworks/base/media/java/android/media/ImageReader.java

```java
    /**
     * Attempts to acquire the next image from the underlying native implementation.
     *
     * <p>
     * Note that unexpected failures will throw at the JNI level.
     * </p>
     *
     * @param si A blank SurfaceImage.
     * @return One of the {@code ACQUIRE_*} codes that determine success or failure.
     *
     * @see #ACQUIRE_MAX_IMAGES
     * @see #ACQUIRE_NO_BUFS
     * @see #ACQUIRE_SUCCESS
     */
    private int acquireNextSurfaceImage(SurfaceImage si) {
        synchronized (mCloseLock) {
            // A null image will eventually be returned if ImageReader is already closed.
            int status = ACQUIRE_NO_BUFS;
            if (mIsReaderValid) {
                status = nativeImageSetup(si);//这边会对应到JNI android_media_ImageReader的对应方法
            }

            switch (status) {
                case ACQUIRE_SUCCESS:
                    si.mIsImageValid = true;
                case ACQUIRE_NO_BUFS:
                case ACQUIRE_MAX_IMAGES: //这是JNI 上报的最大值
                    break;
                default:
                    throw new AssertionError("Unknown nativeImageSetup return code " + status);
            }

            // Only keep track the successfully acquired image, as the native buffer is only mapped
            // for such case.
            if (status == ACQUIRE_SUCCESS) {
                mAcquiredImages.add(si);
            }
            return status;
        }
    }
```



![JNI.png](https://s2.loli.net/2022/05/16/EXj5iBuNPFo3cy4.png)



相关log 关键字

```
CameraService: CameraService::connect call
Camera2ClientBase: Camera 0: Opened
CameraDeviceClient: CameraDeviceClient
open_: open camera
hal3: Constructor
cap_intent=2
ImageReader_JNI
```

这是三方打开相机JNI的相关log.这里我们能清楚看到maxImages的值为4. 这也解释通这个三方apk只能拍4张照片的原因.一旦超过4张照片就不再申请新的buffer去拍照存第5张图片.而我们正常apk  有的设置拍照8张后会立马清空缓冲，然后重新拍照,所以不存在这个问题.也有平台专门对JNI进行更好的兼容,所以也不存在这个问题.但是目前看来这个apk 在mtk 和高通不同的项目上也存在拍照4张后无法拍第5张的情况.说明这个apk 确定很拉胯.而且平板项目这个apk没有拍照的功能.

![2022-05-16 14-21-45 的屏幕截图.png](https://s2.loli.net/2022/05/16/NLPwStgWZJ1hRi5.png)



# 4 修改方案

在acquireNextSurfaceImage拿buffer 满的时候.给它逐一释放关闭.image.close();

```diff

---

diff --git a/media/java/android/media/ImageReader.java b/media/java/android/media/ImageReader.java
index 87c3bb9..8b7f8d0 100644
--- a/media/java/android/media/ImageReader.java
+++ b/media/java/android/media/ImageReader.java
@@ -36,6 +36,7 @@
 import java.util.List;
 import java.util.concurrent.CopyOnWriteArrayList;
 import java.util.concurrent.atomic.AtomicBoolean;
+import android.util.Log;
 
 /**
  * <p>The ImageReader class allows direct application access to image data
@@ -58,7 +59,7 @@
  * production rate.</p>
  */
 public class ImageReader implements AutoCloseable {
-
+    private final static String TAG = "qtf";
     /**
      * Returned by nativeImageSetup when acquiring the image was successful.
      */
@@ -469,6 +470,10 @@
                     si.mIsImageValid = true;
                 case ACQUIRE_NO_BUFS:
                 case ACQUIRE_MAX_IMAGES:
+                    for(Image image : mAcquiredImages){  //sprd_qtf
+                        image.close();
+                        }
+                    Log.i(TAG, "imageBuffer has been clear!");
                     break;
                 default:
                     throw new AssertionError("Unknown nativeImageSetup return code " + status);
```

