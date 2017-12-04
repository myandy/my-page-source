---
title:      "Android视频滤镜"
description: "Android视频滤镜及录制保存"
date:       2017-3-27 15:00:00
author:     "安地"
tags:
         - Android
         - OpenGL
---

## 前言

照片滤镜还有相机实时滤镜的开源项目非常多，最近因为项目需要学习研究了很多，就想做点有区别的东西，空闲就弄了这样一个视频滤镜的项目。项目核心主要有滤镜，视频绘制，视频保存，视频绘制，音视频合成，简单介绍一下思路方法，详细实现请见底部的github地址。

## 滤镜

滤镜参考MagicCamera，主要用到LUT（颜色查找表），LUT来自B612，反编译后找到了资源的加解密算法，就拿来几个作为学习使用，但有额外叠加的filter没有破解所以效果还是有点差别，另外拿到片段着色器后自己加上了一个混合度调整参数。

## 视频保存

视频保存是离屏播放录制surface，录制和播放应该是分开的，所以录制和播放的控制也是分开的，分为两个渲染线程MovieRenderThread和RecordRenderThread，但最后绘制的内容是一致的，所以有共同的渲染器MovieRenderer。
控制周期用的TimerTask，周期为视频的帧间隔，这里我们也可以使用视频的周期，但这种更加灵活，没有视频时也可以录制，比如实现把照片生成视频。
录制surface用MediaCodec，MagicCamera和Google的grafika都有具体的实现。

## 视频绘制

视频绘制在MovieRenderer中，VideoPlayer播放视频到SurfaceTexture中，SurfaceTexture的纹理Id包进一个VideoFilter，通过GpuImageFilter的方式进行滤镜叠加。
整个绘制流程里只设计一个接口IMovieRenderer，从TimerTask发消息到RenderThread，RenderThread转发到MovieRenderer，RenderThread和MovieRenderer都去实现了IMovieRenderer，结构还算清晰简单。

## 音视频合成

视频保存时播放是静音的，录制出纯视频，再和原视频文件的音频流合成既可生成完整的视频。分离和合成视频使用MediaExtractor和MediaMuxer。



## 后记

每个部分的技术难度都只能算中等，整个框架的结构和稳定性最费精力，GpuImageFilter的基本结构可以扩展出各种广义的滤镜，做打码，加logo等功能，以后再慢慢完善。
github地址：<https://github.com/myandy/VideoFilter>