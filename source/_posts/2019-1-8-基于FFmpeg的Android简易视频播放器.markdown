---
title:      "基于FFmpeg的Android简易视频播放器"
description:   "自己动手写个视频播放器"
date:       2019-1-8 12:00:00
author:     "安地"
tags:
      - 音视频
      - FFmpeg
---

# 正文

一个视频文件到播放在屏幕上，主要有这几个步骤，
1.解封装，得到视频流和音频流；
2.解码 得到视频原始帧，一般是yuv格式数据；
3.渲染 把图像帧数据绘制到屏幕上

前两步使用FFmpeg解析，渲软使用native window操纵界面显示,下面来自己动手试试。

## 视频解码

FFmpeg解码视频步骤：
1.av_register_all 注册初始化
2.avformat_open_input 打开文件
3.av_find_stream_info 查找流信息
4.avcode_find_decoder 获取解码器
5.avcode_open 打开解码器
6.av_read_frame 读取帧

FFmpeg的方法名都带av即audio and video，都是同时支持音视频的。

视频解码代码：

```
JNIEXPORT jint JNICALL Java_com_anddymao_ffmpegdemo_MainActivity_decode
  (JNIEnv *env, jobject obj, jstring input_jstr, jstring output_jstr)
{
	AVFormatContext	*pFormatCtx;
	int				i, videoindex;
	AVCodecContext	*pCodecCtx;
	AVCodec			*pCodec;
	AVFrame	*pFrame,*pFrameYUV;
	uint8_t *out_buffer;
	AVPacket *packet;
	int y_size;
	int ret, got_picture;
	struct SwsContext *img_convert_ctx;
	FILE *fp_yuv;
	int frame_cnt;
	clock_t time_start, time_finish;
	double  time_duration = 0.0;

	char input_str[500]={0};
	char output_str[500]={0};
	char info[1000]={0};
	sprintf(input_str,"%s",(*env)->GetStringUTFChars(env,input_jstr, NULL));
	sprintf(output_str,"%s",(*env)->GetStringUTFChars(env,output_jstr, NULL));

	//FFmpeg av_log() callback
  av_log_set_callback(custom_log);

	av_register_all();
	avformat_network_init();
	pFormatCtx = avformat_alloc_context();

	if(avformat_open_input(&pFormatCtx,input_str,NULL,NULL)!=0){
		LOGE("Couldn't open input stream.\n");
		return -1;
	}
	if(avformat_find_stream_info(pFormatCtx,NULL)<0){
		LOGE("Couldn't find stream information.\n");
		return -1;
	}
	videoindex=-1;
	for(i=0; i<pFormatCtx->nb_streams; i++)
		if(pFormatCtx->streams[i]->codec->codec_type==AVMEDIA_TYPE_VIDEO){
			videoindex=i;
			break;
		}
	if(videoindex==-1){
		LOGE("Couldn't find a video stream.\n");
		return -1;
	}
	pCodecCtx=pFormatCtx->streams[videoindex]->codec;
	pCodec=avcodec_find_decoder(pCodecCtx->codec_id);
	if(pCodec==NULL){
		LOGE("Couldn't find Codec.\n");
		return -1;
	}
	if(avcodec_open2(pCodecCtx, pCodec,NULL)<0){
		LOGE("Couldn't open codec.\n");
		return -1;
	}

	pFrame=av_frame_alloc();
	pFrameYUV=av_frame_alloc();
	out_buffer=(unsigned char *)av_malloc(av_image_get_buffer_size(AV_PIX_FMT_YUV420P,  pCodecCtx->width, pCodecCtx->height,1));
	av_image_fill_arrays(pFrameYUV->data, pFrameYUV->linesize,out_buffer,
		AV_PIX_FMT_YUV420P,pCodecCtx->width, pCodecCtx->height,1);


	packet=(AVPacket *)av_malloc(sizeof(AVPacket));

	img_convert_ctx = sws_getContext(pCodecCtx->width, pCodecCtx->height, pCodecCtx->pix_fmt,
	pCodecCtx->width, pCodecCtx->height, AV_PIX_FMT_YUV420P, SWS_BICUBIC, NULL, NULL, NULL);


  sprintf(info,   "[Input     ]%s\n", input_str);
  sprintf(info, "%s[Output    ]%s\n",info,output_str);
  sprintf(info, "%s[Format    ]%s\n",info, pFormatCtx->iformat->name);
  sprintf(info, "%s[Codec     ]%s\n",info, pCodecCtx->codec->name);
  sprintf(info, "%s[Resolution]%dx%d\n",info, pCodecCtx->width,pCodecCtx->height);


  fp_yuv=fopen(output_str,"wb+");
  if(fp_yuv==NULL){
		printf("Cannot open output file.\n");
		return -1;
	}

	frame_cnt=0;
	time_start = clock();

	while(av_read_frame(pFormatCtx, packet)>=0){
		if(packet->stream_index==videoindex){
			ret = avcodec_decode_video2(pCodecCtx, pFrame, &got_picture, packet);
			if(ret < 0){
				LOGE("Decode Error.\n");
				return -1;
			}
			if(got_picture){
				sws_scale(img_convert_ctx, (const uint8_t* const*)pFrame->data, pFrame->linesize, 0, pCodecCtx->height,
					pFrameYUV->data, pFrameYUV->linesize);

				y_size=pCodecCtx->width*pCodecCtx->height;
				fwrite(pFrameYUV->data[0],1,y_size,fp_yuv);    //Y
				fwrite(pFrameYUV->data[1],1,y_size/4,fp_yuv);  //U
				fwrite(pFrameYUV->data[2],1,y_size/4,fp_yuv);  //V
				//Output info
				char pictype_str[10]={0};
				switch(pFrame->pict_type){
					case AV_PICTURE_TYPE_I:sprintf(pictype_str,"I");break;
				  case AV_PICTURE_TYPE_P:sprintf(pictype_str,"P");break;
					case AV_PICTURE_TYPE_B:sprintf(pictype_str,"B");break;
					default:sprintf(pictype_str,"Other");break;
				}
				LOGI("Frame Index: %5d. Type:%s",frame_cnt,pictype_str);
				frame_cnt++;
			}
		}
		av_free_packet(packet);
	}
	//flush decoder
	//FIX: Flush Frames remained in Codec
	while (1) {
		ret = avcodec_decode_video2(pCodecCtx, pFrame, &got_picture, packet);
		if (ret < 0)
			break;
		if (!got_picture)
			break;
		sws_scale(img_convert_ctx, (const uint8_t* const*)pFrame->data, pFrame->linesize, 0, pCodecCtx->height,
			pFrameYUV->data, pFrameYUV->linesize);
		int y_size=pCodecCtx->width*pCodecCtx->height;
		fwrite(pFrameYUV->data[0],1,y_size,fp_yuv);    //Y
		fwrite(pFrameYUV->data[1],1,y_size/4,fp_yuv);  //U
		fwrite(pFrameYUV->data[2],1,y_size/4,fp_yuv);  //V
		//Output info
		char pictype_str[10]={0};
		switch(pFrame->pict_type){
			case AV_PICTURE_TYPE_I:sprintf(pictype_str,"I");break;
		  case AV_PICTURE_TYPE_P:sprintf(pictype_str,"P");break;
			case AV_PICTURE_TYPE_B:sprintf(pictype_str,"B");break;
			default:sprintf(pictype_str,"Other");break;
		}
		LOGI("Frame Index: %5d. Type:%s",frame_cnt,pictype_str);
		frame_cnt++;
	}
	time_finish = clock();
	time_duration=(double)(time_finish - time_start);

	sprintf(info, "%s[Time      ]%fms\n",info,time_duration);
	sprintf(info, "%s[Count     ]%d\n",info,frame_cnt);

	sws_freeContext(img_convert_ctx);

  fclose(fp_yuv);

	av_frame_free(&pFrameYUV);
	av_frame_free(&pFrame);
	avcodec_close(pCodecCtx);
	avformat_close_input(&pFormatCtx);

	return 0;
}
```

