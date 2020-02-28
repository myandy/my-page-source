---
title:      "ijkPlayer深入探究(一)"
description:   "Android硬解设置和解码流程深入解析"
date:       2020-2-27 14:00:00
author:     "安地"
tags:
      - C
      - 音视频
      - Android
---


## 前言

ijkPlayer是开源做得最好的播放器，使用LGPL协议，非常适合播放器使用，也支持二次开发。
综合很多方案后，在项目中选用了ijkPlayer。ijkPlayer虽然使用广泛，人气非常高，核心代码解析也有不少，
但少有细节清楚的，还是得依靠源码。

所以接下来我会基于使用，来分享一些ijkPlayer中的知识点。
第一篇分析ijkPlayer的Android硬件解码流程。


## 参数设置

ijkplayer可以通过option设置做出很多更改，mediacodec解码部分就有以下这些配置。

```java
        ijkMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec", 1);//开启硬解码
        ijkMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec-auto-rotate", value);
        ijkMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec-handle-resolution-change", value);
        
        ijkMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec-hevc", 1);
        ijkMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec-sync", 1);
```

为1时开启，为0时关闭。这些值默认都是关闭。官方demo里面有前三个。
"mediacodec"为是否开启硬解，
"mediacodec-auto-rotate"为是否自动旋转，
"mediacodec-handle-resolution-change"为是否自动分辨率更改，
"mediacodec-hevc"为是否支持h265,这个值要开启才能走h265的硬解，还有其它的格式配置没有列举。如果格式不支持就会硬解初始化不成功，转用软件解码。
"mediacodec-sync"为是否同步解码，异步效率更高。

这里配置名称要非常注意，java中写法都是"-"，而c代码中的配置都是"_"。比如mediacodec-hevc在源码中是mediacodec_hevc。


## 分析

分析源码首先是找到调用流程，把握整体脉络，再去细节上体会原理。
可以入口开始分析，也可以从出口分析。入口就是整体的启动，出口就是关键节点方法，比如MediaCodec的解码方法，FFmpeg的解码方法。
这里为叙述方便从入口分析。

### 初始化

ijkplayer有一个管道的概念，不同平台走不同的管道方法.看ijkplayer/android下的文件。
入口就在ijkplayer_jni.c的IjkMediaPlayer_native_setup这个jni方法。

```C
static void
IjkMediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    MPTRACE("%s\n", __func__);
    IjkMediaPlayer *mp = ijkmp_android_create(message_loop); 
    JNI_CHECK_GOTO(mp, env, "java/lang/OutOfMemoryError", "mpjni: native_setup: ijkmp_create() failed", LABEL_RETURN);

    jni_set_media_player(env, thiz, mp);
    ijkmp_set_weak_thiz(mp, (*env)->NewGlobalRef(env, weak_this));
    ijkmp_set_inject_opaque(mp, ijkmp_get_weak_thiz(mp));
    ijkmp_set_ijkio_inject_opaque(mp, ijkmp_get_weak_thiz(mp));
    ijkmp_android_set_mediacodec_select_callback(mp, mediacodec_select_callback, ijkmp_get_weak_thiz(mp));

LABEL_RETURN:
    ijkmp_dec_ref_p(&mp);
}
```

message_loop创建一个死循环，接受消息并再通过handler发送给Android层。
这些不细究，继续看ijkmp_android_create做了什么。

```C
IjkMediaPlayer *ijkmp_android_create(int(*msg_loop)(void*))
{
    IjkMediaPlayer *mp = ijkmp_create(msg_loop);
    if (!mp)
        goto fail;

    mp->ffplayer->vout = SDL_VoutAndroid_CreateForAndroidSurface();
    if (!mp->ffplayer->vout)
        goto fail;

    mp->ffplayer->pipeline = ffpipeline_create_from_android(mp->ffplayer);
    if (!mp->ffplayer->pipeline)
        goto fail;

    ffpipeline_set_vout(mp->ffplayer->pipeline, mp->ffplayer->vout);

    return mp;

fail:
    ijkmp_dec_ref_p(&mp);
    return NULL;
}
```

这里设置了两个关键参数vout用于输出，pipeline用于解码，都是平台相关。显示部分就是surface转成nativewindow，再对应ffmpeg和mediacodec的使用。这部分不细究。
继续看ffpipeline_create_from_android如何做的。

```C
IJKFF_Pipeline *ffpipeline_create_from_android(FFPlayer *ffp)
{
    ALOGD("ffpipeline_create_from_android()\n");
    IJKFF_Pipeline *pipeline = ffpipeline_alloc(&g_pipeline_class, sizeof(IJKFF_Pipeline_Opaque));
    if (!pipeline)
        return pipeline;

    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    opaque->ffp                   = ffp;
    opaque->surface_mutex         = SDL_CreateMutex();
    opaque->left_volume           = 1.0f;
    opaque->right_volume          = 1.0f;
    if (!opaque->surface_mutex) {
        ALOGE("ffpipeline-android:create SDL_CreateMutex failed\n");
        goto fail;
    }

    pipeline->func_destroy              = func_destroy;
    pipeline->func_open_video_decoder   = func_open_video_decoder;
    pipeline->func_open_audio_output    = func_open_audio_output;
    pipeline->func_init_video_decoder   = func_init_video_decoder;
    pipeline->func_config_video_decoder = func_config_video_decoder;

    return pipeline;
fail:
    ffpipeline_free_p(&pipeline);
    return NULL;
}
```

这里有打开视频解码器和init视频解码器的方法，都由管道转发，在ffpipeline_android.c中：

```C
IJKFF_Pipenode* ffpipeline_open_video_decoder(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    return pipeline->func_open_video_decoder(pipeline, ffp);
}

IJKFF_Pipenode* ffpipeline_init_video_decoder(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    return pipeline->func_init_video_decoder(pipeline, ffp);
}
```

func_open_video_decoder和func_init_video_decoder方法里面的实现很多是重复的，都是获取一个IJKFF_Pipenode。
我们再具体分析流程，找出原因。找调用发现ffpipeline_init_video_decoder在ffplay中准备阶段就调用了，在stream_open中被调用，而func_open_video_decoder在stream_component_open中被调用，且是node_vdec初始化失败才会调用。代码如下：


