---
title:      "NDK中使用MediaCodec编解码视频"
description:   "MediaCodec使用详解"
date:       2019-10-16 12:00:00
author:     "安地"
tags:
      - Android 
      - C 
      - 音视频

---


## 背景

MediaCodec 作为Android自带的视频编解码工具，可以直接利用底层硬件编解码能力，现在已经逐渐成为主流了。API21已经支持NDK方法了，MediaCodec api设计得非常精妙,另一个方面也是很多人觉得不好懂。


## 内容

### MediaCodec的两个Buffer和三板斧

MediaCodec内部包含InputBuffer和OutputBuffer，内部有一个自启线程，不断去查询两个Buffer，是一个生产者消费者模型。

进行数据处理时主要靠三板斧

第一步：取buffer地址 
AMediaCodec_dequeueInputBuffer

第二步：获取buffer数据
AMediaCodec_getInputBuffer

第三步：buffer入队
AMediaCodec_queueInputBuffer

InputBuffer和OutputBuffer基本是对称的：

第一步：取buffer地址 
AMediaCodec_dequeueOutputBuffer

第二步：获取buffer数据
AMediaCodec_getOutputBuffer

第三步：buffer释放
AMediaCodec_releaseOutputBuffer

只有第三步不同，AMediaCodec_queueInputBuffer是数据入队等待消费，AMediaCodec_releaseOutputBuffer是释放数据。
编码和解码过程，InputBuffer和OutputBuffer就互相置换下。
解码： 原始数据（视频流）-> 提取器AMediaExtractor->InputBuffer->OutputBuffer->帧数据(YUV420sp，PCM)
编码： 帧数据（视频流）->InputBuffer->OutputBuffer->合成器AMediaMuxer



### 解码


#### 解码配置


解码开始需要配置AMediaCodec和AMediaExtractor,MediaCodec start后就可以开始解码，


AMediaExtractor需要设置文件描述符，通过AAssetManager_open或者fopen就可以得到。起始点和长度也同样。然后设置进提取器。

```C
AMediaExtractor_setDataSourceFd(mExtractor, virtualFile.fd,
                                                         virtualFile.start,
                                                         virtualFile.length);
                                                         
```

AMediaCodec创建需要设置数据格式，通过AMediaExtractor获取到的AMediaFormat可以得到mime和format。

```C
mCodec = AMediaCodec_createDecoderByType(mime);
AMediaCodec_configure(mCodec, format, NULL, NULL, 0);
AMediaCodec_start(mCodec);
```

解码配置第三个参数为NativeWindow，加了后解码后可以直接吐到surface上，GPU数据直接渲软，效率高但不够灵活。不加的话解码数据就需要输出拷贝。


#### 解码流程

解码也就是操作两个Buffer的过程，执行玩三板斧就可以，然后有一些状态需要处理。

```C

    if (!mInputEof) {
        ssize_t bufidx = AMediaCodec_dequeueInputBuffer(mCodec, 1);
        log_info(NULL, "input buffer %zd", bufidx);
        if (bufidx >= 0) {
            size_t bufsize;
            uint8_t *buf = AMediaCodec_getInputBuffer(mCodec, bufidx, &bufsize);
            int sampleSize = AMediaExtractor_readSampleData(mExtractor, buf, bufsize);
            if (sampleSize < 0) {
                sampleSize = 0;
                mInputEof = true;
                log_info(NULL, "video producer input EOS");
            }
            int64_t presentationTimeUs = AMediaExtractor_getSampleTime(mExtractor);

            AMediaCodec_queueInputBuffer(mCodec, bufidx, 0, sampleSize, presentationTimeUs,
                                         mInputEof ? AMEDIACODEC_BUFFER_FLAG_END_OF_STREAM
                                                   : 0);
            AMediaExtractor_advance(mExtractor);
        }
    }

    if (!mOutputEof) {
        AMediaCodecBufferInfo info;
        ssize_t status = AMediaCodec_dequeueOutputBuffer(mCodec, &info, 1);

        if (status >= 0) {

            if (info.flags & AMEDIACODEC_BUFFER_FLAG_END_OF_STREAM) {
                log_info(NULL, "video producer output EOS");

                eof = true;
                mOutputEof = true;
            }

            uint8_t *outputBuf = AMediaCodec_getOutputBuffer(mCodec, status, NULL/* out_size */);
            size_t dataSize = info.size;
            if (outputBuf != nullptr && dataSize != 0) {
                long pts = info.presentationTimeUs;
                int32_t pts32 = (int32_t) pts;

                *buffer = (uint8_t *) mlt_pool_alloc(dataSize);
                memcpy(*buffer, outputBuf + info.offset, dataSize);
                *buffersize = dataSize;
            }

            int64_t presentationNano = info.presentationTimeUs * 1000;
            log_info(NULL, "video pts %lld outsize %d", info.presentationTimeUs, dataSize);
            /*if (delay > 0) {
                usleep(delay / 1000);
            }*/
            AMediaCodec_releaseOutputBuffer(mCodec, status, info.size != 0);
        } else if (status == AMEDIACODEC_INFO_OUTPUT_BUFFERS_CHANGED) {
            log_info(NULL, "output buffers changed");
        } else if (status == AMEDIACODEC_INFO_OUTPUT_FORMAT_CHANGED) {
            AMediaFormat_delete(format);
        } else if (status == AMEDIACODEC_INFO_TRY_AGAIN_LATER) {
            log_info(NULL, "video no output buffer right now");
        } else {
            log_info(NULL, "unexpected info code: %zd", status);
        }

    }
    
```

