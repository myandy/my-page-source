---
title:      "FramePlayer1——视频控制篇"
description: "基于MediaCodec的帧率控制播放器"
date:       2017-3-9 15:00:00
author:     "安地"
tags:
        - Android
    
---

## 前言

最近项目中用到了MediaCodec和MediaExtractor用作录制视频，但发现在后台录制时绘制过慢，只能去控制录制速度，但录制目标包括视频，那就需要一个按帧来播放的视频播放器了。
就借鉴google grafika的MoviePlayer写了一个播放器，这里是一个简单的demo，还没有完善，用于测试视频还是挺好用的。它实现了视频的控制帧率播放，还可以手动显示下一帧。

## MediaExtractor 和 MediaCodec 介绍

MediaExtractor用于解封装，解出音频流和视频流，可以获取到帧率，时长，视频大小等信息，MediaCodec用于解码视频，可以播放到一个surface上。这两者都是Android暴露的一个比较接近底层解码器的API，加上opengl可以做很多强大的功能。
同时这两者也可以用于加封装和编码视频，就是可以用于录制视频，更具体说是录制surface，具体可以看grafika中的实现。

## FramePlayer 介绍

MoviePlayer中线程开始后就不间断播放，自己控制速度，然后给出一个接口可以做时间的暂停，就是只能降低速度播放。
我想做一个完全自己控制速度的播放器，就加了一个消息队列，接受到一次播放请求才播放下一帧，通过timer控制播放帧率。


初始化线程和消息队列，接受两个消息开始播放和播放一帧：
``` java
        @Override
        public void run() {
            // Establish a Looper for this thread, and define a Handler for it.
            Looper.prepare();
            mLocalHandler = new LocalHandler();
            Looper.loop();
        }

        private class LocalHandler extends Handler {
            @Override
            public void handleMessage(Message msg) {
                int what = msg.what;
                Log.d(TAG, "handleMessage:" + what);
                switch (what) {
                    case MSG_PLAY_START:
                        try {
                            play();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                        break;
                    case MSG_PLAY_PROGRESS:
                        doExtract(msg.arg1);
                        break;
                    default:
                        throw new RuntimeException("Unknown msg " + what);
                }
            }
        }
```

启动播放器，解封装和解码，然后启动一个TimerTask
``` java
     private void play() throws IOException {
            if (!sourceFile.canRead()) {
                throw new FileNotFoundException("Unable to read " + sourceFile);
            }
            try {
                destroyExtractor();
                mMediaExtractor = new MediaExtractor();
                mMediaExtractor.setDataSource(sourceFile.toString());
                mTrackIndex = selectTrack(mMediaExtractor);
                if (mTrackIndex < 0) {
                    throw new RuntimeException("No video track found in " + sourceFile);
                }
                mMediaExtractor.selectTrack(mTrackIndex);

                MediaFormat format = mMediaExtractor.getTrackFormat(mTrackIndex);
                if (mFrameInterval == 0) {
                    mFrameInterval = 1000 / format.getInteger(MediaFormat.KEY_FRAME_RATE);
                }
                String mime = format.getString(MediaFormat.KEY_MIME);
                duration = format.getLong(MediaFormat.KEY_DURATION);
                mMediaCodec = MediaCodec.createDecoderByType(mime);
                mMediaCodec.configure(format, mOutputSurface, null, 0);
                mMediaCodec.start();

                timer = new Timer();
                timerTask = new ProgressTimerTask();
                timer.schedule(timerTask, 0, mFrameInterval);
            } catch (Exception e) {
                Log.e(TAG, e.toString());
                destroyExtractor();
            }
        }
```
 TimerTask发送消息到播放线程播放一帧，isRunning用于控制暂停继续，nextFrame方法可以直接播放下一帧：
``` java
            public void nextFrame() {
                mLocalHandler.sendMessage(mLocalHandler.obtainMessage(MSG_PLAY_PROGRESS, frame++, 0));
            }

            private class ProgressTimerTask extends TimerTask {
                @Override
                public void run() {
                    if (isRunning) {
                        mLocalHandler.sendMessage(mLocalHandler.obtainMessage(MSG_PLAY_PROGRESS, frame++, 0));
                    }
                }
            }
```

播放一帧，MediaCodec分读取和输出队列，每次都只操作一帧：
``` java
        private void doExtract(int frame) {
                ByteBuffer inputBuffer;
                if (mMediaCodec == null) {
                    return;
                }
                int inputBufferIndex = mMediaCodec.dequeueInputBuffer(TIMEOUT_USEC);
                if (inputBufferIndex >= 0) {
                    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
                        // 从输入队列里去空闲buffer
                        inputBuffer = mMediaCodec.getInputBuffers()[inputBufferIndex];
                        inputBuffer.clear();
                    } else {
                        // SDK_INT > LOLLIPOP
                        inputBuffer = mMediaCodec.getInputBuffer(inputBufferIndex);
                    }
                    if (null != inputBuffer) {
                        int chunkSize = mMediaExtractor.readSampleData(inputBuffer, 0);
                        if (chunkSize < 0) {
                            if (isRunning && playListener != null) {
                                playListener.onCompleted();
                                isRunning = false;
                                destroyExtractor();
                                return;
                            }
                            mMediaCodec.queueInputBuffer(inputBufferIndex, 0, 0, 0L,
                                    MediaCodec.BUFFER_FLAG_END_OF_STREAM);
                            if (VERBOSE) Log.d(TAG, "sent input EOS");
                        } else {
                            mMediaCodec.queueInputBuffer(inputBufferIndex, 0, chunkSize, 0, 0);
                            mMediaExtractor.advance();
                        }
                    }
                    int outputBufferIndex = mMediaCodec.dequeueOutputBuffer(mBufferInfo, TIMEOUT_USEC);
                    Log.d(TAG, outputBufferIndex + ":outputBufferIndex");
                    if (outputBufferIndex >= 0) {
                        mMediaCodec.releaseOutputBuffer(outputBufferIndex, true);
                        if (playListener != null) {
                            playListener.onProgress((frame + 1) * mFrameInterval * 1f / duration);
                        }
                    } else {
                        Log.d(TAG, "Reached EOS, looping");
                    }
                }
            }
```
使用很简单，用一个SurfaceView addCallback然后就可以创建一个FramePlayer，不设置帧率的话就会使用视频本身的帧率，详细可以参照工程代码：
``` java
     @Override
        public void surfaceCreated(SurfaceHolder holder) {
            if (framePlayer == null) {
                Surface surface = holder.getSurface();
                framePlayer = new FramePlayer(surface);
                framePlayer.setSourceFile(new File(videoPath));
                framePlayer.execute();
                framePlayer.setPlayListener(new FramePlayer.PlayListener() {
                    @Override
                    public void onCompleted() {
                        Log.d("PlayListener", "onCompleted");
                    }

                    @Override
                    public void onProgress(float progress) {
                        Log.d("PlayListener", "onProgress" + progress);
                    }
                });
            }
        }
```
## 后记

Android真是一个玩具，不过现在Google对权限控制越来越严格了。
github地址：<https://github.com/myandy/frameplayer>