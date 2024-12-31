---
title: Android人脸检测功能和检测特效
date: 2024-12-31 15:45:29
tags: 相机
cover: https://s2.loli.net/2024/12/31/puG79ZqfvHj64sz.jpg
---

<meta name="referrer" content="no-referrer"/>

>这段时间再做安防相关的硬件设备定制，涉及到了小区的业主和流动人员人员登记、闸机管理和小区内部摄像头等等。
这里面用到了人脸检测、识别，人脸检测用了虹软的算法，虹软的文档还是比较全的这里不多做介绍。这里主要说下android自带的人脸检测和人脸框的绘制。

先来看看人脸识别框特效：
![人脸检测特效.gif](https://upload-images.jianshu.io/upload_images/6471979-4fbc626d4d2447be.gif?imageMogr2/auto-orient/strip)

*这个例子使用Camera API实现的并不是Camera2 API，原因是发现2019年了还有的设备居然不支持Camera2的人脸识别，为了通用就用了Camera API。*
以下为主要代码
上代码：
1、下来是处理人脸检测回调方法
```
                    //进行相机初始化
                    cameraHelper = CameraControllerHelper.Builder()
                            .previewViewSize(Point(camera_surfaceview.measuredWidth, camera_surfaceview.measuredHeight))
                            .rotation(mBaseAc.windowManager.defaultDisplay.rotation)
                            .specificCameraId(cameraId)
                            .isMirror(isMirror())
                            .previewOn(camera_surfaceview)
                            .cameraListener(this@CameraFaceFm)
                            //设置人脸检测回调方法
                            .setFaceDetectionListener(object : Camera.FaceDetectionListener {
                                override fun onFaceDetection(p0: Array<out Camera.Face>?, p1: Camera?) {
                                    if (faceRectView != null) {
                                        faceRectView.clearFaceInfo()
                                        faceRectView.addFaceInfo(p0!!.toList())
                                    }
                                }
                            })
                            .build()
                    val p = getScreenPoint()
                    DrawFaceHelper.cameraWidth = p.x.toFloat()
                    DrawFaceHelper.cameraHeight = p.y.toFloat()
                    DrawFaceHelper.isBackCameraId = isBackCameraId()
                    cameraHelper.init()
                    cameraHelper.start()
```
2、相机的启动、参数设置和人脸检测
```
if (mCamera != null) {
                return
            }
            //若指定了相机ID且该相机存在，则打开指定的相机
            if (specificCameraId != null) {
                mCameraId = specificCameraId!!
            } else {
                //相机数量为2则打开1,1则打开0,相机ID 1为前置，0为后置
                mCameraId = Camera.getNumberOfCameras() - 1
            }

            //没有相机
            if (mCameraId == -1) {
                cameraListener?.onCameraError(Exception("camera not found"))
                return
            }
            if (mCamera == null) {
                mCamera = Camera.open(mCameraId)
            }
            displayOrientation = getCameraOri(rotation)
            mCamera!!.setDisplayOrientation(displayOrientation)
            try {
                val parameters = mCamera!!.parameters
                parameters.previewFormat = ImageFormat.NV21

                //预览大小设置
                previewSize = parameters.previewSize
                val supportedPreviewSizes = parameters.supportedPreviewSizes
                if (supportedPreviewSizes != null && supportedPreviewSizes.size > 0) {
                    previewSize = currentPreviewSize
                }

                parameters.setPreviewSize(previewSize!!.width, previewSize!!.height)

                pictureSize = parameters.pictureSize
                val supportedPicviewSizes = parameters.supportedPreviewSizes
                if (supportedPicviewSizes != null && supportedPicviewSizes.size > 0) {
                    pictureSize = currentPictureSize
                }
                parameters.setPictureSize(pictureSize!!.width, pictureSize!!.height)


                //对焦模式设置
                val supportedFocusModes = parameters.supportedFocusModes
                if (supportedFocusModes != null && supportedFocusModes.size > 0) {
                    if (supportedFocusModes.contains(Camera.Parameters.FOCUS_MODE_CONTINUOUS_PICTURE)) {
                        parameters.focusMode = Camera.Parameters.FOCUS_MODE_CONTINUOUS_PICTURE
                    } else if (supportedFocusModes.contains(Camera.Parameters.FOCUS_MODE_CONTINUOUS_VIDEO)) {
                        parameters.focusMode = Camera.Parameters.FOCUS_MODE_CONTINUOUS_VIDEO
                    } else if (supportedFocusModes.contains(Camera.Parameters.FOCUS_MODE_AUTO)) {
                        parameters.focusMode = Camera.Parameters.FOCUS_MODE_AUTO
                    }
                }
                setCameraParameters(parameters)
                mCamera!!.setPreviewTexture(previewDisplayView!!.getSurfaceTexture())
                mCamera!!.setPreviewCallback(this)
                mCamera!!.startPreview()

                //注意这段，这里是设置人脸检测监听和启动
                if (mFaceDetectionListener != null) {
                    mCamera!!.setFaceDetectionListener(mFaceDetectionListener)
                    mCamera!!.startFaceDetection()
                }
                if (autoFocusCallback != null) {
                    mCamera!!.autoFocus(autoFocusCallback)
                }
                cameraListener?.onCameraOpened(mCamera!!, mCameraId, displayOrientation, isMirror)
            } catch (e: Exception) {
                if (cameraListener != null) {
                    cameraListener.onCameraError(e)
                } else {
                    e.printStackTrace()
                }
            }

        }
```

启动相机这块就不多做介绍，以为网上现成的文档还是很多的。

##下来才是重头戏！！！

3、在FaceRectView的onDraw方法内进行人脸框显示
```
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        if (faceRectList != null && faceRectList.isNotEmpty()) {
            DrawFaceHelper.drawFaceRect(canvas, faceRectList[0].rect, color, 3)

            //不断进行刷新，实现人脸框动画更新
            postInvalidate()
        } else {
            DrawFaceHelper.resetTimerInfo()
        }
    }
```

**这里需要说明下，相机返回的Face信息里面的Rect的坐标并不是view的真实坐标，这个需要转换。转换方式见 [android人脸检测点位置转换](https://www.jianshu.com/p/b13a82935351)**

4、用DrawFaceHelper中的drawFaceRect绘制人脸框
```
/**
     * 绘制数据信息到view上，若 [DrawInfo.getName] 不为null则绘制 [DrawInfo.getName]
     *
     * @param canvas            需要被绘制的view的canvas
     * @param drawInfo          绘制信息
     * @param color             rect的颜色
     * @param faceRectThickness 人脸框线条粗细
     */
    fun drawFaceRect(canvas: Canvas?, drawInfo: Rect?, color: Int, faceRectThickness: Int) {

        var displayOrientation: Int = if (isBackCameraId) { 90 } else { 270 }

        //将返回的人脸信息，转为view可识别的信息
        val matrix = CameraUtils.prepareMatrix(isBackCameraId, displayOrientation, cameraWidth.toInt(), cameraHeight.toInt())
        val rectF = RectF(drawInfo)
        matrix.mapRect(rectF)

        if (canvas == null || drawInfo == null) {
            return
        }
        if (timer == null) {
            initTimerInfo()
        }
        val paint = Paint()
        paint.style = Paint.Style.STROKE  //设置空心
        paint.strokeWidth = faceRectThickness.toFloat()

        val spic = 100
        val rect = Rect(rectF.left.toInt() - spic, rectF.top.toInt() - spic, rectF.right.toInt() + spic, rectF.bottom.toInt() + spic)

        val width = rect.right - rect.left
        val height = rect.bottom - rect.top
        //比较小的设置为半径
        val radius = if (height > width) { width / 2 } else { height / 2 }
        val cx = (rect.left + rect.right) / 2
        val cy = (rect.top + rect.bottom) / 2

        //绘制遮罩层
        val maskPaint = Paint()
        //整个相机预览界面进行遮罩，透明度用来设置遮罩半透明
        canvas.saveLayerAlpha(0f, 0f, cameraWidth, cameraHeight, maskAlpha, Canvas.ALL_SAVE_FLAG)
        maskPaint.color = Color.BLACK
        //整个相机预览界面进行遮罩
        canvas.drawRect(Rect(0, 0, cameraWidth.toInt(), cameraHeight.toInt()), maskPaint)
        //重叠区域进行处理
        //只在源图像和目标图像不相交的地方绘制【源图像】，相交的地方根据目标图像的对应地方的alpha进行过滤，
        //目标图像完全不透明则完全过滤，完全透明则不过滤
        maskPaint.xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_OUT)
        maskPaint.color = Color.BLACK
        canvas.drawCircle(cx.toFloat(), cy.toFloat(), radius.toFloat(), maskPaint)

        //自定义人脸框样式
        paint.flags = Paint.ANTI_ALIAS_FLAG  //抗锯齿
        paint.color = color

        //设置正方形，以便下面设置弧形为圆的弧形
        rect.left = cx - radius
        rect.top = cy - radius
        rect.right = cx + radius
        rect.bottom = cy + radius

        val rectF0 = getRectF(faceOffset0, rect)
        paint.strokeWidth = 4f
        paint.alpha = faceAlpha0
        canvas.drawArc(rectF0, getStartAngle(faceStartAngle0, false), faceSweepAngle0, useCenter, paint)

        val rectF1 = getRectF(faceOffset1, rect)
        paint.strokeWidth = 8f
        paint.alpha = faceAlpha1
        canvas.drawArc(rectF1, getStartAngle(faceStartAngle1, true), faceSweepAngle1, useCenter, paint)

        val rectF2 = getRectF(faceOffset2, rect)
        paint.strokeWidth = 3f
        paint.alpha = faceAlpha2
        canvas.drawArc(rectF2, getStartAngle(faceStartAngle2, false), faceSweepAngle2, useCenter, paint)

        paint.alpha = faceAlpha3
        val rectF3 = getRectF(faceOffset3, rect)
        paint.strokeWidth = 9f
        canvas.drawArc(rectF3, getStartAngle(faceStartAngle3, true), faceSweepAngle3, useCenter, paint)

        val rectF4 = getRectF(faceOffset3, rect)
        paint.strokeWidth = 9f
        canvas.drawArc(rectF4, getStartAngle(faceStartAngle4, true), faceSweepAngle4, useCenter, paint)
    }

    /**
     * 设置圆环偏移量
     * @param startAngle 角度
     * @param clockwise  是否顺时针
     */
    private fun getStartAngle(startAngle:Float, clockwise: Boolean):Float {
        val angle = if (clockwise) { startAngle + timeIndex } else { startAngle - timeIndex}

        return angle % 360
    }

    private fun getRectF(offset: Int, rect: Rect): RectF {
        val rectF = RectF(rect)
        rectF.left = rectF.left - offset
        rectF.top = rectF.top - offset
        rectF.right = rectF.right + offset
        rectF.bottom = rectF.bottom + offset

        return rectF
    }
```

这就是实现的大致流程。

代码里面只支持了显示一张人脸的检测方式。如有需要显示多张人脸的话请各位童鞋开动脑力自己解决！
完整代码请戳这里：[FaceCamera](https://github.com/imqingyue/FaceCamera)
