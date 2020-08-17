---
title:      "ijkPlayer深入探究(二)"
description:   "ijkSDL深入解析:Android硬解流程分析"
date:       2020-06-25 14:00:00
author:     "安地"
tags:
      - Android
      - C
      - 音视频
---


## 前言

上一篇从上到下分析了ijkplayer解码的流程，最后用到SDL。
SDL是Simple DirectMedia Layer，提供简单的方式支持跨平台访问输出库，ijksdl思路是和SDL一样的，ijksdl用于音视频渲染输出。


## 正文


### 创建

我们先看IjkMediaPlayer创建的方法：

``` 
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

这里创建IjkMediaPlayer后，又创建了一个vout(视频显示用的结构体)和pipeline(管线， 解码用的结构体)

这样使用统一接口，便于跨平台扩展。

### 结构体

vout是SDL_Vout的结构体，用于显示：
```C
struct SDL_Vout {
    SDL_mutex *mutex;

    SDL_Class       *opaque_class;
    SDL_Vout_Opaque *opaque;
    SDL_VoutOverlay *(*create_overlay)(int width, int height, int frame_format, SDL_Vout *vout);
    void (*free_l)(SDL_Vout *vout);
    int (*display_overlay)(SDL_Vout *vout, SDL_VoutOverlay *overlay);

    Uint32 overlay_format;
};
```

pipeline是IJKFF_Pipeline的结构体:
```C
struct IJKFF_Pipeline {
    SDL_Class             *opaque_class;
    IJKFF_Pipeline_Opaque *opaque;

