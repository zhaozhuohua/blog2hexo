---
title: android人脸检测点位置转换
date: 2024-12-26 17:20:00
tags: android, 相机
comments: true
---

<meta name="referrer" content="no-referrer"/>

>android相机开发很多需要进行人脸检测，很多公司也推出了自己的人脸识别服务。比如：阿里、腾讯、虹软、Face++等等。但这些有缺点，就是必须集成第三方库联网激活，也增大了app。如果只需要检测人脸这种轻量级功能，用第三方库就有些重，幸好android自带了人脸检测功能。下来我们就主要看看Android如何实现人脸检测。

先上效果图：
![人脸检测.gif](https://upload-images.jianshu.io/upload_images/6471979-92cc8d7b2903b744.gif?imageMogr2/auto-orient/strip)

Android 实现人脸检主要调用一下接口和方法

```
mCamera!!.setFaceDetectionListener(object : Camera.FaceDetectionListener {
        override fun onFaceDetection(p0: Array<out Camera.Face>?, p1: Camera?) {
            var size = 0
            if (p0 != null) {
                size = p0.size
            }
            Log.i("face", "size: $size")
        }
    })
//开始人脸检测
mCamera!!.startFaceDetection()
```

以上代码就可以获取到人脸数量。

但是到这里不满足的开发者就会觉得，我还想往上面现实人脸框呢，总不能拿界面上不显示任何信息吧。
那下来咱们看看人脸框是如何现实的

首先明确一点，Camera.Face 这里面对应的坐标点和当前界面的坐标点不是一个意思，看看Android 中对Face类中Rect的介绍吧

>#rect
>Added in [API level 14](https://developer.android.google.cn/guide/topics/manifest/uses-sdk-element.html#ApiLevels)
Deprecated in [API level 21](https://developer.android.google.cn/guide/topics/manifest/uses-sdk-element.html#ApiLevels)
Bounds of the face. (-1000, -1000) represents the top-left of the camera field of view, and (1000, 1000) represents the bottom-right of the field of view. For example, suppose the size of the viewfinder UI is 800x480\. The rect passed from the driver is (-1000, -1000, 0, 0). The corresponding viewfinder rect should be (0, 0, 400, 240). It is guaranteed left < right and top < bottom. The coordinates can be smaller than -1000 or bigger than 1000\. But at least one vertex will be within (-1000, -1000) and (1000, 1000).
>
>The direction is relative to the sensor orientation, that is, what the sensor sees. The direction is not affected by the rotation or mirroring of [Camera.setDisplayOrientation(int)](https://developer.android.google.cn/reference/android/hardware/Camera.html#setDisplayOrientation(int)). The face bounding rectangle does not provide any information about face orientation.

原文地址 [Camera.Face](https://developer.android.google.cn/reference/android/hardware/Camera.Face.html)

英文不好理解的话咱们直接上图
![Camera.Face 中坐标.png](https://upload-images.jianshu.io/upload_images/6471979-8347d50ece1ed520.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如图，正常view (0,0) 坐标是在左上角，而相机返回的人脸坐标(0,0)实在界面中央，所以拿到的Face信息咱们要进行转换后才能使用！
转换方法如下：
```
     /**
     * 准备用于转换的矩阵工具
     *
     * @param isBackCamera  是否后置相机
     * @param displayOrientation  摄像头设置的角度
     * @param viewWidth  预览界面宽
     * @param viewHeight  预览界面高
     */
    fun prepareMatrix(isBackCamera: Boolean, displayOrientation: Int,
                      viewWidth: Int, viewHeight: Int): Matrix {
        val matrix = Matrix()
        //前置摄像头处理镜像关系
        matrix.setScale(1f, (if (!isBackCamera) -1 else 1).toFloat())
        matrix.postRotate(displayOrientation.toFloat())
        matrix.postScale(viewWidth / 2000f, viewHeight / 2000f)
        matrix.postTranslate(viewWidth / 2f, viewHeight / 2f)

        return matrix
    }
```

这也是android官方这转换方法，详见[Camera.Face](https://developer.android.google.cn/reference/android/hardware/Camera.Face.html)

咦？还没看懂。人脸识别更详细的看这里 [Android人脸检测功能和检测特效](https://www.jianshu.com/p/3a61cdaa3f58)