---
title:      "Android后台任务的多种实现方式及使用场景介绍"
description: "AsyncTaskLoader和JobService的详细解析"
date:       2017-1-17 12:00:00
author:     "安地"
tags:
        - Android
    
---

## 后台任务实现方式

Android后台任务常用的方法主要有两种，AsyncTask和IntentService。
AsyncTask应该大家都用过，适合和界面强相关耗时短的任务，背后原理也就是用Executor执行线程后台任务，handler发消息到主线程。
IntentService相对用得少些，适合长时间后台任务，它不可以直接和应用的界面进行交互，可以处理intent携带的消息。背后原理复杂一些，本身是一个service，又创建了一个消息队列。在OnCreate创建了一个HandlerThread，顾名思义，一个带Handler的Thread，它创建一个Looper，循环读取消息。IntentService在onStart时处理Intent，发消息到handlerThread处理，从而在子线程执行onHandleIntent方法。

这两种介绍都很多，这里就不过多介绍了。另外有两种方法使用得比较少，AsyncTaskLoader和JobService，研究一下。

## AsyncTaskLoader

AsyncTaskLoader 属于Loader家族，Loader家族常见的还有CursorLoader，LoaderManager，用于执行耗时加载过程。
Loader本身不具备异步加载数据的能力；AsyncTaskLoader继承自Loader，通过AsyncTask可以异步加载数据；CursorLoader继承自AsyncTaskLoader，可以异步加载Cursor数据。


AsyncTaskLoader实现是在onForceLoad时创建了一个LoadTask，继承自AsyncTask，在状态方法中执行AsyncTaskLoader的对应方法，CountDownLatch用于同步计数。

     final class LoadTask extends AsyncTask<Void, Void, D> implements Runnable {
            private final CountDownLatch mDone = new CountDownLatch(1);
            boolean waiting;

            LoadTask() {
            }

            protected D doInBackground(Void... params) {
                try {
                    Object ex = AsyncTaskLoader.this.onLoadInBackground();
                    return ex;
                } catch (OperationCanceledException var3) {
                    if(!this.isCancelled()) {
                        throw var3;
                    } else {
                        return null;
                    }
                }
            }

            protected void onPostExecute(D data) {
                try {
                    AsyncTaskLoader.this.dispatchOnLoadComplete(this, data);
                } finally {
                    this.mDone.countDown();
                }

            }

            protected void onCancelled(D data) {
                try {
                    AsyncTaskLoader.this.dispatchOnCancelled(this, data);
                } finally {
                    this.mDone.countDown();
                }

            }

            public void run() {
                this.waiting = false;
                AsyncTaskLoader.this.executePendingTask();
            }

            public void waitForLoader() {
                try {
                    this.mDone.await();
                } catch (InterruptedException var2) {
                    ;
                }

            }
        }

 Loader是靠LoaderManager管理的，LoaderManager可以同时管理多个Loader，即LoaderManager与Loader是一对多的关系。我们是在Activity或Fragment使用Loader的，虽然Loader有很多public方法，但是我们不能直接调用Loader的这些public方法，因为这会扰乱LoaderManag对Loader的正常管理，我们应该通过LoaderManager的initLoader以及restartLoader方法间接管理Loader，并通过LoaderCallbacks的回调方法（onCreateLoader、onLoadFinished、onLoaderReset）对数据进行相应处理。
 LoaderManager加载Loader代码：

         LoaderManager manager = this.getLoaderManager();
         manager.initLoader(0, null, myLoader);

 先获取到LoaderManager，再initLoader，参数1是Loader id，参数2是Bundle参数，参数3是LoaderCallbacks，其代码如下：

         private LoaderCallbacks<D> myLoader = new LoaderCallbacks<D>() {
                 //  2.1创建一个新的CursorLoader对象，携带游标
                 public Loader<D> onCreateLoader(int id, Bundle args) {
                     return new MyLoader();
                 }
                 //  2.2加载完成后，更新UI信息
                 public void onLoadFinished(Loader<D> loader, D d) {

                 }
                 public void onLoaderReset(Loader<D> arg0) {
                 }
         };

  在onCreateLoader方法new一个Loader，加载完成回调onLoadFinished方法可以更新UI信息。


 Loader的优点是当数据源改变时能及时通知客户端，发生configuration change时自动重连接。
 和AsyncTask比较的话，我觉得除非需要实时刷新进度，其它情况使用AsyncTaskLoader是一个更好的选择。


## JobService

Google在Android 5.0中引入JobScheduler来执行一些需要满足特定条件但不紧急的后台任务，APP利用JobScheduler来执行这些特殊的后台任务时来减少电量的消耗。

