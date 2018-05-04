---
layout:     post
title:      "Unity与Android的互相调用及异步调用的实现"
description: "通过Android消息机制实现异步调用"
date:       2016-07-01 15:21:00
author:     "安地"
header-img: "img/post-bg-2016.jpg"
tags:
    - Unity
    - Android
---

## 前言

我们unity程序有很多依赖android的地方，以为很简单，后来发现坑好多。unity只有在主线程才能调android的方法，在unity中调android的方法启动线程都不能执行，在unity的子线程无法获取AndroidJavaObject，这样耗时方法调用就会有问题了。

## Unity与Android互相调用方法

Unity调Android，使用AndroidJavaClass和AndroidJavaObject就可以获取到java类和对象了，下面这个方法是获取默认UnityPlayerActivity对象的方法：

``` C#
        AndroidJavaClass jc = new AndroidJavaClass ("com.unity3d.player.UnityPlayer");
        jo = jc.GetStatic<AndroidJavaObject> ("currentActivity");
        string result = jo.Call<string>("getVersion");
```
Android调unity也很简单：

``` java
        UnityPlayer.UnitySendMessage("objectName", "functionName","value");
```
Unity中需要有name为“objectName”的对象，其绑定的脚本类需要这样的方法：

``` C#
        void functionName(string str)
```
这样就可以接受到android发来的消息了，但这个消息参数是String,复杂对象无法直接传递。

## 异步调用的实现
    Unity需要调用Android的耗时方法怎么办呢，我想到可以先Unity调用Android方法，在Android调Unity的方法的形式来达到回调的目的。
Android部分：
    
定义一个消息格式：
``` java
        class UnityCallMessage {
        String contentId;
        String name;

        public UnityCallMessage(String name, String contentId) {
            this.name = name;
            this.contentId = contentId;
        }
        }
```
新建一个handler，用来接收消息：
``` java
         mHandler = new Handler() {
            public void handleMessage(Message msg) {
                if (msg.what == GET_CONTENG && msg.obj instanceof UnityCallMessage) {
                    getVrContentReturn((UnityCallMessage) msg.obj);
                }
                super.handleMessage(msg);
            }
        };
```
被unity调用的方法，发送一个消息，直接结束：
   
        public void getVrContentAsync(String name, String contentId) {
        Log.d("myth", "getVrContentAsync start" + contentId);
        Message msg = new Message();
        msg.what = GET_CONTENG;
        msg.obj = new UnityCallMessage(name, contentId);
        this.mHandler.sendMessage(msg);
        }
        
调unity的方法，这个方法被handler调用，完成后调unity：
``` java
       public void getLocalResReturn(UnityCallMessage message) {
        ContentsCoverData[] data = getData();
        if (data != null) {
            DataTransport.getInstance().put(KEY_GET_LOCAL_RES, data);
            UnityPlayer.UnitySendMessage(message.name, "getLocalResReturn", KEY_GET_LOCAL_RES);
            Log.d("myth", "getLocalResReturn");
            return;
        }
        UnityPlayer.UnitySendMessage(message.name, "getLocalResReturn", "failed");
        Log.d("myth", "failed");
    }
```
其中有个DataTransport类，用于在内存中存储对象的，之前文章有讲过，Unity中可以根据对应key去取到这个对象，对应方法：
``` java
        public Object getSavedObject(String key) {
        Object data = DataTransport.getInstance().get(key);
        Log.d("myth", "data:" + data);
        return data;
        }
```
    
Unity中需要对应方法为：
``` C#
    public void getLocalResReturn(string key)
	{
	AndroidJavaClass jc = new AndroidJavaClass ("com.unity3d.player.UnityPlayer");
	jo = jc.GetStatic<AndroidJavaObject> ("currentActivity");
	AndroidJavaObject[] array = jo.Call<AndroidJavaObject[]> ("getSavedObject", key);
	//do some thing using the datas
	}
```

## 感受

Android和unity互调是比较简单的，但也有很多限制，我们应用因为有奇怪地需求就用了Android消息机制来实现异步调用，但不建议使用。