```C
stream_component_open：
       if (ffp->async_init_decoder) {
            while (!is->initialized_decoder) {
                SDL_Delay(5);
            }
            if (ffp->node_vdec) {
                is->viddec.avctx = avctx;
                ret = ffpipeline_config_video_decoder(ffp->pipeline, ffp);
            }
            if (ret || !ffp->node_vdec) {
                decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
                ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
                if (!ffp->node_vdec)
                    goto fail;
            }
        } else {
            decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
            ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
            if (!ffp->node_vdec)
                goto fail;
        }
        if ((ret = decoder_start(&is->viddec, video_thread, ffp, "ff_video_dec")) < 0)
            goto out;
```
这里看到更多细节，打开流时先看解码器初始化模式，如果是异步，没有初始化完成就dealy5毫秒。
再如果没有初始化完成，就是要ffpipeline_open_video_decoder启动。非异步就直接使用ffpipeline_open_video_decoder打开。结束后再打开video_thread线程，这个留待后面分析。
打开解码器方法func_open_video_decoder：

```C
static IJKFF_Pipenode *func_open_video_decoder(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    IJKFF_Pipenode        *node = NULL;

    if (ffp->mediacodec_all_videos || ffp->mediacodec_avc || ffp->mediacodec_hevc || ffp->mediacodec_mpeg2)
        node = ffpipenode_create_video_decoder_from_android_mediacodec(ffp, pipeline, opaque->weak_vout);
    if (!node) {
        node = ffpipenode_create_video_decoder_from_ffplay(ffp);
    }

    return node;
}
```
通过格式支持先尝试打开硬件解码，如果node没有成功创建再使用ffplay进行软件解码。
再看ffpipenode_create_video_decoder_from_android_mediacodec方法：
```C
IJKFF_Pipenode *ffpipenode_create_video_decoder_from_android_mediacodec(FFPlayer *ffp, IJKFF_Pipeline *pipeline, SDL_Vout *vout)
{
    ALOGD("ffpipenode_create_video_decoder_from_android_mediacodec()\n");
    if (SDL_Android_GetApiLevel() < IJK_API_16_JELLY_BEAN)
        return NULL;

    if (!ffp || !ffp->is)
        return NULL;

    IJKFF_Pipenode *node = ffpipenode_alloc(sizeof(IJKFF_Pipenode_Opaque));
    if (!node)
        return node;

    VideoState            *is     = ffp->is;
    IJKFF_Pipenode_Opaque *opaque = node->opaque;
    JNIEnv                *env    = NULL;
    int                    ret    = 0;
    jobject                jsurface = NULL;

    node->func_destroy  = func_destroy;
    //设置解码线程
    if (ffp->mediacodec_sync) {
        node->func_run_sync = func_run_sync_loop;
    } else {
        node->func_run_sync = func_run_sync;
    }
    node->func_flush    = func_flush;
    opaque->pipeline    = pipeline;
    opaque->ffp         = ffp;
    opaque->decoder     = &is->viddec;
    opaque->weak_vout   = vout;

    opaque->codecpar = avcodec_parameters_alloc();
    if (!opaque->codecpar)
        goto fail;
    
    //获取解码器参数
    ret = avcodec_parameters_from_context(opaque->codecpar, opaque->decoder->avctx);
    if (ret)
        goto fail;

    switch (opaque->codecpar->codec_id) {
    case AV_CODEC_ID_H264:
        if (!ffp->mediacodec_avc && !ffp->mediacodec_all_videos) {
            ALOGE("%s: MediaCodec: AVC/H264 is disabled. codec_id:%d \n", __func__, opaque->codecpar->codec_id);
            goto fail;
        }
        switch (opaque->codecpar->profile) {
            case FF_PROFILE_H264_BASELINE:
                ALOGI("%s: MediaCodec: H264_BASELINE: enabled\n", __func__);
                break;
            case FF_PROFILE_H264_CONSTRAINED_BASELINE:
                ALOGI("%s: MediaCodec: H264_CONSTRAINED_BASELINE: enabled\n", __func__);
                break;
           ...
    case AV_CODEC_ID_HEVC:
        if (!ffp->mediacodec_hevc && !ffp->mediacodec_all_videos) {
            ALOGE("%s: MediaCodec/HEVC is disabled. codec_id:%d \n", __func__, opaque->codecpar->codec_id);
            goto fail;
        }
        strcpy(opaque->mcc.mime_type, SDL_AMIME_VIDEO_HEVC);
        opaque->mcc.profile = opaque->codecpar->profile;
        opaque->mcc.level   = opaque->codecpar->level;
        break;
    ...
fail:
    ffpipenode_free_p(&node);
    return NULL;
}
```
这里通过mediacodec_sync这个配置设置了解码线程，同步则使用func_run_sync，异步使用func_run_sync。
通过codecpar的codec_id和profile的判断是否支持，不支持就会失败，然后回到前面走软解ffplay。

### 解码

#### 启动解码线程

通过初始化我们获得了IJKFF_Pipenode的节点管道，即ffp的node_vdec。且给node设置了func_run_sync方法。
func_run_sync在ff_ffpipenode.c中被转发：

```C
int ffpipenode_run_sync(IJKFF_Pipenode *node)
{
    return node->func_run_sync(node);
}
```
如名称，这是一个线程。在ffplay中video_thread中被启动，就是前面stream_component_open中打开解码器后被调用的方法。
```C
static int video_thread(void *arg)
{
    FFPlayer *ffp = (FFPlayer *)arg;
    int       ret = 0;

    if (ffp->node_vdec) {
        ret = ffpipenode_run_sync(ffp->node_vdec);
    }
    return ret;
}
```

#### 同步解码

前面讲到mediacodec_sync设置，同步情况下就启动func_run_sync_loop方法：