    void            (*func_destroy)             (IJKFF_Pipeline *pipeline);
    IJKFF_Pipenode *(*func_open_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
    SDL_Aout       *(*func_open_audio_output)   (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
    IJKFF_Pipenode *(*func_init_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
    int           (*func_config_video_decoder)  (IJKFF_Pipeline *pipeline, FFPlayer *ffp);
};
```

管线中有管道节点IJKFF_Pipenode，即解码器:
```C
struct IJKFF_Pipenode {
    SDL_mutex *mutex;
    void *opaque;

    void (*func_destroy) (IJKFF_Pipenode *node);
    int  (*func_run_sync)(IJKFF_Pipenode *node);
    int  (*func_flush)   (IJKFF_Pipenode *node); // optional
};
```

IJKFF_Pipenode里有黑盒opaque,对应结构体IJKFF_Pipenode_Opaque，具体实现都在这个黑盒里，外面不用关心具体实现，只需要关注输入输出，但我们还是打开具体看看:

```
 typedef struct IJKFF_Pipenode_Opaque {
        FFPlayer                 *ffp;
        IJKFF_Pipeline           *pipeline;
        Decoder                  *decoder;
        SDL_Vout                 *weak_vout;

        ijkmp_mediacodecinfo_context mcc;

        jobject                   jsurface;
        SDL_AMediaFormat         *input_aformat;
        SDL_AMediaCodec          *acodec;
        SDL_AMediaFormat         *output_aformat;
        char                      acodec_name[128];
        int                       frame_width;
        int                       frame_height;
        int                       frame_rotate_degrees;

        ...
    } IJKFF_Pipenode_Opaque;
```

IJKFF_Pipenode_Opaque里对应AMedia对应的SDL封装结构体，SDL_AMediaFormat，SDL_AMediaCodec，SDL_AMediaFormat

### 解码流程

结合上一篇根据结构体调用分析解码流程：
1. 初始化完成后得到IJKFF_Pipeline和IJKFF_Pipenode，还有输出结构体SDL_Vout
2. 启动管道节点IJKFF_Pipenode的func_run_sync方法执行解码流程
3. 黑盒IJKFF_Pipenode_Opaque，调用SDL方法如SDL_AMediaCodec_dequeueInputBuffer(opaque->acodec)进行解码
4. SDL_VoutAndroid_releaseBufferProxyP方法传入SDL_Vout和IJKFF_Pipenode_Opaque进行输出


### SDL方法调用流程

我们以SDL_AMediaCodec_dequeueInputBuffer看SDL方法调用流程：

```C
ssize_t SDL_AMediaCodec_dequeueInputBuffer(SDL_AMediaCodec* acodec, int64_t timeoutUs)
{
    assert(acodec->func_dequeueInputBuffer);
    return acodec->func_dequeueInputBuffer(acodec, timeoutUs);
}
```

SDL_AMediaCodec等于是一个抽象接口，看其如何创建的：

```C
static SDL_AMediaCodec *create_codec_l(JNIEnv *env, IJKFF_Pipenode *node)
{
    IJKFF_Pipenode_Opaque        *opaque   = node->opaque;
    ijkmp_mediacodecinfo_context *mcc      = &opaque->mcc;
    SDL_AMediaCodec              *acodec   = NULL;

    if (opaque->jsurface == NULL) {
        // we don't need real codec if we don't have a surface
        acodec = SDL_AMediaCodecDummy_create();
    } else {
        acodec = SDL_AMediaCodecJava_createByCodecName(env, mcc->codec_name);
        if (acodec) {
            strncpy(opaque->acodec_name, mcc->codec_name, sizeof(opaque->acodec_name) / sizeof(*opaque->acodec_name));
            opaque->acodec_name[sizeof(opaque->acodec_name) / sizeof(*opaque->acodec_name) - 1] = 0;
        }
    }

#if 0
    if (!acodec)
        acodec = SDL_AMediaCodecJava_createDecoderByType(env, mcc->mime_type);
#endif

    if (acodec) {
        // QUIRK: always recreate MediaCodec for reconfigure
        opaque->quirk_reconfigure_with_new_codec = true;
        /*-
        if (0 == strncasecmp(mcc->codec_name, "OMX.TI.DUCATI1.", 15)) {
            opaque->quirk_reconfigure_with_new_codec = true;
        }
        */
        /* delaying output makes it possible to correct frame order, hopefully */
        if (0 == strncasecmp(mcc->codec_name, "OMX.TI.DUCATI1.", 15)) {
            /* this is the only acceptable value on Nexus S */
            opaque->n_buf_out = 1;
            ALOGD("using buffered output for %s", mcc->codec_name);
        }
    }

    if (opaque->frame_rotate_degrees == 90 || opaque->frame_rotate_degrees == 270) {
        opaque->frame_width  = opaque->codecpar->height;
        opaque->frame_height = opaque->codecpar->width;
    } else {
        opaque->frame_width  = opaque->codecpar->width;
        opaque->frame_height = opaque->codecpar->height;
    }

    return acodec;
}
```

这里有两种初始化方法SDL_AMediaCodecDummy_create和SDL_AMediaCodecJava_createByCodecName，SDL_AMediaCodecDummy_create是一个傀儡实现，用于容错的。
看SDL_AMediaCodecJava_createByCodecName方法：

```C
static SDL_AMediaCodec* SDL_AMediaCodecJava_init(JNIEnv *env, jobject android_media_codec)
{
    SDLTRACE("%s", __func__);

    jobject global_android_media_codec = (*env)->NewGlobalRef(env, android_media_codec);
    if (J4A_ExceptionCheck__catchAll(env) || !global_android_media_codec) {
        return NULL;
    }

    SDL_AMediaCodec *acodec = SDL_AMediaCodec_CreateInternal(sizeof(SDL_AMediaCodec_Opaque));
    if (!acodec) {
        SDL_JNI_DeleteGlobalRefP(env, &global_android_media_codec);
        return NULL;
    }

    SDL_AMediaCodec_Opaque *opaque = acodec->opaque;
    opaque->android_media_codec         = global_android_media_codec;

    acodec->opaque_class                = &g_amediacodec_class;
    acodec->func_delete                 = SDL_AMediaCodecJava_delete;
    acodec->func_configure              = NULL;
    acodec->func_configure_surface      = SDL_AMediaCodecJava_configure_surface;

    acodec->func_start                  = SDL_AMediaCodecJava_start;
    acodec->func_stop                   = SDL_AMediaCodecJava_stop;
    acodec->func_flush                  = SDL_AMediaCodecJava_flush;

    acodec->func_writeInputData         = SDL_AMediaCodecJava_writeInputData;

    acodec->func_dequeueInputBuffer     = SDL_AMediaCodecJava_dequeueInputBuffer;
    acodec->func_queueInputBuffer       = SDL_AMediaCodecJava_queueInputBuffer;

    acodec->func_dequeueOutputBuffer    = SDL_AMediaCodecJava_dequeueOutputBuffer;
    acodec->func_getOutputFormat        = SDL_AMediaCodecJava_getOutputFormat;
    acodec->func_releaseOutputBuffer    = SDL_AMediaCodecJava_releaseOutputBuffer;

    acodec->func_isInputBuffersValid    = SDL_AMediaCodecJava_isInputBuffersValid;

    SDL_AMediaCodec_increaseReference(acodec);
    return acodec;
}

SDL_AMediaCodec* SDL_AMediaCodecJava_createByCodecName(JNIEnv *env, const char *codec_name)
{
    SDLTRACE("%s", __func__);

    jobject android_media_codec = J4AC_MediaCodec__createByCodecName__withCString__catchAll(env, codec_name); //j4a方法创建解码器
    if (J4A_ExceptionCheck__catchAll(env) || !android_media_codec) {
        return NULL;
    }

    SDL_AMediaCodec* acodec = SDL_AMediaCodecJava_init(env, android_media_codec);
    acodec->object_serial = SDL_AMediaCodec_create_object_serial();
    SDL_JNI_DeleteLocalRefP(env, &android_media_codec);
    return acodec;
}

```

J4AC_MediaCodec__createByCodecName__withCString__catchAll用的是jni4android调用，其实就是正常的jni调用java方法，bilibili写了一个插件可以自动生成对应方法，手动写jni调用java太长了
想了解可以看项目主页： https://github.com/bilibili/jni4android

创建解码器后，初始化了SDL_AMediaCodec的结构体,赋值了对应方法，继续看SDL_AMediaCodecJava_dequeueInputBuffer方法：

```C
ssize_t SDL_AMediaCodecJava_dequeueInputBuffer(SDL_AMediaCodec* acodec, int64_t timeoutUs)
{
    AMCTRACE("%s(%d)", __func__, (int)timeoutUs);

    JNIEnv *env = NULL;
    if (JNI_OK != SDL_JNI_SetupThreadEnv(&env)) {
        ALOGE("%s: SetupThreadEnv failed", __func__);
        return -1;
    }

    SDL_AMediaCodec_Opaque *opaque = (SDL_AMediaCodec_Opaque *)acodec->opaque;
    // docs lie, getInputBuffers should be good after
    // m_codec->start() but the internal refs are not
    // setup until much later on some devices.
    //if (-1 == getInputBuffers(env, acodec)) {
    //    ALOGE("%s: getInputBuffers failed", __func__);
    //    return -1;
    //}

    jobject android_media_codec = opaque->android_media_codec;
    jint idx = J4AC_MediaCodec__dequeueInputBuffer(env, android_media_codec, (jlong)timeoutUs);
    if (J4A_ExceptionCheck__catchAll(env)) {
        ALOGE("%s: dequeueInputBuffer failed", __func__);
        opaque->is_input_buffer_valid = false;
        return -1;
        
    }

    return idx;
}
```

这里也是调用到j4a方法J4AC_MediaCodec__dequeueInputBuffer，这就是最终解码的实现方法了，通过j4a调用到java的MediaCodec对应方法。

## 总结

本文分析了ijksdk的方法调用过程，通过跨平台接口调用，在android上还是走到了Java层MediaCodec方法中。

