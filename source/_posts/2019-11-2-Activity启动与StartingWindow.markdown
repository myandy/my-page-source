---
title:      "Activity启动与StartingWindow流程深入解析"
description:   "从Activity启动流程深入解析Starting Window"
date:       2019-11-2 14:00:00
author:     "安地"
tags:
      - Android
---


## 基础介绍

Starting Window就是一个用于在Activity创建并初始化成功前显示的临时窗口，拥有的Window Type是TYPE_APPLICATION_STARTING。
在startActivity，从而能够立即响应，当activity显示第一帧后会移除这个窗口。

设置windowDisablePreview属性可以控制Starting Window是否显示，默认是开启，Starting Window样式根据activity的主题生成。

```C
 <item name="android:windowDisablePreview">true</item>
```

Starting Window是每个Activity都可以设置的，默认的黑或者白其实就是startWindow，比如从Activity A启动Activity B，A执行onPause后界面就会进入Starting Window了，
此时B的onCreate可能都还没执行，直到B的第一帧显示出来后Starting Window才会消失。所以在onCreate改变主题没法影响到startWindow。


## Activity启动流程

我基于android-28源码分析下整个流程。

### 应用启动AMS

Activity启动流程，首先在应用进程调用startActivity方法，然后调用到Instrumentation，Instrumentation再调用到AMS,我们看下Instrumentation:execStartActivity方法：


```Java
   public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

ActivityManager.getService()获取到IActivityManager的binder引用，然后执行binder方法startActivity，checkStartActivityResult处理返回结果，启动失败都是在这处理的，常见如permission和class not found问题。

### AMS启动Activity

#### ActivityManagerService

ActivityManagerService的startActivity方法会调用startActivityAsUser，startActivityAsUser代码如下：

```java
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivity");

        userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

    }
```

#### ActivityStarter

ActivityManagerService通过控制器ActivityStartController设置参数最后执行ActivityStarter的execute方法：


```java
    int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            }
        } finally {
            onExecutionComplete();
        }
    }
```

execute调用startActivityMayWait或者startActivity，这两个方法最后都会调用到startActivityUnchecked。

startActivityUnchecked获取到Activity的信息ActivityRecord，ActivityRecord是activity历史栈中的一个节点，代表一个activity.
startActivityUnchecked再调用到ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法。


#### ActivityStack和ActivityStackSupervisor

ActivityStack是activity的栈，ActivityStackSupervisor是ActivityStack的监督者，核心调用转到了ActivityStackSupervisor实现，里面调用顺序如下：

ActivityStackSupervisor： resumeFocusedStackTopActivityLocked
ActivityStack: resumeTopActivityUncheckedLocked
ActivityStackSupervisor: resumeTopActivityInnerLocked
ActivityStackSupervisor: startSpecificActivityLocked
ActivityStackSupervisor: realStartActivityLocked


realStartActivityLocked就是真正启动activity的地方了，里面调用到了LaunchActivityItem，代码如下：

```java
              final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));
                 ...
                 // Schedule transaction.
                 mService.getLifecycleManager().scheduleTransaction(clientTransaction);
```

clientTransaction添加了一个LaunchActivityItem的事务，LaunchActivityItem留待后续再详解。

#### ClientTransaction事务传递

ClientLifecycleManager的scheduleTransaction方法：

```java
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            // If client is not an instance of Binder - it's a remote call and at this point it is
            // safe to recycle the object. All objects used for local calls will be recycled after
            // the transaction is executed on client in ActivityThread.
            transaction.recycle();
        }
    }
```

ClientTransaction的schedule方法：
```java
    public void schedule() throws RemoteException {
        this.mClient.scheduleTransaction(this);
    }
```

ApplicationThread的scheduleTransaction方法：
```java
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

ClientTransaction的scheduleTransaction方法,层层调用下来，调用到mClient的scheduleTransaction，mClient是IApplicationThread类型，IApplicationThread是一个控制接口，android.app.IApplicationThread.Stub继承Binder并实现IApplicationThread，
最后又被ActivityThread中的ApplicationThread实现。这是一个binder方法，调用到了应用进程，执行ApplicationThread的scheduleTransaction方法。


### 用户进程ActivityThread启动Activity

#### scheduleTransaction