```C
static int func_run_sync_loop(IJKFF_Pipenode *node) {
    JNIEnv                *env           = NULL;
    IJKFF_Pipenode_Opaque *opaque        = node->opaque;
    FFPlayer              *ffp           = opaque->ffp;
    VideoState            *is            = ffp->is;
    Decoder               *d             = &is->viddec;
    PacketQueue           *q             = d->queue;
    int                    ret           = 0;
    int                    dequeue_count = 0;
    int                    enqueue_count = 0;
    AVFrame               *frame         = NULL;
    AVRational             frame_rate    = av_guess_frame_rate(is->ic, is->video_st, NULL);

    if (!opaque->acodec) {
        return ffp_video_thread(ffp);
    }

    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("%s: SetupThreadEnv failed\n", __func__);
        return -1;
    }

    frame = av_frame_alloc();
    if (!frame)
        goto fail;
        
    //关键代码
    while (!q->abort_request) {
        ret = drain_output_buffer2(env, node, AMC_SYNC_OUTPUT_TIMEOUT_US, &dequeue_count, frame, frame_rate);
        ret = feed_input_buffer2(env, node, AMC_SYNC_INPUT_TIMEOUT_US, &enqueue_count);
    }

fail:
    av_frame_free(&frame);
    opaque->abort = true;
    if (opaque->n_buf_out) {
        free(opaque->amc_buf_out);
        opaque->n_buf_out = 0;
        opaque->amc_buf_out = NULL;
        opaque->off_buf_out = 0;
        opaque->last_queued_pts = AV_NOPTS_VALUE;
    }
    if (opaque->acodec) {
        SDL_VoutAndroid_invalidateAllBuffers(opaque->weak_vout);
    }
    SDL_AMediaCodec_stop(opaque->acodec);
    SDL_AMediaCodec_decreaseReferenceP(&opaque->acodec);
    ALOGI("MediaCodec: %s: exit: %d", __func__, ret);
    return ret;
}
```
关键代码在while循环中，同步进行获取输出，填充输入。

看feed_input_buffer2方法：