AMediaCodec和AMediaExtractor是没有直接交流的，AMediaCodec取到InputBuffer后实际数据为空，需要从AMediaExtractor_readSampleData获取到buffer数据。
AMediaCodec数据入队后，AMediaExtractor调用AMediaExtractor_advance前进到下一个数据位置。

OutputBuffer操作时有些不一样，AMediaCodec_dequeueOutputBuffer获取的是解码好的帧，AMediaCodec_getOutputBuffer取到的就已经是解码好的数据了，可以直接拷贝使用。
AMediaCodec_releaseOutputBuffer是释放buffer，如果配置了surface，就会渲软到surface上。

### 编码


#### 编码配置
编码是解码的逆过程，首先设置格式,然后根据格式创建编码器MediaCodec，再根据文件创建合成器MediaMuxer。


```C
void NativeEncoder::prepareEncoder(int width, int height, int fps, std::string strPath) {

    mWidth = width;
    mHeight = height;
    mFps = fps;

    AMediaFormat *format = AMediaFormat_new();
    AMediaFormat_setString(format, AMEDIAFORMAT_KEY_MIME, mStrMime.c_str());
    AMediaFormat_setInt32(format, AMEDIAFORMAT_KEY_WIDTH, mWidth);
    AMediaFormat_setInt32(format, AMEDIAFORMAT_KEY_HEIGHT, mHeight);

    AMediaFormat_setInt32(format,AMEDIAFORMAT_KEY_COLOR_FORMAT, COLOR_FORMAT_SURFACE);
    AMediaFormat_setInt32(format, AMEDIAFORMAT_KEY_BIT_RATE, mBitRate);
    AMediaFormat_setInt32(format, AMEDIAFORMAT_KEY_FRAME_RATE, mFps);
    AMediaFormat_setInt32(format, AMEDIAFORMAT_KEY_I_FRAME_INTERVAL, mIFrameInternal);

    const char *s = AMediaFormat_toString(format);
    log_info(NULL, "encoder video format: %s", s);


    mCodec = AMediaCodec_createEncoderByType(mStrMime);


    media_status_t status = AMediaCodec_configure(mCodec, format, NULL, NULL,
                                                  AMEDIACODEC_CONFIGURE_FLAG_ENCODE);
    if (status != 0) {
        log_error(NULL, "AMediaCodec_configure() failed with error %i for format %u",
                      (int) status, 21);
    } else {

    }
    AMediaFormat_delete(format);

    FILE *fp = fopen(strPath.c_str(), "wb");

    if (fp != NULL) {
        mFd = fileno(fp);
    } else {
        mFd = -1;
        log_error(NULL, "create file %s fail", strPath.c_str());
    }

    if(mMuxer == NULL)
        mMuxer = AMediaMuxer_new(mFd, AMEDIAMUXER_OUTPUT_FORMAT_MPEG_4);

    mMuxerStarted = false;

    fclose(fp);

}
```

这里注意下配置类型是 "video/avc"，基本视频都是这个格式，可以看官网格式支持信息，比特率mBitRate是6000000，这个要根据需求对应配置，I帧间隔mIFrameInternal是1秒，间隔长获取关键帧信息会有问题。

#### 编码准备

编码视频流需要创建一个surface，再把这个surface绑定到共享的EGLContext上。

