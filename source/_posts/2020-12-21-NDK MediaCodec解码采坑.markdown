---
title:      "NDK MediaCodec解码采坑记录"
description:   "使用NDK MediaCodec解码遇到的问题和解决方案"
date:       2020-12-21 14:00:00
author:     "安地"
tags:
      - Android
      - 音视频

---

# 介绍

Android硬件解码器坑是比较多的，看使用场景，我分享一些我遇到的坑，遇到基本很难直接找到解决方案，经过艰难的分析和寻找方案才找对解决办法。
这里作为记录和分享，遇到问题需要较真。

# 问题

## 获取视频宽高


获取视频大小

```
AMediaFormat_getInt32(format, AMEDIAFORMAT_KEY_WIDTH, &width);
AMediaFormat_getInt32(format, AMEDIAFORMAT_KEY_HEIGHT, &height);
```

用这个值是视频原始大小，如果需要获取crop大小,即视频真正显示的大小，需要这样去获取。
java层：
```java
if (outformat.containsKey("crop-left") && outformat.containsKey("crop-right")) {
    mVideoWidth = outformat.getInteger("crop-right") + 1 - outformat.getInteger("crop-left");
}
if (outformat.containsKey("crop-top") && outformat.containsKey("crop-bottom")) {
    mVideoHeight = outformat.getInteger("crop-bottom") + 1 - outformat.getInteger("crop-top");
}
```
native：
```
AMediaFormat *format = AMediaCodec_getOutputFormat(decodeData->mCodec);
const char *s = AMediaFormat_toString(format);
std::string strFormat = s;
int cropbegin = strFormat.find("crop");
```

通过对应格式去获取crop大小，native下没有标准api可以直接获取，只能通过format去解析，麻烦很多。

## 延时设置

解码失败的问题。
AMediaCodec_dequeueInputBuffer和AMediaCodec_dequeueOutputBuffer需要等待缓冲队列，不设置延迟很容易失败，所以需要给一个timeout时间

```
#define AMC_INPUT_TIMEOUT_US  (100 * 1000)
#define AMC_OUTPUT_TIMEOUT_US (100 * 1000)

#define AMC_SYNC_INPUT_TIMEOUT_US  (30 * 1000)
#define AMC_SYNC_OUTPUT_TIMEOUT_US (30 * 1000)
```

这个是ijkplayer设置的异步和同步的timeout时间，分别是100ms和30ms，尽量保证能成功，这个可以根据实际情况进行调整。

如果数据不足会等待数据，我所用的场景新数据没到会等待一个timeout时间，感觉太慢，就在解码成功后把timeout时间设置一个较小的值。


## 硬解失败自动切换软解

Android硬解对不同机型不同格式不能完全保证适配，就需要一个判断。我们先判断机型是否支持硬解，再选择解码器进行解码。
但这种情况下还是有可能硬解失败，就需要增加解码失败自动切换软解机制。

实现是在AMediaCodec_dequeueOutputBuffer返回值判断，如果返回错误，立马初始化软解解码器，后面的帧就直接切换软解进行解码。


## 时间戳设置

硬解的时间戳也不能随意设置，各个系统实现不一样，比如vivo就给我反馈通过时间戳计算动态帧率，给出的时间戳不对会导致帧率计算错误，错误太大甚至会导致程序卡死。

之前介绍过音视频参数，包括系统时基。和pts，dts等，这次自己就遇到这个坑了。

我遇到的问题，pts给的毫秒时间戳，时基是90000，我直接传pts会导致部分机型有问题。
正确的pts = 毫秒时间戳 / 1000 × 90000
pts = 时间 × 时基，必须经过这个换算才对。

另外pts不可小于0，部分手机会直接解码失败，ijkplayer对这个也有设置防范，pts小于0直接返回0。

