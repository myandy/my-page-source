---
title:      "YUV和RGB互相转换及OpenGL显示YUV数据"
description:   "人眼对亮度信息更敏感,所以显示器都是直接使用的YUV数据,Android从相机拿到的原始数据就是YUV格式,所以底层图像处理的算法都是基于YUV.
                Android上层Bitmap使用的是RGBA格式,OpenGL也是用的RGBA数据,这就需要灵活转换"
date:       2017-12-4 12:00:00
author:     "安地"
tags:
      - OpenG
      - Android
---

## 前言

人眼对亮度信息更敏感,所以显示器都是直接使用的YUV数据,Android从相机拿到的原始数据就是YUV格式,所以底层图像处理的算法都是基于YUV.
Android上层Bitmap使用的是RGBA格式,OpenGL也是用的RGBA数据,这就需要灵活转换.

## 转换公式

RGB转YUV

    Y = 0.299 R + 0.587 G + 0.114 B

    U = - 0.1687 R - 0.3313 G + 0.5 B + 128

    V = 0.5 R - 0.4187 G - 0.0813 B + 128

YUV转RGB

    R = Y + 1.402 (V-128)

    G = Y - 0.34414 (U-128) - 0.71414 (V-128)

    B = Y + 1.772 (U-128)


## YUV格式

YUV有很多种格式,android常用的YUV420sp.YUV的主要信息是亮度Y,YUV420sp格式也即NV21,Y会占4位,UV一共占2位,交替出现,就是一个UV数据要对应4个Y数据,这样YUV数据的size就是RGB数据的1.5倍,
虽然size增大了,但每一位是一个字节,RGBA的话每一位是int型,占4个字节,所以整体大小是减小了,且丢失了一部分UV信息,但这些肉眼也基本感觉不出来.

RGB转YUV420sp算法:

    void encodeyuv420sp(int *rgbData, MUInt8 *yuv, int width, int height) {
        int frameSize = width * height;

        int yIndex = 0;
        int uvIndex = frameSize;
        int R, G, B, Y, U, V;
        int index = 0;
        for (int j = 0; j < height; j++) {
            for (int i = 0; i < width; i++) {

                B = (rgbData[index] & 0xff0000) >> 16 ;
                G = (rgbData[index] & 0xff00) >> 8 ;
                R = (rgbData[index] & 0xff) ;

                Y = (int)((0.299 * R + 0.587 * G + 0.114 * B ) + 0.5 ) ;
                U = (int)((-0.147 * R - 0.289 * G + 0.436 * B ) + 0.5 + 128);
                V = (int)((0.615 * R - 0.515 * G - 0.100 * B ) + 0.5  + 128);

                yuv[yIndex++] = (Y < 0) ? 0 : ((Y > 255) ? 255 : Y);
                if (j % 2 == 0 && index % 2 == 0) {
                    yuv[uvIndex++] = (V<0) ? 0 : ((V > 255) ? 255 : V);
                    yuv[uvIndex++] = (U<0) ? 0 : ((U > 255) ? 255 : U);
                }

                index ++;
            }
        }
    }

## OpenGL中加载NV21数据