```C

void NativeEncoder::prepareEncoderWithShareCtx(int width, int height, int fps, std::string strPath,
                                   EGLContext shareCtx) {

    prepareEncoder(width,height,fps,strPath);
    ANativeWindow *surface;
    AMediaCodec_createInputSurface(mCodec, &surface);
    media_status_t status;
    if ((status = AMediaCodec_start(mCodec)) != AMEDIA_OK) {
        log_error(NULL, "AMediaCodec_start: Could not start encoder.");
    } else {
        log_info(NULL, "AMediaCodec_start: encoder successfully started");
    }
    mCodecInputSurface = new CodecInputSurface(surface);
    mCodecInputSurface->setupEGL(shareCtx);
}
```


#### 编码流程

编码需要先进行渲染，从外部共享的EGLContext传入一个纹理，渲软到编码器对应的surface上，再进行编码。

传入纹理并渲染：

```C
void NativeEncoder::feedFrame(uint64_t pts, int tex) {
    drainEncoder(false);

    mCodecInputSurface->makeCurrent();
    glViewport(0,0,mWidth,mHeight);
    mCodecInputSurface->renderOnSurface(tex);
    mCodecInputSurface->setPresentationTime(pts);
    mCodecInputSurface->swapBuffers();
    mCodecInputSurface->makeNothingCurrent();
}
```

编码：

```C
void NativeEncoder::drainEncoder(bool eof) {

    if (eof) {

        ssize_t ret = AMediaCodec_signalEndOfInputStream(mCodec);
        log_info(NULL, "drainEncoder eof = %d",ret);
    }

    while (true) {

        AMediaCodecBufferInfo info;
        //time out usec 1
        ssize_t status = AMediaCodec_dequeueOutputBuffer(mCodec, &info, 1);

        if (status == AMEDIACODEC_INFO_TRY_AGAIN_LATER) {

            if (!eof) {
                break;
            } else {
                log_info(NULL, "video no output available, spinning to await EOS");
            }
        } else if (status == AMEDIACODEC_INFO_OUTPUT_BUFFERS_CHANGED) {
            // not expected for an encoder
        } else if (status == AMEDIACODEC_INFO_OUTPUT_FORMAT_CHANGED) {
            if (mMuxerStarted) {
                log_warning(NULL, "format changed twice");
            }

            AMediaFormat *fmt = AMediaCodec_getOutputFormat(mCodec);
            const char *s = AMediaFormat_toString(fmt);
            log_info(NULL, "video output format %s", s);

            mTrackIndex = AMediaMuxer_addTrack(mMuxer, fmt);

            if(mAudioTrackIndex != -1 && mTrackIndex != -1) {

                log_info(NULL,"AMediaMuxer_start");
                AMediaMuxer_start(mMuxer);
                mMuxerStarted = true;
            }

        } else {

            uint8_t *encodeData = AMediaCodec_getOutputBuffer(mCodec, status, NULL/* out_size */);

            if (encodeData == NULL) {
                log_error(NULL, "encoder output buffer was null");
            }

            if ((info.flags & AMEDIACODEC_BUFFER_FLAG_CODEC_CONFIG) != 0) {
                log_info(NULL, "ignoring AMEDIACODEC_BUFFER_FLAG_CODEC_CONFIG");
                info.size = 0;
            }

            size_t dataSize = info.size;

            if (dataSize != 0) {

                if (!mMuxerStarted) {
                    log_error(NULL, "muxer has't started");
                }
                log_info(NULL,"AMediaMuxer_writeSampleData video size %d",dataSize);
                AMediaMuxer_writeSampleData(mMuxer, mTrackIndex, encodeData, &info);
            }

            AMediaCodec_releaseOutputBuffer(mCodec, status, false);

            if ((info.flags & AMEDIACODEC_BUFFER_FLAG_END_OF_STREAM) != 0) {

                if (!eof) {
                    log_warning(NULL, "reached end of stream unexpectly");
                } else {
                    log_info(NULL, "video end of stream reached");
                }

                break;
            }
        }
    }
}
```

除了结尾标记，编码时没有操作InputBuffer，因为InputBuffer对应的就是surface的源，所以编码第一步实际是渲软，通过opengl render到surface上再交换缓冲区到surface上；
第二步获取到OutputBuffer数据，调用AMediaCodec_getOutputBuffer；
第三步合成器写数据，调用AMediaMuxer_writeSampleData然后释放outputBuffer，调用AMediaCodec_releaseOutputBuffer。



## 总结

总结了下MediaCodec在ndk中的使用，MediaCodec是一个非常灵活的api，编解码音视频都是同一个，掌握双缓冲和三板斧就对流程有了非常清楚的了解，对编解码代码也可以不再畏惧了。