ActivityThread的scheduleTransaction方法，实际在父类ClientTransactionHandler中，发送一个消息执行事务：

```java
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        this.sendMessage(159, transaction);
    }
```

在H（handler）中被执行，这里到了应用主线程：

```java
           case 159:
                ClientTransaction transaction = (ClientTransaction)msg.obj;
                ActivityThread.this.mTransactionExecutor.execute(transaction);
                if (ActivityThread.isSystem()) {
                    transaction.recycle();
                }
                break;
```

#### ClientTransaction事务
这里有个execute方法，事务执行器启动了事务。这个就是之前被添加的LaunchActivityItem，看其execute方法：


```java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```
我们知道ClientTransactionHandler的实现就是ActivityThread，这个事务处理已经在用户主线程了，看对应handleLaunchActivity方法。

#### activity创建

ActivityThread的handleLaunchActivity方法，进行窗口和配置的初始化，然后调用performLaunchActivity。

```java
    public Activity handleLaunchActivity(ActivityThread.ActivityClientRecord r, PendingTransactionActions pendingActions, Intent customIntent) {
        this.unscheduleGcIdler();
        this.mSomeActivitiesChanged = true;
        if (r.profilerInfo != null) {
            this.mProfiler.setProfiler(r.profilerInfo);
            this.mProfiler.startProfiling();
        }

        this.handleConfigurationChanged((Configuration)null, (CompatibilityInfo)null);
        if (!ThreadedRenderer.sRendererDisabled) {
            GraphicsEnvironment.earlyInitEGL();
        }

        WindowManagerGlobal.initialize();
        Activity a = this.performLaunchActivity(r, customIntent);
        if (a != null) {
            r.createdConfig = new Configuration(this.mConfiguration);
            this.reportSizeConfigurations(r);
            if (!r.activity.mFinished && pendingActions != null) {
                pendingActions.setOldState(r.state);
                pendingActions.setRestoreInstanceState(true);
                pendingActions.setCallOnPostCreate(true);
            }
        } else {
            try {
                ActivityManager.getService().finishActivity(r.token, 0, (Intent)null, 0);
            } catch (RemoteException var6) {
                throw var6.rethrowFromSystemServer();
            }
        }

        return a;
    }
```

performLaunchActivity获取到了classloader，然后调用mInstrumentation.newActivity创建activity：


```java
 private Activity performLaunchActivity(ActivityThread.ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = this.getPackageInfo((ApplicationInfo)aInfo.applicationInfo, r.compatInfo, 1);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(this.mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName, r.activityInfo.targetActivity);
        }

        ContextImpl appContext = this.createBaseContextForActivity(r);
        Activity activity = null;

        try {
            ClassLoader cl = appContext.getClassLoader();
            activity = this.mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception var14) {
            if (!this.mInstrumentation.onException(activity, var14)) {
                throw new RuntimeException("Unable to instantiate activity " + component + ": " + var14.toString(), var14);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, this.mInstrumentation);
            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(this.mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }

                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }

                appContext.setOuterContext(activity);
                activity.attach(appContext, this, this.getInstrumentation(), r.token, r.ident, app, r.intent, r.activityInfo, title, r.parent, r.embeddedID, r.lastNonConfigurationInstances, config, r.referrer, r.voiceInteractor, window, r.configCallback);
                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }

                r.lastNonConfigurationInstances = null;
                this.checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    this.mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    this.mInstrumentation.callActivityOnCreate(activity, r.state);
                }

                if (!activity.mCalled) {
                    throw new SuperNotCalledException("Activity " + r.intent.getComponent().toShortString() + " did not call through to super.onCreate()");
                }

                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }

                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            this.mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state, r.persistentState);
                        }
                    } else if (r.state != null) {
                        this.mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }

                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        this.mInstrumentation.callActivityOnPostCreate(activity, r.state, r.persistentState);
                    } else {
                        this.mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }

                    if (!activity.mCalled) {
                        throw new SuperNotCalledException("Activity " + r.intent.getComponent().toShortString() + " did not call through to super.onPostCreate()");
                    }
                }
            }

            r.paused = true;
            this.mActivities.put(r.token, r);
        } catch (SuperNotCalledException var12) {
            throw var12;
        } catch (Exception var13) {
            if (!this.mInstrumentation.onException(activity, var13)) {
                throw new RuntimeException("Unable to start activity " + component + ": " + var13.toString(), var13);
            }
        }

        return activity;
    }
```

