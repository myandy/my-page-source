---
title:      Android页面启动速度优化
description: 如何最快地展示出你的界面
date:       2017-6-18 12:00:00
author:     "安地"
tags:
          - Android
---

## 优化方法

### Application优化

页面启动分冷启动和热启动，冷启动需要先加载Application，所以优化Application的加载时间是十分必要的。
主线程尽量减少耗时操作，把不需要同步加载的项放到异步线程中初始化，必须同步加载的项针对做加载时间优化。
应用有多个进程的情况，判断进程名称做对应初始化。

判断进程名称方法：

    public static String getCurrentProcessName(Context context) {
            String processName = null;
            ActivityManager actMgr = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
            List<ActivityManager.RunningAppProcessInfo> appList = actMgr.getRunningAppProcesses();
            if (null != appList) {
                for (ActivityManager.RunningAppProcessInfo info : appList) {
                    if (info.pid == android.os.Process.myPid()) {
                        processName = info.processName;
                        break;
                    }
                }
            }
            if (processName == null) {
                processName = context.getApplicationInfo().name;
            }
            return processName;
        }

### Activity优化

1.最基本的是布局优化，简化布局层级，优化过度绘制。
2.BaseActivity的优化，BaseActivity很多东西可能在当前activity中都用不到，计算消耗时间作对应优化。
3.提前启动耗时加载项，在super.onCreate之前启动耗时加载线程。
4.在界面加载完毕后再做额外加载项：

     getWindow().getDecorView().post(new Runnable() {
                @Override
                public void run() {
                }
            });

如果有fragment的界面，优化目标是界面完全显示的话就要在onCreate时加加载，优化目标是activity开始显示的话可以只加载一个loading view。

### Fragment优化

和activity挺类似。
1.布局的优化。
2.基本类的优化。
3.提前启动耗时加载项，在super.onCreate之前启动耗时加载线程。
4.加载view要在onCreateView中加载，不要放一部分到onViewCreate(),及时有顺序依赖也都在onCreateView中加载，注意好顺序就行。

### 图片加载优化

图片加载优化关键是缓存，然后是注意加载大小。

### 网络加载优化

网络加载想要最快的话可以先使用缓存数据再去进行网络请求刷新界面，看业务需求使用不同的加载和缓存策略。

### 感知优化

这种不是实际的优化，但却能让用户感觉到更快。
一个最常用的是启动白屏或者黑屏，可以在manifest中把启动activity改成透明主题，在activity的onCreate之前改为自己的主题。
还有一种是activity跳转动画，这种会让点击后界面跳转变慢，但两边界面元素的联动会让跳转很自然，让用户感觉更快一些。
感知优化看情况使用，效果还是可以的。

## 总结

仔细分析每一个部分的加载时间，针对性优化，可以用trackview看执行时间。