这里是复制雷神的源码，解码成流后保存到文件中，改到自己工程试了没有问题，注意下读写权限就可以了。

这里解析出视频帧后打印了视频帧的类型（I，P，B帧）
源码在<https://github.com/leixiaohua1020/simplest_ffmpeg_mobile>

## 视频播放


### 底层代码

视频播放在解码之后需要增加渲软的工作，在c里面用native window就可以对应到java的surface。代码如下：

```
ANativeWindow *nativeWindow;
ANativeWindow_Buffer windowBuffer;


JNIEXPORT jint JNICALL Java_com_anddymao_ffmpegdemo_MainActivity_play
        (JNIEnv *env, jobject obj, jstring input_jstr, jobject surface) {
    AVFormatContext *pFormatCtx;
    int i, videoindex;
    AVCodecContext *pCodecCtx;
    AVCodec *pCodec;
    struct SwsContext *img_convert_ctx;

    char input_str[500] = {0};
    sprintf(input_str, "%s", (*env)->GetStringUTFChars(env, input_jstr, NULL));

    av_register_all();
    avformat_network_init();
    pFormatCtx = avformat_alloc_context();

    if (avformat_open_input(&pFormatCtx, input_str, NULL, NULL) != 0) {
        LOGE("Couldn't open input stream.\n");
        return -1;
    }
    if (avformat_find_stream_info(pFormatCtx, NULL) < 0) {
        LOGE("Couldn't find stream information.\n");
        return -1;
    }
    videoindex = -1;
    for (i = 0; i < pFormatCtx->nb_streams; i++)
        if (pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO) {
            videoindex = i;
            break;
        }
    if (videoindex == -1) {
        LOGE("Couldn't find a video stream.\n");
        return -1;
    }
    pCodecCtx = pFormatCtx->streams[videoindex]->codec;
    pCodec = avcodec_find_decoder(pCodecCtx->codec_id);
    if (pCodec == NULL) {
        LOGE("Couldn't find Codec.\n");
        return -1;
    }
    if (avcodec_open2(pCodecCtx, pCodec, NULL) < 0) {
        LOGE("Couldn't open codec.\n");
        return -1;
    }

    //获取界面传下来的surface
    nativeWindow = ANativeWindow_fromSurface(env, surface);
    if (0 == nativeWindow) {
        LOGE("Couldn't get native window from surface.\n");
        return -1;
    }
    int width = pCodecCtx->width;
    int height = pCodecCtx->height;
    //分配一个帧指针，指向解码后的原始帧
    AVFrame *vFrame = av_frame_alloc();
    AVPacket *vPacket = (AVPacket *) av_malloc(sizeof(AVPacket));
    AVFrame *pFrameRGBA = av_frame_alloc();

    img_convert_ctx = sws_getContext(width, height, pCodecCtx->pix_fmt,
                                     width, height, AV_PIX_FMT_RGBA, SWS_BICUBIC, NULL, NULL, NULL);
    if (0 >
        ANativeWindow_setBuffersGeometry(nativeWindow, width, height, WINDOW_FORMAT_RGBA_8888)) {
        LOGE("Couldn't set buffers geometry.\n");
        ANativeWindow_release(nativeWindow);
        return -1;
    }

    //读取帧
    while (av_read_frame(pFormatCtx, vPacket) >= 0) {
        if (vPacket->stream_index == videoindex) {
            //视频解码
            int ret = avcodec_send_packet(pCodecCtx, vPacket);
            if (ret < 0 && ret != AVERROR(EAGAIN) && ret != AVERROR_EOF) {
                LOGE("video avcodec_send_packet error %d", ret);
                return -1;
            }
            ret = avcodec_receive_frame(pCodecCtx, vFrame);
            if (ret < 0 && ret != AVERROR_EOF) {
                LOGE("video avcodec_receive_frame error %d", ret);
                av_packet_unref(vPacket);
                continue;
            }
            //转化格式
            sws_scale(img_convert_ctx, (const uint8_t *const *) vFrame->data, vFrame->linesize, 0,
                      pCodecCtx->height,
                      pFrameRGBA->data, pFrameRGBA->linesize);
            if (ANativeWindow_lock(nativeWindow, &windowBuffer, NULL) < 0) {
                LOGE("cannot lock window");
            } else {
                av_image_fill_arrays(pFrameRGBA->data, pFrameRGBA->linesize,
                                     (const uint8_t *) windowBuffer.bits, AV_PIX_FMT_RGBA,
                                     width, height, 1);
                ANativeWindow_unlockAndPost(nativeWindow);
            }
        }
        av_packet_unref(vPacket);
    }
    //释放内存
    sws_freeContext(img_convert_ctx);
    av_free(vPacket);
    av_free(pFrameRGBA);
    avcodec_close(pCodecCtx);
    avformat_close_input(&pFormatCtx);
    return 0;

}
```

开始在avcodec_receive_frame这边返回了错误，改成continue就可以了，使用cmake可以很方便的调试native代码。

### Java层调用

java代码就非常简单，添加一个surfaceView，调用play方法就可以了：

```
        SurfaceView surfaceView = findViewById(R.id.surface_view);
        surfaceView.getHolder().addCallback(new SurfaceHolder.Callback() {
            @Override
            public void surfaceCreated(final SurfaceHolder holder) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        play(inputurl, holder.getSurface());
                    }
                }).start();
            }

            @Override
            public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {

            }

            @Override
            public void surfaceDestroyed(SurfaceHolder holder) {

            }
        });
public native int play(String inputurl, Surface surface);
```

添加好文件权限，这个就可以播放了，只有图像的播放，没有控制播放速度，基本的流程就走通了。