Instrumentation创建activity：
```java
    public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        return (Activity)cl.loadClass(className).newInstance();
    }
```



### 总结


应用启动activity
1.Activity:startActivityForResult
2.Instrumentation:execStartActivity
3.ActivityManagerService:startActivity

AMS启动Activity
1.ActivityManagerService:startActivity AMS检查用户信息调用到Activity启动器
2.ActivityStarter:startActivityUnchecked Activity启动器获取到activity栈等信息，设置一些状态
3.ActivityStackSupervisor：realStartActivityLocked添加使用LaunchActivityItem事务并使用IApplicationThread安排进计划

ActivityThread创建Activity
1.ActivityThread:scheduleTransaction安排事务，发送一个消息给handler
2.ActivityThread在Handler主线程中执行事务，即LaunchActivityItem的execute方法
3.ActivityThread：handleLaunchActivity到performLaunchActivity获取classloader，创建window
4.Instrumentation的newActivity通过classloader加载activity类

## Starting Window 流程

### Starting Window显示

#### 调用入口
Starting Window是启动activity立即显示的，所以在AMS启动activity中被创建的，果然在ActivityStarter找到了对应方法。

ActivityStarter调用startActivityUnchecked时会调用setTargetStackAndMoveToFrontIfNeeded：

 ```java
    /**
     * Figure out which task and activity to bring to front when we have found an existing matching
     * activity record in history. May also clear the task if needed.
     * @param intentActivity Existing matching activity.
     * @return {@link ActivityRecord} brought to front.
     */
    private ActivityRecord setTargetStackAndMoveToFrontIfNeeded(ActivityRecord intentActivity) {
    
          ...
          intentActivity.showStartingWindow(null /* prev */, false /* newTask */,
                            true /* taskSwitch */);
                            
    }
```

这个方法是已经有activity存在时恢复activity调用的，恢复前会调用showStartingWindow显示StartingWindow。

又在resumeTopActivityInnerLocked和moveTaskToFrontLocked等处找到了showStartingWindow的调用。
resumeTopActivityInnerLocked是前面正常启动activity的步骤，moveTaskToFrontLocked应该是把activity移到前台的调用，
都是activity回到前台时需要，符合我们的理解。

ActivityRecord的showStartingWindow方法：
              
```java
      void showStartingWindow(ActivityRecord prev, boolean newTask, boolean taskSwitch,
              boolean fromRecents) {
          if (mWindowContainerController == null) {
              return;
          }
          if (mTaskOverlay) {
              // We don't show starting window for overlay activities.
              return;
          }
  
          final CompatibilityInfo compatInfo =
                  service.compatibilityInfoForPackageLocked(info.applicationInfo);
          final boolean shown = mWindowContainerController.addStartingWindow(packageName, theme,
                  compatInfo, nonLocalizedLabel, labelRes, icon, logo, windowFlags,
                  prev != null ? prev.appToken : null, newTask, taskSwitch, isProcessRunning(),
                  allowTaskSnapshot(),
                  mState.ordinal() >= RESUMED.ordinal() && mState.ordinal() <= STOPPED.ordinal(),
                  fromRecents);
          if (shown) {
              mStartingWindowState = STARTING_WINDOW_SHOWN;
          }
      }
```
                            
                            
#### 判断主题是否添加                            
mWindowContainerController.addStartingWindow，
这里获取了主题样式，判断是否要显示starting window，如果是透明或者是窗口模式等等就不会显示，之前讲的windowDisablePreview熟悉也是在这里获取，不可用就直接false，最后调用scheduleAddStartingWindow。代码如下：