```C
static int feed_input_buffer2(JNIEnv *env, IJKFF_Pipenode *node, int64_t timeUs, int *enqueue_count)
{
    IJKFF_Pipenode_Opaque *opaque   = node->opaque;
    FFPlayer              *ffp      = opaque->ffp;
    IJKFF_Pipeline        *pipeline = opaque->pipeline;
    VideoState            *is       = ffp->is;
    Decoder               *d        = &is->viddec;
    PacketQueue           *q        = d->queue;
    sdl_amedia_status_t    amc_ret  = 0;
    int                    ret      = 0;
    ssize_t  input_buffer_index     = 0;
    ssize_t  copy_size              = 0;
    int64_t  time_stamp             = 0;
    uint32_t queue_flags            = 0;

    if (enqueue_count)
        *enqueue_count = 0;

    if (d->queue->abort_request) {
        ret = ACODEC_EXIT;
        goto fail;
    }

    if (!d->packet_pending || d->queue->serial != d->pkt_serial) {
#if AMC_USE_AVBITSTREAM_FILTER
#else
        H264ConvertState convert_state = {0, 0};
#endif
        //数据源
        AVPacket pkt;
        do {
            if (d->queue->nb_packets == 0)
                SDL_CondSignal(d->empty_queue_cond);
            //从队列获取数据
            if (ffp_packet_queue_get_or_buffering(ffp, d->queue, &pkt, &d->pkt_serial, &d->finished) < 0) {
                ret = -1;
                goto fail;
            }
            if (ffp_is_flush_packet(&pkt) || opaque->acodec_flush_request) {
                // request flush before lock, or never get mutex
                opaque->acodec_flush_request = true;
                if (SDL_AMediaCodec_isStarted(opaque->acodec)) {
                    if (opaque->input_packet_count > 0) {
                        // flush empty queue cause error on OMX.SEC.AVC.Decoder (Nexus S)
                        SDL_VoutAndroid_invalidateAllBuffers(opaque->weak_vout);
                        SDL_AMediaCodec_flush(opaque->acodec);
                        opaque->input_packet_count = 0;
                    }
                    // If codec is configured in synchronous mode, codec will resume automatically
                    // SDL_AMediaCodec_start(opaque->acodec);
                }
                opaque->acodec_flush_request = false;
                d->finished = 0;
                d->next_pts = d->start_pts;
                d->next_pts_tb = d->start_pts_tb;
            }
        } while (ffp_is_flush_packet(&pkt) || d->queue->serial != d->pkt_serial);
        av_packet_split_side_data(&pkt);
        av_packet_unref(&d->pkt);
        d->pkt_temp = d->pkt = pkt;
        d->packet_pending = 1;

        //ffp的配置之一，通过ffmpeg，处理大小变化
        if (opaque->ffp->mediacodec_handle_resolution_change &&
            opaque->codecpar->codec_id == AV_CODEC_ID_H264) {
            uint8_t  *size_data      = NULL;
            int       size_data_size = 0;
            AVPacket *avpkt          = &d->pkt_temp;
            size_data = av_packet_get_side_data(avpkt, AV_PKT_DATA_NEW_EXTRADATA, &size_data_size);
            // minimum avcC(sps,pps) = 7
            if (size_data && size_data_size >= 7) {
                int             got_picture = 0;
                AVFrame        *frame      = av_frame_alloc();
                AVDictionary   *codec_opts = NULL;
                const AVCodec  *codec      = opaque->decoder->avctx->codec;
                AVCodecContext *new_avctx  = avcodec_alloc_context3(codec);
                int change_ret = 0;

                if (!new_avctx)
                    return AVERROR(ENOMEM);

                avcodec_parameters_to_context(new_avctx, opaque->codecpar);
                av_freep(&new_avctx->extradata);
                new_avctx->extradata = av_mallocz(size_data_size + AV_INPUT_BUFFER_PADDING_SIZE);
                if (!new_avctx->extradata) {
                    avcodec_free_context(&new_avctx);
                    return AVERROR(ENOMEM);
                }
                memcpy(new_avctx->extradata, size_data, size_data_size);
                new_avctx->extradata_size = size_data_size;

                av_dict_set(&codec_opts, "threads", "1", 0);
                change_ret = avcodec_open2(new_avctx, codec, &codec_opts);
                av_dict_free(&codec_opts);
                if (change_ret < 0) {
                    avcodec_free_context(&new_avctx);
                    return change_ret;
                }

                change_ret = avcodec_decode_video2(new_avctx, frame, &got_picture, avpkt);
                if (change_ret < 0) {
                    avcodec_free_context(&new_avctx);
                    return change_ret;
                } else {
                    if (opaque->codecpar->width  != new_avctx->width &&
                        opaque->codecpar->height != new_avctx->height) {
                        ALOGW("AV_PKT_DATA_NEW_EXTRADATA: %d x %d\n", new_avctx->width, new_avctx->height);
                        avcodec_parameters_from_context(opaque->codecpar, new_avctx);
                        opaque->aformat_need_recreate = true;
                        ffpipeline_set_surface_need_reconfigure_l(pipeline, true);
                    }
                }

                av_frame_unref(frame);
                avcodec_free_context(&new_avctx);
            }
        }
        
        ...

        queue_flags = 0;
        //获取输入缓存队列
        input_buffer_index = SDL_AMediaCodec_dequeueInputBuffer(opaque->acodec, timeUs);
        if (input_buffer_index < 0) {
            if (SDL_AMediaCodec_isInputBuffersValid(opaque->acodec)) {
                // timeout
                ret = 0;
                goto fail;
            } else {
                // enqueue fake frame
                queue_flags |= AMEDIACODEC__BUFFER_FLAG_FAKE_FRAME;
                copy_size    = d->pkt_temp.size;
            }
        } else {
            SDL_AMediaCodecFake_flushFakeFrames(opaque->acodec);
            //写数据
            copy_size = SDL_AMediaCodec_writeInputData(opaque->acodec, input_buffer_index, d->pkt_temp.data, d->pkt_temp.size);
            if (!copy_size) {
                ALOGE("%s: SDL_AMediaCodec_getInputBuffer failed\n", __func__);
                ret = -1;
                goto fail;
            }
        }

        time_stamp = d->pkt_temp.pts;
        if (time_stamp == AV_NOPTS_VALUE && d->pkt_temp.dts != AV_NOPTS_VALUE)
            time_stamp = d->pkt_temp.dts;
        if (time_stamp >= 0) {
            time_stamp = av_rescale_q(time_stamp, is->video_st->time_base, AV_TIME_BASE_Q);
        } else {
            time_stamp = 0;
        }
        // ALOGE("queueInputBuffer, %lld\n", time_stamp);
        //buffer入队列
        amc_ret = SDL_AMediaCodec_queueInputBuffer(opaque->acodec, input_buffer_index, 0, copy_size, time_stamp, queue_flags);
        if (amc_ret != SDL_AMEDIA_OK) {
            ALOGE("%s: SDL_AMediaCodec_getInputBuffer failed\n", __func__);
            ret = -1;
            goto fail;
        }
        // ALOGE("%s: queue %d/%d", __func__, (int)copy_size, (int)input_buffer_size);
        opaque->input_packet_count++;
        if (enqueue_count)
            ++*enqueue_count;
    }

    if (copy_size < 0) {
        d->packet_pending = 0;
    } else {
        d->pkt_temp.dts =
        d->pkt_temp.pts = AV_NOPTS_VALUE;
        if (d->pkt_temp.data) {
            d->pkt_temp.data += copy_size;
            d->pkt_temp.size -= copy_size;
            if (d->pkt_temp.size <= 0)
                d->packet_pending = 0;
        } else {
            // FIXME: detect if decode finished
            // if (!got_frame) {
                d->packet_pending = 0;
                d->finished = d->pkt_serial;
            // }
        }
    }

fail:
    return ret;
}
```
了解MeidiaCodec就清楚输入流程了，看几个关键步骤就非常明了。
这里数据源是从ffp_packet_queue_get_or_buffering获取的，数据读取还是通过ffmpeg的avformat，没有通过Android的MediaExtractor，只是解码使用了mediacodec。
再看输出方法drain_output_buffer2：
```C
static int drain_output_buffer2(JNIEnv *env, IJKFF_Pipenode *node, int64_t timeUs, int *dequeue_count, AVFrame *frame, AVRational frame_rate)
{
    IJKFF_Pipenode_Opaque *opaque    = node->opaque;
    FFPlayer              *ffp       = opaque->ffp;
    VideoState            *is        = ffp->is;
    AVRational            tb         = is->video_st->time_base;
    int                   got_frame  = 0;
    int                   ret        = -1;
    double                duration;
    double                pts;
    while (ret) {
        got_frame = 0;
        //真正获取数据方法，数据在frame，是否获取是got_frame
        ret = drain_output_buffer2_l(env, node, timeUs, dequeue_count, frame, &got_frame);

        if (opaque->decoder->queue->abort_request) {
            if (got_frame && frame->opaque)
                SDL_VoutAndroid_releaseBufferProxyP(opaque->weak_vout, (SDL_AMediaCodecBufferProxy **)&frame->opaque, false);

            return ACODEC_EXIT;
        }

        if (ret != 0) {
            if (got_frame && frame->opaque)
                SDL_VoutAndroid_releaseBufferProxyP(opaque->weak_vout, (SDL_AMediaCodecBufferProxy **)&frame->opaque, false);
        }
    }

    if (got_frame) {
        duration = (frame_rate.num && frame_rate.den ? av_q2d((AVRational){frame_rate.den, frame_rate.num}) : 0);
        pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
        if (ffp->framedrop > 0 || (ffp->framedrop && ffp_get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) {
            ffp->stat.decode_frame_count++;
            if (frame->pts != AV_NOPTS_VALUE) {
                double dpts = pts;
                double diff = dpts - ffp_get_master_clock(is);
                if (!isnan(diff) && fabs(diff) < AV_NOSYNC_THRESHOLD &&
                    diff - is->frame_last_filter_delay < 0 &&
                    is->viddec.pkt_serial == is->vidclk.serial &&
                    is->videoq.nb_packets) {
                    is->frame_drops_early++;
                    is->continuous_frame_drops_early++;
                    if (is->continuous_frame_drops_early > ffp->framedrop) {
                        is->continuous_frame_drops_early = 0;
                    } else {
                        ffp->stat.drop_frame_count++;
                        ffp->stat.drop_frame_rate = (float)(ffp->stat.drop_frame_count) / (float)(ffp->stat.decode_frame_count);
                        if (frame->opaque) {
                            SDL_VoutAndroid_releaseBufferProxyP(opaque->weak_vout, (SDL_AMediaCodecBufferProxy **)&frame->opaque, false);
                        }
                        av_frame_unref(frame);
                        return ret;
                    }
                }
            }
        }
        ret = ffp_queue_picture(ffp, frame, pts, duration, av_frame_get_pkt_pos(frame), is->viddec.pkt_serial);
        if (ret) {
            if (frame->opaque)
                SDL_VoutAndroid_releaseBufferProxyP(opaque->weak_vout, (SDL_AMediaCodecBufferProxy **)&frame->opaque, false);
        }
        av_frame_unref(frame);
    }

    return ret;
}
```

