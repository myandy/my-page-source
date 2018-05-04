---
title:      "FramePlayer2——音频控制篇"
description: "使用AudioTrack进行音频播放"
date:       2017-3-22 15:00:00
author:     "安地"
tags:
      - Android
---

## 前言

上一篇介绍了控制视频的帧率的方法，使用MediaExtractor和MediaCodec加上自己设置的timer可以完全控制到每一帧的解析。音频控制没有帧的概念，音频就是波形，都是连贯的，控制速度更简单一点，读取到波形数据后，自己设置频率速度就可以了。

## 音频播放流程

第一步还是解封装，得到音频编码，音频编码格式主要有mp3，aac，wav等，其中wav是无损的不需要要解码，通过MediaCodec解码成后PCM（Pulse-code modulation：脉冲编码调制）就可以传给AudioTrack进行播放，MediaPlayer底层也使用了AudioTrack进行音频播放。

## 代码解析

先解封装和配置MediaCodec和AudioTrack，其中sampleRate就是脉冲频率，乘以一个视频原始帧率和播放帧率比例就达到了和视频保持一样的速度。最后起一个音频播放线程单独去播放。
``` java
            mAudioExtractor = new MediaExtractor();
            mAudioExtractor.setDataSource(sourceFile.toString());
            mAudioTrackIndex = selectAudioTrack(mAudioExtractor);

            //only support pcm
            if (mAudioTrackIndex != -1) {
                relaxResources(true);
                MediaFormat audioFormat = mMediaExtractor.getTrackFormat(mAudioTrackIndex);
                String audioMime = audioFormat.getString(MediaFormat.KEY_MIME);
                try {
                    // 实例化一个指定类型的解码器,提供数据输出
                    mAudioCodec = MediaCodec.createDecoderByType(audioMime);
                } catch (IOException e) {
                    e.printStackTrace();
                }

                mAudioCodec.configure(audioFormat, null /* surface */, null /* crypto */, 0 /* flags */);
                mAudioCodec.start();


                int channels = audioFormat.getInteger(MediaFormat.KEY_CHANNEL_COUNT);
                int sampleRate = (int) (1.0f * audioFormat.getInteger(MediaFormat.KEY_SAMPLE_RATE) * (1000 / videoFrameRate) / mFrameInterval);
                int channelConfiguration = channels == 1 ? AudioFormat.CHANNEL_OUT_MONO : AudioFormat.CHANNEL_OUT_STEREO;
                audioTrack = new AudioTrack(
                        AudioManager.STREAM_MUSIC,
                        sampleRate,
                        channelConfiguration,
                        AudioFormat.ENCODING_PCM_16BIT,
                        AudioTrack.getMinBufferSize(
                                sampleRate,
                                channelConfiguration,
                                AudioFormat.ENCODING_PCM_16BIT
                        ),
                        AudioTrack.MODE_STREAM
                );

                //开始play，等待write发出声音
                audioTrack.play();
                mAudioExtractor.selectTrack(mAudioTrackIndex);

                if (mAudioPlayTask != null) {
                    mAudioPlayTask.cancel(true);
                }
                mAudioPlayTask = new AudioPlayTask();
                mAudioPlayTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);

            }
```
AudioPlayTask只做一件事，doAudio中在while循环中解码播放，解码读入读出和视频解析很相似，不过最后需要调用audioTrack.write来发出声音
``` java
    /**
     * AsyncTask that takes care of running the decode/playback loop
     */
    private class AudioPlayTask extends AsyncTask<Void, Void, Void> {

        @Override
        protected Void doInBackground(Void... values) {
            doAudio();
            return null;
        }

        @Override
        protected void onPreExecute() {
        }

        @Override
        protected void onProgressUpdate(Void... values) {
        }
    }

    private void doAudio() {

        ByteBuffer[] codecInputBuffers;
        ByteBuffer[] codecOutputBuffers;


        MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();

        codecInputBuffers = mAudioCodec.getInputBuffers();
        // 解码后的数据
        codecOutputBuffers = mAudioCodec.getOutputBuffers();

        // 解码
        boolean sawInputEOS = false;
        boolean sawOutputEOS = false;
        int noOutputCounter = 0;
        int noOutputCounterLimit = 50;

        int inputBufIndex;
        doStop = false;
        while (!sawOutputEOS && noOutputCounter < noOutputCounterLimit && !doStop) {

            if (!isRunning) {
                try {
                    //防止死循环ANR
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                continue;
            }

            noOutputCounter++;
            if (!sawInputEOS) {
                if (seekOffsetFlag) {
                    seekOffsetFlag = false;
                    mAudioExtractor.seekTo(seekOffset, SEEK_TO_PREVIOUS_SYNC);
                }

                inputBufIndex = mAudioCodec.dequeueInputBuffer(TIMEOUT_USEC);

                if (inputBufIndex >= 0) {
                    ByteBuffer dstBuf = codecInputBuffers[inputBufIndex];

                    int sampleSize =
                            mAudioExtractor.readSampleData(dstBuf, 0 /* offset */);

                    long presentationTimeUs = 0;

                    if (sampleSize < 0) {
                        sawInputEOS = true;
                        sampleSize = 0;
                    } else {
                        presentationTimeUs = mAudioExtractor.getSampleTime();
                    }
                    curPosition = presentationTimeUs;
                    mAudioCodec.queueInputBuffer(
                            inputBufIndex,
                            0 /* offset */,
                            sampleSize,
                            presentationTimeUs,
                            sawInputEOS ? MediaCodec.BUFFER_FLAG_END_OF_STREAM : 0);


                    if (!sawInputEOS) {
                        mAudioExtractor.advance();
                    }
                } else {
                    Log.e(TAG, "inputBufIndex " + inputBufIndex);
                }
            }

            // decode to PCM and push it to the AudioTrack player
            // 解码数据为PCM
            int res = mAudioCodec.dequeueOutputBuffer(info, TIMEOUT_USEC);

            if (res >= 0) {
                if (info.size > 0) {
                    noOutputCounter = 0;
                }
                int outputBufIndex = res;
                ByteBuffer buf = codecOutputBuffers[outputBufIndex];

                final byte[] chunk = new byte[info.size];
                buf.get(chunk);
                buf.clear();
                if (chunk.length > 0 && audioTrack != null && !doStop) {
                    //播放
                    audioTrack.write(chunk, 0, chunk.length);
                    hadPlay = true;
                }
                //释放
                mAudioCodec.releaseOutputBuffer(outputBufIndex, false /* render */);
                if ((info.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                    sawOutputEOS = true;
                }
            } else if (res == MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED) {
                codecOutputBuffers = mAudioCodec.getOutputBuffers();
            } else if (res == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
                MediaFormat oformat = mAudioCodec.getOutputFormat();
            } else {
            }
        }

        relaxResources(true);

        doStop = true;

        if (sawOutputEOS) {
            try {
                if (isLoop || !hadPlay) {
                    doAudio();
                    return;
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```
## 后记

音频变速两倍差异后就已经不能入耳，所以网易云课堂提供的变速只有1.25和1.5，视频更大的变速播放可以用来扫描内容，自己拿来玩玩还是挺有意思的。项目代码不够完善，只是作为玩具和研究。
github地址：<https://github.com/myandy/frameplayer>