```java
   public boolean addStartingWindow(String pkg, int theme, CompatibilityInfo compatInfo,
            CharSequence nonLocalizedLabel, int labelRes, int icon, int logo, int windowFlags,
            IBinder transferFrom, boolean newTask, boolean taskSwitch, boolean processRunning,
            boolean allowTaskSnapshot, boolean activityCreated, boolean fromRecents) {
        synchronized(mWindowMap) {
            if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "setAppStartingWindow: token=" + mToken
                    + " pkg=" + pkg + " transferFrom=" + transferFrom + " newTask=" + newTask
                    + " taskSwitch=" + taskSwitch + " processRunning=" + processRunning
                    + " allowTaskSnapshot=" + allowTaskSnapshot);

            if (mContainer == null) {
                Slog.w(TAG_WM, "Attempted to set icon of non-existing app token: " + mToken);
                return false;
            }

            // If the display is frozen, we won't do anything until the actual window is
            // displayed so there is no reason to put in the starting window.
            if (!mContainer.okToDisplay()) {
                return false;
            }

            if (mContainer.startingData != null) {
                return false;
            }

            final WindowState mainWin = mContainer.findMainWindow();
            if (mainWin != null && mainWin.mWinAnimator.getShown()) {
                // App already has a visible window...why would you want a starting window?
                return false;
            }

            final TaskSnapshot snapshot = mService.mTaskSnapshotController.getSnapshot(
                    mContainer.getTask().mTaskId, mContainer.getTask().mUserId,
                    false /* restoreFromDisk */, false /* reducedResolution */);
            final int type = getStartingWindowType(newTask, taskSwitch, processRunning,
                    allowTaskSnapshot, activityCreated, fromRecents, snapshot);

            if (type == STARTING_WINDOW_TYPE_SNAPSHOT) {
                return createSnapshot(snapshot);
            }

            // If this is a translucent window, then don't show a starting window -- the current
            // effect (a full-screen opaque starting window that fades away to the real contents
            // when it is ready) does not work for this.
            if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "Checking theme of starting window: 0x"
                    + Integer.toHexString(theme));
            if (theme != 0) {
                AttributeCache.Entry ent = AttributeCache.instance().get(pkg, theme,
                        com.android.internal.R.styleable.Window, mService.mCurrentUserId);
                if (ent == null) {
                    // Whoops!  App doesn't exist. Um. Okay. We'll just pretend like we didn't
                    // see that.
                    return false;
                }
                final boolean windowIsTranslucent = ent.array.getBoolean(
                        com.android.internal.R.styleable.Window_windowIsTranslucent, false);
                final boolean windowIsFloating = ent.array.getBoolean(
                        com.android.internal.R.styleable.Window_windowIsFloating, false);
                final boolean windowShowWallpaper = ent.array.getBoolean(
                        com.android.internal.R.styleable.Window_windowShowWallpaper, false);
                final boolean windowDisableStarting = ent.array.getBoolean(
                        com.android.internal.R.styleable.Window_windowDisablePreview, false);
                if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "Translucent=" + windowIsTranslucent
                        + " Floating=" + windowIsFloating
                        + " ShowWallpaper=" + windowShowWallpaper);
                if (windowIsTranslucent) {
                    return false;
                }
                if (windowIsFloating || windowDisableStarting) {
                    return false;
                }
                if (windowShowWallpaper) {
                    if (mContainer.getDisplayContent().mWallpaperController.getWallpaperTarget()
                            == null) {
                        // If this theme is requesting a wallpaper, and the wallpaper
                        // is not currently visible, then this effectively serves as
                        // an opaque window and our starting window transition animation
                        // can still work.  We just need to make sure the starting window
                        // is also showing the wallpaper.
                        windowFlags |= FLAG_SHOW_WALLPAPER;
                    } else {
                        return false;
                    }
                }
            }

            if (mContainer.transferStartingWindow(transferFrom)) {
                return true;
            }

            // There is no existing starting window, and we don't want to create a splash screen, so
            // that's it!
            if (type != STARTING_WINDOW_TYPE_SPLASH_SCREEN) {
                return false;
            }

            if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "Creating SplashScreenStartingData");
            mContainer.startingData = new SplashScreenStartingData(mService, pkg, theme,
                    compatInfo, nonLocalizedLabel, labelRes, icon, logo, windowFlags,
                    mContainer.getMergedOverrideConfiguration());
            scheduleAddStartingWindow();
        }
        return true;
    }
```

#### 创建窗口
scheduleAddStartingWindow有一个动画控制handler，把Starting Windows放到了第一个消息：
 
 ```java
     void scheduleAddStartingWindow() {
         // Note: we really want to do sendMessageAtFrontOfQueue() because we
         // want to process the message ASAP, before any other queued
         // messages.
         if (!mService.mAnimationHandler.hasCallbacks(mAddStartingWindow)) {
             if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "Enqueueing ADD_STARTING");
             mService.mAnimationHandler.postAtFrontOfQueue(mAddStartingWindow);
         }
     }
```     

