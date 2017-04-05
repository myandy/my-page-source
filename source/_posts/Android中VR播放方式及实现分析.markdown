---
title:      "Android中VR播放方式及实现分析"
description:   "在一个Android应用中实现了Unity播放和Android播放"
date:       2016-10-13 12:00:00
author:     "安地"
tags:
        - Android
        - Unity
    
---

## 实现方式分析

手机VR核心就是分屏显示，对左右眼显示不同的画面，通过眼镜展示出VR效果。这一块的功能在Android中通过OpenGL就可以做，Google，Oculus还有国内的暴风魔镜都推出了VR SDK，内容开发者在SDK上开发就可以了。手机VR上Oculus的OVR只能在支持GearVR的手机上使用，其它两家的SDK都可以在普通手机上使用，但效果确实有点差。SDK一般都有Android版，iOS版和Unity版，暴风魔镜还有unreal版。

我们之前有做过使用OVR的Unity版，我们现在的需求是在普通Android手机里面上播放VR视频,就决定使用GVR了，暴风魔镜SDK太low了不考虑。再决定用什么平台的SDK，使用Android SDK的话进入播放更快，接入简单，还可以半屏播放，使用Unity SDK的话是在场景中播放，播放普通3D视频更有沉浸感，对于全景视频的话是一样的，还有可以做播放之外的场景内容。

我们之前做的OVR版的场景可以移植到GVR中，甚至可以共用一套场景代码，加之Unity本身可以做的更多东西，最后决定使用Unity版。我们的方式是把Unity应用导出Android代码，合并到Android工程中，一个小缺点就是进入3D速度比较慢，要进入unity中看3D。暴风魔镜也是使用的这种方式，3D播播，橙子VR的话使用的Android直接播放的方式。

## 遇到的问题

### Unity和Android合并
Unity导出Android代码后，manifest中是GoogleUnityActivity,我们需要在Activity上做一些工作，就继承了它，把manifest改称我们继承的HomeActivity，就可以在里面做很多事情，比如加统计，写给Unity调用的方法等。


### Unity返回崩溃的问题
开始Unity返回时没有效果，就在Unity端调用了GoogleUnityActivity的finish方法，但就提示意外关闭，几番尝试改为调用杀死本进程并把Unity的Activity放入远程进程中终于可以了。就是manifest的Activity加上 android:process=":remote"，在unity退出时调下面这个方法。

    public void quit() {
        android.os.Process.killProcess(android.os.Process.myPid());
        System.exit(0);

    }
  
### 混淆的问题

官方没有说明，很多地方不能混淆的，还有Android和Unity互相调用的部分也要防混淆，都加上就可以了，不注意这些bug就真难找。

### Unity SDK代码和Android SDK 代码冲突的问题
前面说我们最终使用Unity播放，但Android SDK中我也尝试了下，非常简单集成就可以播放VR视频了，还可以半屏和全屏播放。结果老大让把这个也加上，就是同时可以使用GVR的Android版和Unity播放。我们把用Android SDK播放叫预览，主要是用于半屏播放。一个大坑挖好了。


把两种播放方式都加进来后发现lib冲突，因为两边的SDK有部分代码是一样的，然后就慢慢分析对比了，结果发现两边都有一个lib叫"com.google.vr.cardboard"，一大一小，还用哪个都不行。我发现是因为Unity中用的CardBoard SDK，和Unity开发沟通，说一大推，就是更换成本太高，不想换，WTF。
只能继续想办法了，对两边类进行分析，最后把反编译的代码直接拿出来，直接把相关类放入工程中，各种错误，慢慢fix，最后总算能用了。

解决这个问题用去一整天，这样的事情其实不应该出现的，前面改需求，后面版本不统一，最后勉强解决问题，但对以后更新和维护就是一个大坑，但我所能做的就这么多了。


