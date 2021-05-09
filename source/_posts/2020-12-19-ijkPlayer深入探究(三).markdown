---
title:      "ijkPlayer深入探究(三)"
description:   "ijkPlayer自定义demuxer（解封装器）"
date:       2020-10-19 14:00:00
author:     "安地"
tags:
      - Android
      - C
      - 音视频

---

# 背景

前面介绍了ijkplayer播放解码的流程还有音视频的参数，这篇从实际使用介绍如何自定义demuxer。


# 原理

ffmpeg有很多支持的protocol和demuxer，且支持自定义配置。ijkplayer又提供一套的protocol和demuxer的注册方式，可以不编译ffmpeg的情况下直接在ijkplayer里面添加。

看allformats.c，ijkav_register_all注册方法就能直接把所有自定义的protocol和demuxer注册进ffmpeg

```C

void ijkav_register_all(void)
{
    static int initialized;

    if (initialized)
        return;
    initialized = 1;

    av_register_all();

    /* protocols */
    av_log(NULL, AV_LOG_INFO, "===== custom modules begin =====\n");
#ifdef __ANDROID__
    IJK_REGISTER_PROTOCOL(ijkmediadatasource);
#endif
    IJK_REGISTER_PROTOCOL(ijkio);
    IJK_REGISTER_PROTOCOL(async);
    IJK_REGISTER_PROTOCOL(ijklongurl);
    IJK_REGISTER_PROTOCOL(ijktcphook);
    IJK_REGISTER_PROTOCOL(ijkhttphook);
    IJK_REGISTER_PROTOCOL(ijksegment);
    /* demuxers */
    IJK_REGISTER_DEMUXER(ijklivehook);
    av_log(NULL, AV_LOG_INFO, "===== custom modules end =====\n");
}

```

#  实现demuxer

我们仿照ijkiourlhoock实现基本方法


```C

#include "libavformat/avformat.h"
#include "libavformat/url.h"
#include "libavutil/avstring.h"
#include "libavutil/opt.h"

#include "ijkplayer/ijkavutil/opt.h"

#include "ijkavformat.h"
#include "libavutil/application.h"


typedef struct {
    AVClass *class;

    int64_t app_ctx_intptr;

    AVDictionary *opts;
} Context;

//返回0-AVPROBE_SCORE_MAX 之间，AVPROBE_SCORE_MAX表示最匹配就会使用此demuxer
static int rtmj_probe(AVProbeData *probe) {
    if (av_strstart(probe->filename, "customer:", NULL))
        return AVPROBE_SCORE_MAX;
    return 0;
}

static int rtmj_read_close(AVFormatContext *avf) {
    return 0;
}

// FIXME: install libavformat/internal.h
int ff_alloc_extradata(AVCodecParameters *par, int size);


static int rtmj_read_header(AVFormatContext *ic, AVDictionary **options) {
    av_log(NULL, AV_LOG_INFO, "rtmj read head");
    Context *s = ic->priv_data;

    char *param = ic->filename;
   //读取url

    //视频
    AVStream *st;
    st = avformat_new_stream(ic, NULL);
    st->codecpar->codec_type = AVMEDIA_TYPE_VIDEO;
    st->codecpar->codec_id = AV_CODEC_ID_H265;
    st->codecpar->format = AV_PIX_FMT_YUV420P;
    st->codecpar->profile = FF_PROFILE_H264_HIGH;
    //设置帧率
    st->r_frame_rate.num = 30;
    st->r_frame_rate.den = 1;
    st->codecpar->width = 640;
    st->codecpar->height = 360;
    st->start_time = 0;

      //音频
      AVStream *audioST = avformat_new_stream(ic, NULL);
      audioST->codecpar->codec_type = AVMEDIA_TYPE_AUDIO;
      audioST->codecpar->codec_id = AV_CODEC_ID_PCM_ALAW;
      audioST->codecpar->channels = channel;
      audioST->codecpar->sample_rate = sampleRate;
      audioST->codecpar->format = AV_SAMPLE_FMT_S16;

    return 1;
}

static int rtmj_read_packet(AVFormatContext *ic, AVPacket *pkt) {
    Context *s = ic->priv_data;
    //自定义获取数据方法 int ret = read_data(pkt);
    return ret;
}

#define OFFSET(x) offsetof(Context, x)
#define D AV_OPT_FLAG_DECODING_PARAM

static const AVOption options[] = {
        {"ijkapplication", "AVApplicationContext", OFFSET(app_ctx_intptr), AV_OPT_TYPE_INT64, {.i64 = 0}, INT64_MIN, INT64_MAX, .flags = D},
        {NULL}
};

#undef D
#undef OFFSET

static const AVClass rtmj_class = {
        .class_name = "RTMJ demuxer",
        .item_name  = av_default_item_name,
        .option     = options,
        .version    = LIBAVUTIL_VERSION_INT,
};

AVInputFormat ijkff_rtmj_demuxer = {
        .name           = "rtmj",
        .long_name      = "RTMJ Controller",
        .flags          = AVFMT_NOFILE | AVFMT_TS_DISCONT,
        .priv_data_size = sizeof(Context),
        .read_probe     = rtmj_probe,
        .read_header2   = rtmj_read_header,
        .read_packet    = rtmj_read_packet,
        .read_close     = rtmj_read_close,
        .priv_class     = &rtmj_class,
        .extensions     = "rtmj",
};


```

关键定义都在AVInputFormat结构体中，名称对应上就能注册进，然后实现对应方法即可。