JobService继承自Service,实现了onBind，binder代码如下：

    /** Binder for this service. */
        IJobService mBinder = new IJobService.Stub() {
            @Override
            public void startJob(JobParameters jobParams) {
                ensureHandler();
                Message m = Message.obtain(mHandler, MSG_EXECUTE_JOB, jobParams);
                m.sendToTarget();
            }
            @Override
            public void stopJob(JobParameters jobParams) {
                ensureHandler();
                Message m = Message.obtain(mHandler, MSG_STOP_JOB, jobParams);
                m.sendToTarget();
            }
        };

        /** @hide */
        void ensureHandler() {
            synchronized (mHandlerLock) {
                if (mHandler == null) {
                    mHandler = new JobHandler(getMainLooper());
                }
            }
        }

代码就是生成Handler，发送开始和结束消息，再看handler代码：


     /**
         * Runs on application's main thread - callbacks are meant to offboard work to some other
         * (app-specified) mechanism.
         * @hide
         */
        class JobHandler extends Handler {
            JobHandler(Looper looper) {
                super(looper);
            }

            @Override
            public void handleMessage(Message msg) {
                final JobParameters params = (JobParameters) msg.obj;
                switch (msg.what) {
                    case MSG_EXECUTE_JOB:
                        try {
                            boolean workOngoing = JobService.this.onStartJob(params);
                            ackStartMessage(params, workOngoing);
                        } catch (Exception e) {
                            Log.e(TAG, "Error while executing job: " + params.getJobId());
                            throw new RuntimeException(e);
                        }
                        break;
                    case MSG_STOP_JOB:
                        try {
                            boolean ret = JobService.this.onStopJob(params);
                            ackStopMessage(params, ret);
                        } catch (Exception e) {
                            Log.e(TAG, "Application unable to handle onStopJob.", e);
                            throw new RuntimeException(e);
                        }
                        break;
                    case MSG_JOB_FINISHED:
                        final boolean needsReschedule = (msg.arg2 == 1);
                        IJobCallback callback = params.getCallback();
                        if (callback != null) {
                            try {
                                callback.jobFinished(params.getJobId(), needsReschedule);
                            } catch (RemoteException e) {
                                Log.e(TAG, "Error reporting job finish to system: binder has gone" +
                                        "away.");
                            }
                        } else {
                            Log.e(TAG, "finishJob() called for a nonexistent job id.");
                        }
                        break;
                    default:
                        Log.e(TAG, "Unrecognised message received.");
                        break;
                }
            }

            private void ackStartMessage(JobParameters params, boolean workOngoing) {
                final IJobCallback callback = params.getCallback();
                final int jobId = params.getJobId();
                if (callback != null) {
                    try {
                         callback.acknowledgeStartMessage(jobId, workOngoing);
                    } catch(RemoteException e) {
                        Log.e(TAG, "System unreachable for starting job.");
                    }
                } else {
                    if (Log.isLoggable(TAG, Log.DEBUG)) {
                        Log.d(TAG, "Attempting to ack a job that has already been processed.");
                    }
                }
            }

            private void ackStopMessage(JobParameters params, boolean reschedule) {
                final IJobCallback callback = params.getCallback();
                final int jobId = params.getJobId();
                if (callback != null) {
                    try {
                        callback.acknowledgeStopMessage(jobId, reschedule);
                    } catch(RemoteException e) {
                        Log.e(TAG, "System unreachable for stopping job.");
                    }
                } else {
                    if (Log.isLoggable(TAG, Log.DEBUG)) {
                        Log.d(TAG, "Attempting to ack a job that has already been processed.");
                    }
                }
            }
        }

在收到MSG_EXECUTE_JOB消息时通过JobService.this.onStartJob(params)来执行任务，ackStartMessage可以看到里面是执行callback方法，就是操作执行完成后的回调方法。


JobService是通过JobScheduler来执行一个JobInfo，调用过程如下：


    mJobScheduler = (JobScheduler) getSystemService( Context.JOB_SCHEDULER_SERVICE );

    JobInfo.Builder builder = new JobInfo.Builder( 1, new ComponentName( getPackageName(),MyJobService.class.getName()));


    if( mJobScheduler.schedule( builder.build() ) <= 0 ) {
        //If something goes wrong
    }


builder的new 方法传的是jobId和jobService组件名，还可以设置各种参数，比如延迟时间，网络状态，充电状态等。JobScheduler对象的schedule方法把JobInfo贴进任务表，就会按照设定执行了。

有一个要注意的是JobService各种没有自己起线程，所以不能执行耗时任务，如果有需要，要再起一个线程去做，JobService可以结合Thread/AsyncTask/AsyncTaskLoader使用。

## 总结

执行后台任务要弄清需求场景选择合适的方法策略使用。
理解各种方法的原理，多看源码。