先加载YUV数据,分割成YBuffer和uvBuffer,再在OpenGL线程加载成纹理.这里要非常注意宽度必须是4的倍数,否则显示会形变.
因为OpenGL纹理加载必须是2的倍数,YUVFilter需要传入UV的纹理,UV的纹理的宽度是原图宽度的一半.这个问题找了非常久,后面发现是和大小有关,才一步步找到原因.

        /**
         * OpenGL纹理加载必须是2的倍数,YUVFilter需要传入UV的纹理,所以宽度必须是4的倍数
         *
         * @param data
         * @param width  必须是4的倍数
         * @param height
         */
        public void genYUVTextures(final byte[] data, final int width, final int height) {
            final int bufferSize = width * height;
            if (mYBuffer == null) {
                mYBuffer = ByteBuffer.allocateDirect(bufferSize);
                mYBuffer.order(ByteOrder.nativeOrder());
            }

            if (mUVBuffer == null) {
                mUVBuffer = ByteBuffer.allocateDirect(bufferSize / 2);
                mUVBuffer.order(ByteOrder.nativeOrder());
            }

            mYBuffer.put(data, 0, bufferSize).position(0);
            mUVBuffer.put(data, bufferSize, bufferSize >> 1).position(0);

            runOnDraw(new Runnable() {
                @Override
                public void run() {
                    OpenGlUtils.loadYuvToTvextures(mYBuffer, mUVBuffer, width, height, yuvTextureIds);
                }
            });
        }

 yBuffer是原图片大小的纹理,uvBuffer的宽高都为原图片的1/2,正好1/4大小,然后按位置即可对应到图片的RGB数据了.

     public static void loadYuvToTextures(final Buffer channelY, final Buffer channelUV, final int width, final int height, int[] yuvTextures) {
            if (channelY == null || channelUV == null) {
                return;
            }
            if (yuvTextures == null || yuvTextures.length < 2) {
                return;
            }

            if (yuvTextures[0] == NO_TEXTURE) {
                GLES20.glGenTextures(1, yuvTextures, 0);
                GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
                GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, yuvTextures[0]);

                GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_LUMINANCE, width, height, 0,
                        GLES20.GL_LUMINANCE, GLES20.GL_UNSIGNED_BYTE, channelY);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
                        GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
                        GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
                        GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
                        GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE);
            } else {
                GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
                GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, yuvTextures[0]);
                GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_LUMINANCE, width, height, 0,
                        GLES20.GL_LUMINANCE, GLES20.GL_UNSIGNED_BYTE, channelY);
            }

            if (yuvTextures[1] == NO_TEXTURE) {
                GLES20.glGenTextures(1, yuvTextures, 1);
                GLES20.glActiveTexture(GLES20.GL_TEXTURE1);
                GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, yuvTextures[1]);

                GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_LUMINANCE_ALPHA, width / 2,
                        height / 2, 0,
                        GLES20.GL_LUMINANCE_ALPHA, GLES20.GL_UNSIGNED_BYTE, channelUV);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
                        GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
                        GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
                        GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE);
                GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
                        GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE);
            } else {
                GLES20.glActiveTexture(GLES20.GL_TEXTURE1);
                GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, yuvTextures[1]);
                GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_LUMINANCE_ALPHA, width / 2,
                        height / 2, 0,
                        GLES20.GL_LUMINANCE_ALPHA, GLES20.GL_UNSIGNED_BYTE, channelUV);
            }
        }

片段着色器代码,从对应位置加载到YUV数据,然后转换成RGB,即可显示出来了.

        public static final String fragmentShader =
                "precision highp float;                             \n" +

                        "varying vec2 v_texCoord;                           \n" +
                        "uniform sampler2D y_texture;                       \n" +
                        "uniform sampler2D uv_texture;                      \n" +

                        "void main (void){                                  \n" +
                        "   float r, g, b, y, u, v;                         \n" +

                        //We had put the Y values of each pixel to the R,G,B components by GL_LUMINANCE,
                        //that's why we're pulling it from the R component, we could also use G or B
                        "   y = texture2D(y_texture, v_texCoord).r;         \n" +

                        //We had put the U and V values of each pixel to the A and R,G,B components of the
                        //texture respectively using GL_LUMINANCE_ALPHA. Since U,V bytes are interspread
                        //in the texture, this is probably the fastest way to use them in the shader
                        "   u = texture2D(uv_texture, v_texCoord).a - 0.5;  \n" +
                        "   v = texture2D(uv_texture, v_texCoord).r - 0.5;  \n" +

                        //The numbers are just YUV to RGB conversion constants
                        "   r = y + 1.402 * v;\n" +
                        "   g = y - 0.34414 * u - 0.71414 * v;\n" +
                        "   b = y + 1.772 * u;\n" +

                        //We finally set the RGB color of our pixel
                        "   gl_FragColor = vec4(r, g, b, 1.0);              \n" +
                        "}                                                  \n";

## 总结

在处理算法是YUV格式的,且需要频繁调用时,用OpenGL直接加载YUV数据效率是非常高的,用OpenGL作为显示,最后保存时再处理一遍就行了.