获取到frame后通过ffp_queue_picture去显示，看输出真正逻辑所在drain_output_buffer2_l方法：
```C
static int drain_output_buffer2_l(JNIEnv *env, IJKFF_Pipenode *node, int64_t timeUs, int *dequeue_count, AVFrame *frame, int *got_frame)
{
    IJKFF_Pipenode_Opaque *opaque         = node->opaque;
    FFPlayer              *ffp            = opaque->ffp;
    SDL_AMediaCodecBufferInfo bufferInfo;
    ssize_t          output_buffer_index  = 0;

    if (dequeue_count)
        *dequeue_count = 0;

    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("%s:create: SetupThreadEnv failed\n", __func__);
        return ACODEC_RETRY;
    }

    //获取输出buffer索引，然后是熟悉的返回值判断
    output_buffer_index = SDL_AMediaCodecFake_dequeueOutputBuffer(opaque->acodec, &bufferInfo, timeUs);
    if (output_buffer_index == AMEDIACODEC__INFO_OUTPUT_BUFFERS_CHANGED) {
        ALOGD("AMEDIACODEC__INFO_OUTPUT_BUFFERS_CHANGED\n");
        return ACODEC_RETRY;
    } else if (output_buffer_index == AMEDIACODEC__INFO_OUTPUT_FORMAT_CHANGED) {
        ALOGD("AMEDIACODEC__INFO_OUTPUT_FORMAT_CHANGED\n");
        SDL_AMediaFormat_deleteP(&opaque->output_aformat);
        opaque->output_aformat = SDL_AMediaCodec_getOutputFormat(opaque->acodec);
        if (opaque->output_aformat) {
            int width        = 0;
            int height       = 0;
            int color_format = 0;
            int stride       = 0;
            int slice_height = 0;
            int crop_left    = 0;
            int crop_top     = 0;
            int crop_right   = 0;
            int crop_bottom  = 0;

            SDL_AMediaFormat_getInt32(opaque->output_aformat, "width",          &width);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "height",         &height);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "color-format",   &color_format);

            SDL_AMediaFormat_getInt32(opaque->output_aformat, "stride",         &stride);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "slice-height",   &slice_height);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "crop-left",      &crop_left);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "crop-top",       &crop_top);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "crop-right",     &crop_right);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "crop-bottom",    &crop_bottom);

            // TI decoder could crash after reconfigure
            // ffp_notify_msg3(ffp, FFP_MSG_VIDEO_SIZE_CHANGED, width, height);
            // opaque->frame_width  = width;
            // opaque->frame_height = height;
            ALOGI(
                "AMEDIACODEC__INFO_OUTPUT_FORMAT_CHANGED\n"
                "    width-height: (%d x %d)\n"
                "    color-format: (%s: 0x%x)\n"
                "    stride:       (%d)\n"
                "    slice-height: (%d)\n"
                "    crop:         (%d, %d, %d, %d)\n"
                ,
                width, height,
                SDL_AMediaCodec_getColorFormatName(color_format), color_format,
                stride,
                slice_height,
                crop_left, crop_top, crop_right, crop_bottom);
        }
        return ACODEC_RETRY;
        // continue;
    } else if (output_buffer_index == AMEDIACODEC__INFO_TRY_AGAIN_LATER) {
        return 0;
        // continue;
    } else if (output_buffer_index < 0) {
        return 0;
    } else if (output_buffer_index >= 0) {
        ffp->stat.vdps = SDL_SpeedSamplerAdd(&opaque->sampler, FFP_SHOW_VDPS_MEDIACODEC, "vdps[MediaCodec]");
        
        //成功后计数加1
        if (dequeue_count)
            ++*dequeue_count;

        if (opaque->n_buf_out) {
            AMC_Buf_Out *buf_out;
            if (opaque->off_buf_out < opaque->n_buf_out) {
                // ALOGD("filling buffer... %d", opaque->off_buf_out);
                buf_out = &opaque->amc_buf_out[opaque->off_buf_out++];
                buf_out->acodec_serial = SDL_AMediaCodec_getSerial(opaque->acodec);
                buf_out->port = output_buffer_index;
                buf_out->info = bufferInfo;
                buf_out->pts = pts_from_buffer_info(node, &bufferInfo);
                sort_amc_buf_out(opaque->amc_buf_out, opaque->off_buf_out);
            } else {
                double pts;

                pts = pts_from_buffer_info(node, &bufferInfo);
                if (opaque->last_queued_pts != AV_NOPTS_VALUE &&
                    pts < opaque->last_queued_pts) {
                    // FIXME: drop unordered picture to avoid dither
                    // ALOGE("early picture, drop!");
                    // SDL_AMediaCodec_releaseOutputBuffer(opaque->acodec, output_buffer_index, false);
                    // goto done;
                }
                /* already sorted */
                buf_out = &opaque->amc_buf_out[opaque->off_buf_out - 1];
                /* new picture is the most aged, send now */
                if (pts < buf_out->pts) {
                    //关键填充frame方法，多种条件触发
                    amc_fill_frame(node, frame, got_frame, output_buffer_index, SDL_AMediaCodec_getSerial(opaque->acodec), &bufferInfo);
                    opaque->last_queued_pts = pts;
                    // ALOGD("pts = %f", pts);
                } else {
                    int i;

                    /* find one to send */
                    for (i = opaque->off_buf_out - 1; i >= 0; i--) {
                        buf_out = &opaque->amc_buf_out[i];
                        if (pts > buf_out->pts) {
                            amc_fill_frame(node, frame, got_frame, buf_out->port, buf_out->acodec_serial, &buf_out->info);
                            opaque->last_queued_pts = buf_out->pts;
                            // ALOGD("pts = %f", buf_out->pts);
                            /* replace for sort later */
                            buf_out->acodec_serial = SDL_AMediaCodec_getSerial(opaque->acodec);
                            buf_out->port = output_buffer_index;
                            buf_out->info = bufferInfo;
                            buf_out->pts = pts_from_buffer_info(node, &bufferInfo);
                            sort_amc_buf_out(opaque->amc_buf_out, opaque->n_buf_out);
                            break;
                        }
                    }
                    /* need to discard current buffer */
                    if (i < 0) {
                        // ALOGE("buffer too small, drop picture!");
                        if (!(bufferInfo.flags & AMEDIACODEC__BUFFER_FLAG_FAKE_FRAME)) {
                            SDL_AMediaCodec_releaseOutputBuffer(opaque->acodec, output_buffer_index, false);
                            return 0;
                        }
                    }
                }
            }
        } else {
            amc_fill_frame(node, frame, got_frame, output_buffer_index, SDL_AMediaCodec_getSerial(opaque->acodec), &bufferInfo);
        }
    }

    return 0;
}
```

