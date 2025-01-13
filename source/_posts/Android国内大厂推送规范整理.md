---
title: Android国内大厂推送规范整理
date: 2025-01-13 14:48:02
tags: 推送
categories: android推送
---

<meta name="referrer" content="no-referrer"/>

当前很多APP用的是极光推送，现在Android系统对应用管理比前几年规范的多。因此造成了很多比较重要的通知并不能很及时的推送给用户，导致这个问题的原因就是APP很可能在后台被杀死了。所以要让APP适配下国内各大厂商的推送服务。

下来梳理下几个大厂商推送信息：
|厂商|推送方式|透传|支持自定义铃声|支持的设备|文档地址|
|-|-|-|-|-|-|
|小米|标签（Topic）、RegID、别名（Alias）、Useraccount四种消息发送方式|支持|支持|支持Android2.2以上和IOS系统推送|[文档中心](https://dev.mi.com/console/doc/detail?pId=230)|
|华为|支持主题、Token、特定的受众群组|支持|支持|1、华为手机、华为平板EMUI 3.1及以上。<br/>2、非华为手机和平板。Android 5.1及以上。<br/>3、沃尔沃和小康车机，Android 9.0及以上。<br/>4、iPhone，iOS 10.0及以上|[推送服务](https://developer.huawei.com/consumer/cn/doc/development/HMSCore-Guides-V5/service-introduction-0000001050040060-V5)|
|OPPO|推送服务支持标签、RegID、Alias等推送方式|不支持|不支持|支持 ColorOS3.1及以上的系统的OPPO的机型，一加5/5t及以上机型，realme所有机型。|[OPPO开放平台](https://open.oppomobile.com/wiki/doc#id=10742)|
|VIVO|支持标签、RegID、Alias等消息发送方式。|不支持|不支持|vivo和iqoo手机并且需要通过`PushClient.getInstance(context).isSupport();（ture :系统支持push、false 系统不支持push）`方法准确获知当前系统是否支持push。|[vivo开放平台](https://dev.vivo.com.cn/documentCenter/doc/541)|
|魅族|PushId 推送、别名推送、标签推送|支持|不支持|魅族手机、flyme系统|[推送文档](http://open-wiki.flyme.cn/doc-wiki/index#id?129)|

### 开发过程中遇到的问题
1、android 8及以上的系统推送通知都分了系统通道和其它通道，不重要等几个通道默认是没有提醒的，如果需要提醒就要手动在应用设置界面打开。
|![image.png](https://upload-images.jianshu.io/upload_images/6471979-12886ab1af9324fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)|![image.png](https://upload-images.jianshu.io/upload_images/6471979-f2a942a3ee756daa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)|
|-|-|
|||

2、android 8及以上的系统横幅通知默认是关闭的，需要手动打开（*微信、钉钉等大佬不用😂，估计国内手机厂商都不敢默认给人家不开启横幅通知*）
以下是华为官方回答：

![image.png](https://upload-images.jianshu.io/upload_images/6471979-bf8b553e82fc7f34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3、小米自定义铃声需要申请通知通道在申请的通知通道中设置（`注意我没发现申请后的通知通道能够编辑，因此需要在申请通道时正确的配置`）
![image.png](https://upload-images.jianshu.io/upload_images/6471979-392fcf3b9b61b0cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 注册和接收通知流程图
![流程图](https://upload-images.jianshu.io/upload_images/6471979-cba37b291f1c17fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)