mAddStartingWindow是一个runnable，mAddStartingWindow创建并添加了surface，如果发现了中断就用surface.remove()移除界面。
```java
    private final Runnable mAddStartingWindow = new Runnable() {

        @Override
        public void run() {
            final StartingData startingData;
            final AppWindowToken container;

            synchronized (mWindowMap) {
                if (mContainer == null) {
                    if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "mContainer was null while trying to"
                            + " add starting window");
                    return;
                }

                // There can only be one adding request, silly caller!
                mService.mAnimationHandler.removeCallbacks(this);

                startingData = mContainer.startingData;
                container = mContainer;
            }

            if (startingData == null) {
                // Animation has been canceled... do nothing.
                if (DEBUG_STARTING_WINDOW)
                    Slog.v(TAG_WM, "startingData was nulled out before handling"
                            + " mAddStartingWindow: " + mContainer);
                return;
            }

            if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "Add starting "
                    + AppWindowContainerController.this + ": startingData="
                    + container.startingData);

            StartingSurface surface = null;
            try {
                surface = startingData.createStartingSurface(container);
            } catch (Exception e) {
                Slog.w(TAG_WM, "Exception when adding starting window", e);
            }
            if (surface != null) {
                boolean abort = false;
                synchronized (mWindowMap) {
                    // If the window was successfully added, then
                    // we need to remove it.
                    if (container.removed || container.startingData == null) {
                        if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM,
                                "Aborted starting " + container
                                        + ": removed=" + container.removed
                                        + " startingData=" + container.startingData);
                        container.startingWindow = null;
                        container.startingData = null;
                        abort = true;
                    } else {
                        container.startingSurface = surface;
                    }
                    if (DEBUG_STARTING_WINDOW && !abort) Slog.v(TAG_WM,
                            "Added starting " + mContainer
                                    + ": startingWindow="
                                    + container.startingWindow + " startingView="
                                    + container.startingSurface);
                }
                if (abort) {
                    surface.remove();
                }
            } else if (DEBUG_STARTING_WINDOW) {
                Slog.v(TAG_WM, "Surface returned was null: " + mContainer);
            }
        }
    };
```

createStartingSurface属于StartingData的虚方法，实现在SplashScreenStartingData中：

```java
    @Override
    StartingSurface createStartingSurface(AppWindowToken atoken) {
        return mService.mPolicy.addSplashScreen(atoken.token, mPkg, mTheme, mCompatInfo,
                mNonLocalizedLabel, mLabelRes, mIcon, mLogo, mWindowFlags,
                mMergedOverrideConfiguration, atoken.getDisplayContent().getDisplayId());
    }
```


调用到PhoneWindowManager的addSplashScreen方法，根据样式添加界面，最后通过熟悉的WindowManager.addView显示在界面上，window的类型是TYPE_APPLICATION_STARTING。代码如下：