可以看到MediaCodec操作input的关键流程，最后通过amc_fill_frame填充frame，把数据交回ffplay进行显示。
继续看amc_fill_frame代码：

```C
static int amc_fill_frame(
    IJKFF_Pipenode            *node,
    AVFrame                   *frame,
    int                       *got_frame,
    int                        output_buffer_index,
    int                        acodec_serial,
    SDL_AMediaCodecBufferInfo *buffer_info)
{
    IJKFF_Pipenode_Opaque *opaque     = node->opaque;
    FFPlayer              *ffp        = opaque->ffp;
    VideoState            *is         = ffp->is;

    frame->opaque = SDL_VoutAndroid_obtainBufferProxy(opaque->weak_vout, acodec_serial, output_buffer_index, buffer_info);
    if (!frame->opaque)
        goto fail;

    frame->width  = opaque->frame_width;
    frame->height = opaque->frame_height;
    frame->format = IJK_AV_PIX_FMT__ANDROID_MEDIACODEC;
    frame->sample_aspect_ratio = opaque->codecpar->sample_aspect_ratio;
    frame->pts    = av_rescale_q(buffer_info->presentationTimeUs, AV_TIME_BASE_Q, is->video_st->time_base);
    if (frame->pts < 0)
        frame->pts = AV_NOPTS_VALUE;
    // ALOGE("%s: %f", __func__, (float)frame->pts);

    *got_frame = 1;
    return 0;
fail:
    *got_frame = 0;
    return -1;
}
```
IJKFF_Pipenode_Opaque是解码结构体，用于输出和显示，在前面初始化IJKFF_Pipenode时也被创建。显示部分就不再分析。

#### 异步解码

mediacodec_sync设置为0时开启异步解码，也是默认模式。看func_run_sync线程：
```C
static int func_run_sync(IJKFF_Pipenode *node)
{
    JNIEnv                *env      = NULL;
    IJKFF_Pipenode_Opaque *opaque   = node->opaque;
    FFPlayer              *ffp      = opaque->ffp;
    VideoState            *is       = ffp->is;
    Decoder               *d        = &is->viddec;
    PacketQueue           *q        = d->queue;
    int                    ret      = 0;
    int                    dequeue_count = 0;
    AVFrame               *frame    = NULL;
    int                    got_frame = 0;
    AVRational             tb         = is->video_st->time_base;
    AVRational             frame_rate = av_guess_frame_rate(is->ic, is->video_st, NULL);
    double                 duration;
    double                 pts;

    //再次容错切换软解
    if (!opaque->acodec) {
        return ffp_video_thread(ffp);
    }

    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("%s: SetupThreadEnv failed\n", __func__);
        return -1;
    }

    frame = av_frame_alloc();
    if (!frame)
        goto fail;
    
    //通过opaque开启input队列线程
    opaque->enqueue_thread = SDL_CreateThreadEx(&opaque->_enqueue_thread, enqueue_thread_func, node, "amediacodec_input_thread");
    if (!opaque->enqueue_thread) {
        ALOGE("%s: SDL_CreateThreadEx failed\n", __func__);
        ret = -1;
        goto fail;
    }

    while (!q->abort_request) {
        int64_t timeUs = opaque->acodec_first_dequeue_output_request ? 0 : AMC_OUTPUT_TIMEOUT_US;
        got_frame = 0;
        //获取输出方法
        ret = drain_output_buffer(env, node, timeUs, &dequeue_count, frame, &got_frame);
        if (opaque->acodec_first_dequeue_output_request) {
            SDL_LockMutex(opaque->acodec_first_dequeue_output_mutex);
            opaque->acodec_first_dequeue_output_request = false;
            SDL_CondSignal(opaque->acodec_first_dequeue_output_cond);
            SDL_UnlockMutex(opaque->acodec_first_dequeue_output_mutex);
        }
        if (ret != 0) {
            ret = -1;
            if (got_frame && frame->opaque)
                SDL_VoutAndroid_releaseBufferProxyP(opaque->weak_vout, (SDL_AMediaCodecBufferProxy **)&frame->opaque, false);
            goto fail;
        }
        //获取到帧，和同步逻辑一样
        if (got_frame) {
            duration = (frame_rate.num && frame_rate.den ? av_q2d((AVRational){frame_rate.den, frame_rate.num}) : 0);
            pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
            if (ffp->framedrop > 0 || (ffp->framedrop && ffp_get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) {
                ffp->stat.decode_frame_count++;
                if (frame->pts != AV_NOPTS_VALUE) {
                    double dpts = pts;
                    double diff = dpts - ffp_get_master_clock(is);
                    if (!isnan(diff) && fabs(diff) < AV_NOSYNC_THRESHOLD &&
                        diff - is->frame_last_filter_delay < 0 &&
                        is->viddec.pkt_serial == is->vidclk.serial &&
                        is->videoq.nb_packets) {
                        is->frame_drops_early++;
                        is->continuous_frame_drops_early++;
                        if (is->continuous_frame_drops_early > ffp->framedrop) {
                            is->continuous_frame_drops_early = 0;
                        } else {
                            ffp->stat.drop_frame_count++;
                            ffp->stat.drop_frame_rate = (float)(ffp->stat.drop_frame_count) / (float)(ffp->stat.decode_frame_count);
                            if (frame->opaque) {
                                SDL_VoutAndroid_releaseBufferProxyP(opaque->weak_vout, (SDL_AMediaCodecBufferProxy **)&frame->opaque, false);
                            }
                            av_frame_unref(frame);
                            continue;
                        }
                    }
                }
            }
            //推送显示
            ret = ffp_queue_picture(ffp, frame, pts, duration, av_frame_get_pkt_pos(frame), is->viddec.pkt_serial);
            if (ret) {
                if (frame->opaque)
                    SDL_VoutAndroid_releaseBufferProxyP(opaque->weak_vout, (SDL_AMediaCodecBufferProxy **)&frame->opaque, false);
            }
            av_frame_unref(frame);
        }
    }

fail:
    av_frame_free(&frame);
    opaque->abort = true;
    SDL_WaitThread(opaque->enqueue_thread, NULL);
    SDL_AMediaCodecFake_abort(opaque->acodec);
    if (opaque->n_buf_out) {
        free(opaque->amc_buf_out);
        opaque->n_buf_out = 0;
        opaque->amc_buf_out = NULL;
        opaque->off_buf_out = 0;
        opaque->last_queued_pts = AV_NOPTS_VALUE;
    }
    if (opaque->acodec) {
        SDL_VoutAndroid_invalidateAllBuffers(opaque->weak_vout);
        SDL_LockMutex(opaque->acodec_mutex);
        SDL_UnlockMutex(opaque->acodec_mutex);
    }
    SDL_AMediaCodec_stop(opaque->acodec);
    SDL_AMediaCodec_decreaseReferenceP(&opaque->acodec);
    ALOGI("MediaCodec: %s: exit: %d", __func__, ret);
    return ret;
#if 0
fallback_to_ffplay:
    ALOGW("fallback to ffplay decoder\n");
    return ffp_video_thread(opaque->ffp);
#endif
}
```