```java
   @Override
    public StartingSurface addSplashScreen(IBinder appToken, String packageName, int theme,
            CompatibilityInfo compatInfo, CharSequence nonLocalizedLabel, int labelRes, int icon,
            int logo, int windowFlags, Configuration overrideConfig, int displayId) {
        if (!SHOW_SPLASH_SCREENS) {
            return null;
        }
        if (packageName == null) {
            return null;
        }

        WindowManager wm = null;
        View view = null;

        try {
            Context context = mContext;
            if (DEBUG_SPLASH_SCREEN) Slog.d(TAG, "addSplashScreen " + packageName
                    + ": nonLocalizedLabel=" + nonLocalizedLabel + " theme="
                    + Integer.toHexString(theme));

            // Obtain proper context to launch on the right display.
            final Context displayContext = getDisplayContext(context, displayId);
            if (displayContext == null) {
                // Can't show splash screen on requested display, so skip showing at all.
                return null;
            }
            context = displayContext;

            if (theme != context.getThemeResId() || labelRes != 0) {
                try {
                    context = context.createPackageContext(packageName, CONTEXT_RESTRICTED);
                    context.setTheme(theme);
                } catch (PackageManager.NameNotFoundException e) {
                    // Ignore
                }
            }

            if (overrideConfig != null && !overrideConfig.equals(EMPTY)) {
                if (DEBUG_SPLASH_SCREEN) Slog.d(TAG, "addSplashScreen: creating context based"
                        + " on overrideConfig" + overrideConfig + " for splash screen");
                final Context overrideContext = context.createConfigurationContext(overrideConfig);
                overrideContext.setTheme(theme);
                final TypedArray typedArray = overrideContext.obtainStyledAttributes(
                        com.android.internal.R.styleable.Window);
                final int resId = typedArray.getResourceId(R.styleable.Window_windowBackground, 0);
                if (resId != 0 && overrideContext.getDrawable(resId) != null) {
                    // We want to use the windowBackground for the override context if it is
                    // available, otherwise we use the default one to make sure a themed starting
                    // window is displayed for the app.
                    if (DEBUG_SPLASH_SCREEN) Slog.d(TAG, "addSplashScreen: apply overrideConfig"
                            + overrideConfig + " to starting window resId=" + resId);
                    context = overrideContext;
                }
                typedArray.recycle();
            }

            final PhoneWindow win = new PhoneWindow(context);
            win.setIsStartingWindow(true);

            CharSequence label = context.getResources().getText(labelRes, null);
            // Only change the accessibility title if the label is localized
            if (label != null) {
                win.setTitle(label, true);
            } else {
                win.setTitle(nonLocalizedLabel, false);
            }

            win.setType(
                WindowManager.LayoutParams.TYPE_APPLICATION_STARTING);

            synchronized (mWindowManagerFuncs.getWindowManagerLock()) {
                // Assumes it's safe to show starting windows of launched apps while
                // the keyguard is being hidden. This is okay because starting windows never show
                // secret information.
                if (mKeyguardOccluded) {
                    windowFlags |= FLAG_SHOW_WHEN_LOCKED;
                }
            }

            // Force the window flags: this is a fake window, so it is not really
            // touchable or focusable by the user.  We also add in the ALT_FOCUSABLE_IM
            // flag because we do know that the next window will take input
            // focus, so we want to get the IME window up on top of us right away.
            win.setFlags(
                windowFlags|
                WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE|
                WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE|
                WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM,
                windowFlags|
                WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE|
                WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE|
                WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM);

            win.setDefaultIcon(icon);
            win.setDefaultLogo(logo);

            win.setLayout(WindowManager.LayoutParams.MATCH_PARENT,
                    WindowManager.LayoutParams.MATCH_PARENT);

            final WindowManager.LayoutParams params = win.getAttributes();
            params.token = appToken;
            params.packageName = packageName;
            params.windowAnimations = win.getWindowStyle().getResourceId(
                    com.android.internal.R.styleable.Window_windowAnimationStyle, 0);
            params.privateFlags |=
                    WindowManager.LayoutParams.PRIVATE_FLAG_FAKE_HARDWARE_ACCELERATED;
            params.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_SHOW_FOR_ALL_USERS;

            if (!compatInfo.supportsScreen()) {
                params.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
            }

            params.setTitle("Splash Screen " + packageName);
            addSplashscreenContent(win, context);

            wm = (WindowManager) context.getSystemService(WINDOW_SERVICE);
            view = win.getDecorView();

            if (DEBUG_SPLASH_SCREEN) Slog.d(TAG, "Adding splash screen window for "
                + packageName + " / " + appToken + ": " + (view.getParent() != null ? view : null));

            wm.addView(view, params);

            // Only return the view if it was successfully added to the
            // window manager... which we can tell by it having a parent.
            return view.getParent() != null ? new SplashScreenSurface(view, appToken) : null;
        } catch (WindowManager.BadTokenException e) {
            // ignore
            Log.w(TAG, appToken + " already running, starting window not displayed. " +
                    e.getMessage());
        } catch (RuntimeException e) {
            // don't crash if something else bad happens, for example a
            // failure loading resources because we are loading from an app
            // on external storage that has been unmounted.
            Log.w(TAG, appToken + " failed creating starting window", e);
        } finally {
            if (view != null && view.getParent() == null) {
                Log.w(TAG, "view not successfully added to wm, removing view");
                wm.removeViewImmediate(view);
            }
        }

        return null;
    }
```