异步解码单独开启了输入的线程，输入输出分离，增加吞吐效率。
看输出关键方法drain_output_buffer：

```C
static int drain_output_buffer(JNIEnv *env, IJKFF_Pipenode *node, int64_t timeUs, int *dequeue_count, AVFrame *frame, int *got_frame)
{
    IJKFF_Pipenode_Opaque *opaque = node->opaque;
    SDL_LockMutex(opaque->acodec_mutex);

    if (opaque->acodec_flush_request || opaque->acodec_reconfigure_request) {
        // TODO: invalid picture here?
        // let feed_input_buffer() get mutex
        SDL_CondWaitTimeout(opaque->acodec_cond, opaque->acodec_mutex, 100);
    }

    int ret = drain_output_buffer_l(env, node, timeUs, dequeue_count, frame, got_frame);
    SDL_UnlockMutex(opaque->acodec_mutex);
    return ret;
}
```
等待输出，然后调用核心方法drain_output_buffer_l：
```C
static int drain_output_buffer_l(JNIEnv *env, IJKFF_Pipenode *node, int64_t timeUs, int *dequeue_count, AVFrame *frame, int *got_frame)
{
    IJKFF_Pipenode_Opaque *opaque   = node->opaque;
    FFPlayer              *ffp      = opaque->ffp;
    int                    ret      = 0;
    SDL_AMediaCodecBufferInfo bufferInfo;
    ssize_t                   output_buffer_index = 0;

    if (dequeue_count)
        *dequeue_count = 0;

    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("%s:create: SetupThreadEnv failed\n", __func__);
        goto fail;
    }

    output_buffer_index = SDL_AMediaCodecFake_dequeueOutputBuffer(opaque->acodec, &bufferInfo, timeUs);
    if (output_buffer_index == AMEDIACODEC__INFO_OUTPUT_BUFFERS_CHANGED) {
        ALOGI("AMEDIACODEC__INFO_OUTPUT_BUFFERS_CHANGED\n");
        // continue;
    } else if (output_buffer_index == AMEDIACODEC__INFO_OUTPUT_FORMAT_CHANGED) {
        ALOGI("AMEDIACODEC__INFO_OUTPUT_FORMAT_CHANGED\n");
        SDL_AMediaFormat_deleteP(&opaque->output_aformat);
        opaque->output_aformat = SDL_AMediaCodec_getOutputFormat(opaque->acodec);
        if (opaque->output_aformat) {
            int width        = 0;
            int height       = 0;
            int color_format = 0;
            int stride       = 0;
            int slice_height = 0;
            int crop_left    = 0;
            int crop_top     = 0;
            int crop_right   = 0;
            int crop_bottom  = 0;

            SDL_AMediaFormat_getInt32(opaque->output_aformat, "width",          &width);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "height",         &height);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "color-format",   &color_format);

            SDL_AMediaFormat_getInt32(opaque->output_aformat, "stride",         &stride);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "slice-height",   &slice_height);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "crop-left",      &crop_left);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "crop-top",       &crop_top);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "crop-right",     &crop_right);
            SDL_AMediaFormat_getInt32(opaque->output_aformat, "crop-bottom",    &crop_bottom);

            // TI decoder could crash after reconfigure
            // ffp_notify_msg3(ffp, FFP_MSG_VIDEO_SIZE_CHANGED, width, height);
            // opaque->frame_width  = width;
            // opaque->frame_height = height;
            ALOGI(
                "AMEDIACODEC__INFO_OUTPUT_FORMAT_CHANGED\n"
                "    width-height: (%d x %d)\n"
                "    color-format: (%s: 0x%x)\n"
                "    stride:       (%d)\n"
                "    slice-height: (%d)\n"
                "    crop:         (%d, %d, %d, %d)\n"
                ,
                width, height,
                SDL_AMediaCodec_getColorFormatName(color_format), color_format,
                stride,
                slice_height,
                crop_left, crop_top, crop_right, crop_bottom);
        }
        // continue;
    } else if (output_buffer_index == AMEDIACODEC__INFO_TRY_AGAIN_LATER) {
        AMCTRACE("AMEDIACODEC__INFO_TRY_AGAIN_LATER\n");
        // continue;
    } else if (output_buffer_index < 0) {
        //没有输出时进行等待输入同步信号
        SDL_LockMutex(opaque->any_input_mutex);
        SDL_CondWaitTimeout(opaque->any_input_cond, opaque->any_input_mutex, 1000);
        SDL_UnlockMutex(opaque->any_input_mutex);

        goto done;
    } else if (output_buffer_index >= 0) {
        ffp->stat.vdps = SDL_SpeedSamplerAdd(&opaque->sampler, FFP_SHOW_VDPS_MEDIACODEC, "vdps[MediaCodec]");

        if (dequeue_count)
            ++*dequeue_count;

#ifdef FFP_SHOW_AMC_VDPS
        {
            if (opaque->benchmark_start_time == 0) {
                opaque->benchmark_start_time   = SDL_GetTickHR();
            }
            opaque->benchmark_frame_count += 1;
            if (0 == (opaque->benchmark_frame_count % 240)) {
                Uint64 diff = SDL_GetTickHR() - opaque->benchmark_start_time;
                double per_frame_ms = ((double) diff) / opaque->benchmark_frame_count;
                double fps          = ((double) opaque->benchmark_frame_count) * 1000 / diff;
                ALOGE("%lf fps, %lf ms/frame, %"PRIu64" frames\n",
                      fps, per_frame_ms, opaque->benchmark_frame_count);
            }
        }
#endif
#ifdef FFP_AMC_DISABLE_OUTPUT
        if (!(bufferInfo.flags & AMEDIACODEC__BUFFER_FLAG_FAKE_FRAME)) {
            SDL_AMediaCodec_releaseOutputBuffer(opaque->acodec, output_buffer_index, false);
        }
        goto done;
#endif

        if (opaque->n_buf_out) {
            AMC_Buf_Out *buf_out;

            if (opaque->off_buf_out < opaque->n_buf_out) {
                // ALOGD("filling buffer... %d", opaque->off_buf_out);
                buf_out = &opaque->amc_buf_out[opaque->off_buf_out++];
                buf_out->acodec_serial = SDL_AMediaCodec_getSerial(opaque->acodec);
                buf_out->port = output_buffer_index;
                buf_out->info = bufferInfo;
                buf_out->pts = pts_from_buffer_info(node, &bufferInfo);
                sort_amc_buf_out(opaque->amc_buf_out, opaque->off_buf_out);
            } else {
                double pts;

                pts = pts_from_buffer_info(node, &bufferInfo);
                if (opaque->last_queued_pts != AV_NOPTS_VALUE &&
                    pts < opaque->last_queued_pts) {
                    // FIXME: drop unordered picture to avoid dither
                    // ALOGE("early picture, drop!");
                    // SDL_AMediaCodec_releaseOutputBuffer(opaque->acodec, output_buffer_index, false);
                    // goto done;
                }
                /* already sorted */
                buf_out = &opaque->amc_buf_out[opaque->off_buf_out - 1];
                /* new picture is the most aged, send now */
                if (pts < buf_out->pts) {
                    ret = amc_fill_frame(node, frame, got_frame, output_buffer_index, SDL_AMediaCodec_getSerial(opaque->acodec), &bufferInfo);
                    opaque->last_queued_pts = pts;
                    // ALOGD("pts = %f", pts);
                } else {
                    int i;

                    /* find one to send */
                    for (i = opaque->off_buf_out - 1; i >= 0; i--) {
                        buf_out = &opaque->amc_buf_out[i];
                        if (pts > buf_out->pts) {
                            ret = amc_fill_frame(node, frame, got_frame, buf_out->port, buf_out->acodec_serial, &buf_out->info);
                            opaque->last_queued_pts = buf_out->pts;
                            // ALOGD("pts = %f", buf_out->pts);
                            /* replace for sort later */
                            buf_out->acodec_serial = SDL_AMediaCodec_getSerial(opaque->acodec);
                            buf_out->port = output_buffer_index;
                            buf_out->info = bufferInfo;
                            buf_out->pts = pts_from_buffer_info(node, &bufferInfo);
                            sort_amc_buf_out(opaque->amc_buf_out, opaque->n_buf_out);
                            break;
                        }
                    }
                    /* need to discard current buffer */
                    if (i < 0) {
                        // ALOGE("buffer too small, drop picture!");
                        if (!(bufferInfo.flags & AMEDIACODEC__BUFFER_FLAG_FAKE_FRAME)) {
                            SDL_AMediaCodec_releaseOutputBuffer(opaque->acodec, output_buffer_index, false);
                            goto done;
                        }
                    }
                }
            }
        } else {
            ret = amc_fill_frame(node, frame, got_frame, output_buffer_index, SDL_AMediaCodec_getSerial(opaque->acodec), &bufferInfo);
        }
    }

done:
    if (opaque->decoder->queue->abort_request)
        ret = -1;
    else
        ret = 0;
fail:
    return ret;
}
```

drain_output_buffer_l流程和drain_output_buffer2_l和差不多一样。
异步请求drain_output_buffer_l失败时会等待输入，
同步请求drain_output_buffer2_l失败时直接返回ACODEC_RETRY，直接失败。



## 总结

前面已经把解码流程分析完毕，从option设置，到ffmPlayer初始化,再到解码器初始化，再分析到同步和异步解码。整体解码流程和管道使用都熟悉了。SDL的AMediaCodec方法如何调用到java的AMediaCodec还没有分析，这个留待下篇再分析。