### Starting Window移除

显示第一帧后Starting Window就可以消失了。


#### 调用入口

在AppWindowToken:onFirstWindowDrawn找到了调用：

```java
    void onFirstWindowDrawn(WindowState win, WindowStateAnimator winAnimator) {
        firstWindowDrawn = true;

        // We now have a good window to show, remove dead placeholders
        removeDeadWindows();

        if (startingWindow != null) {
            if (DEBUG_STARTING_WINDOW || DEBUG_ANIM) Slog.v(TAG, "Finish starting "
                    + win.mToken + ": first real window is shown, no animation");
            // If this initial window is animating, stop it -- we will do an animation to reveal
            // it from behind the starting window, so there is no need for it to also be doing its
            // own stuff.
            win.cancelAnimation();
            if (getController() != null) {
                getController().removeStartingWindow();
            }
        }
        updateReportedVisibilityLocked();
    }
```

另外还有handleClosingApps，notifyAppStopped等activity异常退出时也需要销毁掉 Starting Window，这里不一一看了，直接看remove流程。


#### remove流程

AppWindowContainerController的removeStartingWindow方法：
```java
  public void removeStartingWindow() {
        synchronized (mWindowMap) {
            if (mContainer.startingWindow == null) {
                if (mContainer.startingData != null) {
                    // Starting window has not been added yet, but it is scheduled to be added.
                    // Go ahead and cancel the request.
                    if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM,
                            "Clearing startingData for token=" + mContainer);
                    mContainer.startingData = null;
                }
                return;
            }

            final StartingSurface surface;
            if (mContainer.startingData != null) {
                surface = mContainer.startingSurface;
                mContainer.startingData = null;
                mContainer.startingSurface = null;
                mContainer.startingWindow = null;
                mContainer.startingDisplayed = false;
                if (surface == null && DEBUG_STARTING_WINDOW) {
                    Slog.v(TAG_WM, "startingWindow was set but startingSurface==null, couldn't "
                            + "remove");
                }
            } else {
                if (DEBUG_STARTING_WINDOW) {
                    Slog.v(TAG_WM, "Tried to remove starting window but startingWindow was null:"
                            + mContainer);
                }
                return;
            }

            if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "Schedule remove starting " + mContainer
                    + " startingWindow=" + mContainer.startingWindow
                    + " startingView=" + mContainer.startingSurface);

            // Use the same thread to remove the window as we used to add it, as otherwise we end up
            // with things in the view hierarchy being called from different threads.
            mService.mAnimationHandler.post(() -> {
                if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "Removing startingView=" + surface);
                try {
                    surface.remove();
                } catch (Exception e) {
                    Slog.w(TAG_WM, "Exception when removing starting window", e);
                }
            });
        }
    }
```

在mAnimationHandler中调用surface.remove()方法，和前面创建时一样。

StartingSurface的remove方法在SplashScreenSurface实现，直接调用WindowManager的removeView方法：

```java
class SplashScreenSurface implements StartingSurface {

    private static final String TAG = PhoneWindowManager.TAG;
    private final View mView;
    private final IBinder mAppToken;

    SplashScreenSurface(View view, IBinder appToken) {
        mView = view;
        mAppToken = appToken;
    }

    @Override
    public void remove() {
        if (DEBUG_SPLASH_SCREEN) Slog.v(TAG, "Removing splash screen window for " + mAppToken + ": "
                        + this + " Callers=" + Debug.getCallers(4));

        final WindowManager wm = mView.getContext().getSystemService(WindowManager.class);
        wm.removeView(mView);
    }
}
```

### 总结

Starting Window显示
时机：AMS找到Activity信息后，创建activity前，直接由AMS进程控制
流程：
1.AppWindowContainerController:addStartingWindow 控制是否添加
2.scheduleAddStartingWindow把mAddStartingWindow放到mAnimationHandler中执行
3.SplashScreenStartingData:createStartingSurface创建窗口
4.PhoneWindowManager:addSplashScreen通过WMS添加view

Starting Window消失
时机：activity显示第一帧或者activity异常退出
流程：
1.AppWindowContainerController:removeStartingWindow 如果surface存在则发消息到mAnimationHandler
2.mAnimationHandler执行surface.remove();
3.SplashScreenSurface的remove方法通过WMS移除view