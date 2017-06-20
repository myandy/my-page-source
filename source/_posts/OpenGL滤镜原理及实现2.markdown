---
title:      OpenGL滤镜原理及实现2
description: 常用滤镜算法详解
date:       2017-6-19 12:00:00
author:     安地
tags:
          - OpenGL
---

# 滤镜算法详解

看本篇前请先看OpenGL滤镜原理及实现1,1中主要讲了颜色查找表类型滤镜，这里补充下讲解，再介绍下其它的一些滤镜。

## 颜色查找表滤镜

### 分色颜色转换滤镜

这种滤镜可以分别对RGB颜色进行转换，读取一张512*3像素的颜色查找表图片，分为RGB三行，位置是原色值，位置上的颜色是转换后的色值。

实现算法非常简单，片段着色器代码如下：

     precision mediump float;

     varying mediump vec2 textureCoordinate;

     uniform sampler2D inputImageTexture;
     uniform sampler2D inputImageTexture2;

     void main()
     {

         vec3 texel = texture2D(inputImageTexture, textureCoordinate).rgb;

         texel = vec3(
                      texture2D(inputImageTexture2, vec2(texel.r, .16666)).r,  //取上1/3中心点的对应位置颜色
                      texture2D(inputImageTexture2, vec2(texel.g, .5)).g,      //取中1/3中心点的对应位置颜色
                      texture2D(inputImageTexture2, vec2(texel.b, .83333)).b); //取下1/3中心点的对应位置颜色

         gl_FragColor = vec4(texel, 1.0);
     }

![分色查找表图](/img/post_filter_4.png)

### 512颜色查找表滤镜

原理前面讲过，就不介绍了，片段着色器代码如下：

        precision mediump float;
        uniform sampler2D inputImageTexture;
        uniform sampler2D inputImageTexture2; // lookup texture
        uniform float strength;
        void main()
        {
        lowp vec4 textureColor = texture2D(inputImageTexture, textureCoordinate);

        mediump float blueColor = textureColor.b * 63.0;

        mediump vec2 quad1;
        quad1.y = floor(blueColor/8.0);
        quad1.x = floor(blueColor) - (quad1.y * 8.0);

        mediump vec2 quad2;
        quad2.y = floor(ceil(blueColor)/7.999);
        quad2.x = ceil(blueColor) - (quad2.y * 8.0);

        highp vec2 texPos1;
        texPos1.x = (quad1.x * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.r);
        texPos1.y = (quad1.y * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.g);

        highp vec2 texPos2;
        texPos2.x = (quad2.x * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.r);
        texPos2.y = (quad2.y * 0.125) + 0.5/512.0 + ((0.125 - 1.0/512.0) * textureColor.g);

        lowp vec4 newColor1 = texture2D(inputImageTexture2, texPos1);
        lowp vec4 newColor2 = texture2D(inputImageTexture2, texPos2);

        lowp vec4 newColor = mix(newColor1, newColor2, fract(blueColor));
        gl_FragColor = mix(textureColor, vec4(newColor.rgb, textureColor.w), strength);
        }

因为一共是64个8×8的小正方形，先根据blue色值找到对应的小正方形的坐标，再根据red色值算出位置小正方形内的横坐标，根据green色值算出小正方形内的纵坐标，就可以得出整体坐标了。
要注意两个点，一个是位置计算是取中心点，所以有0.5/512等于半个像素的偏移，再计算RG时又向左偏移了一个像素，这样为0时向右偏移半个像素计算，其它值时向左偏移半个像素计算，非常巧妙。
还有一个是色值偏差处理，因为只能处理64×64×64中颜色，就会有不准确的问题，之前说到内插法处理偏移，这里就是取两个点的色值根据比例取最后的色值。blueColor取整后只能算到0-63，实际却有0-255，最后就需要根据小数部分计算出最近的两个点，这样就能得出相对准确的值。

![512查找表图](/img/post_filter_3.png)

## 基础效果滤镜

### 对比度

对比度是原色值中差值大小，让原色值减去中间数，这里简单点直接定为0.5，乘以对比度的比例，再加上原来的中间数。
片段着色器代码如下：

             varying highp vec2 textureCoordinate;
             uniform sampler2D inputImageTexture;
             uniform lowp float contrast;
             void main()
             {
                 lowp vec4 textureColor = texture2D(inputImageTexture, textureCoordinate);
                 gl_FragColor = vec4(((textureColor.rgb - vec3(0.5)) * contrast + vec3(0.5)), textureColor.w);
             }
### 亮度

这个非常简单，rgb同时增加一个值就可以增加亮度了，代码就不贴了。

### 饱和度

饱和度是指某一颜色的纯度，饱和度越低越接近灰色，越高就越鲜艳。
先根据亮度比例计算出灰度值，用灰度值与原色通过饱和度混合就可以得到新的颜色了。

片段着色器代码如下：

    varying highp vec2 textureCoordinate;
    uniform sampler2D inputImageTexture;
    uniform lowp float saturation; //0.0-2.0

    // Values from "Graphics Shaders: Theory and Practice" by Bailey and Cunningham
    const mediump vec3 luminanceWeighting = vec3(0.2125, 0.7154, 0.0721);

    void main()
    {
        lowp vec4 textureColor = texture2D(inputImageTexture, textureCoordinate);
        lowp float luminance = dot(textureColor.rgb, luminanceWeighting);
        lowp vec3 greyScaleColor = vec3(luminance);

        gl_FragColor = vec4(mix(greyScaleColor, textureColor.rgb, saturation), textureColor.w);
    }

### 锐化

可以通过增加相邻像素点之间的对比，使图像清晰化，提高对比度，使画面更加鲜明。
算法过程就是根据一定间隔计算上下左右的颜色值，根据锐化程度改变中心点和四周点的差值。

顶点着色器代码如下：

            attribute vec4 position;
            attribute vec4 inputTextureCoordinate;

            uniform float imageWidthFactor;
            uniform float imageHeightFactor;
            uniform float sharpness;

            varying vec2 textureCoordinate;
            varying vec2 leftTextureCoordinate;
            varying vec2 rightTextureCoordinate;
            varying vec2 topTextureCoordinate;
            varying vec2 bottomTextureCoordinate;

            varying float centerMultiplier;
            varying float edgeMultiplier;

            void main()
            {
                gl_Position = position;

                mediump vec2 widthStep = vec2(imageWidthFactor, 0.0);
                mediump vec2 heightStep = vec2(0.0, imageHeightFactor);

                textureCoordinate = inputTextureCoordinate.xy;
                leftTextureCoordinate = inputTextureCoordinate.xy - widthStep;
                rightTextureCoordinate = inputTextureCoordinate.xy + widthStep;
                topTextureCoordinate = inputTextureCoordinate.xy + heightStep;
                bottomTextureCoordinate = inputTextureCoordinate.xy - heightStep;

                centerMultiplier = 1.0 + 4.0 * sharpness;
                edgeMultiplier = sharpness;
            }

片段着色器代码如下：

     varying highp vec2 textureCoordinate;
     varying highp vec2 leftTextureCoordinate;
     varying highp vec2 rightTextureCoordinate;
     varying highp vec2 topTextureCoordinate;
     varying highp vec2 bottomTextureCoordinate;

     varying highp float centerMultiplier;
     varying highp float edgeMultiplier;

     uniform sampler2D inputImageTexture;

     void main()
     {
         mediump vec3 textureColor = texture2D(inputImageTexture, textureCoordinate).rgb;
         mediump vec3 leftTextureColor = texture2D(inputImageTexture, leftTextureCoordinate).rgb;
         mediump vec3 rightTextureColor = texture2D(inputImageTexture, rightTextureCoordinate).rgb;
         mediump vec3 topTextureColor = texture2D(inputImageTexture, topTextureCoordinate).rgb;
         mediump vec3 bottomTextureColor = texture2D(inputImageTexture, bottomTextureCoordinate).rgb;

         gl_FragColor = vec4((textureColor * centerMultiplier - (leftTextureColor * edgeMultiplier + rightTextureColor * edgeMultiplier + topTextureColor * edgeMultiplier + bottomTextureColor * edgeMultiplier)), texture2D(inputImageTexture, bottomTextureCoordinate).w);
     }

### 晕影

晕影或暗角是指图像的外围部分的亮度或饱和度比中心区域低。
算法是先计算和中心点的距离，然后计算出距离在最近距离和最远距离中的比例，最后根据比例混合晕影颜色。

片段着色器代码如下：

    uniform sampler2D inputImageTexture;
    varying highp vec2 textureCoordinate;

    uniform lowp vec2 vignetteCenter;
    uniform lowp vec3 vignetteColor;
    uniform highp float vignetteStart;
    uniform highp float vignetteEnd;

    void main()
    {
        lowp vec3 rgb = texture2D(inputImageTexture, textureCoordinate).rgb;
        lowp float d = distance(textureCoordinate, vec2(vignetteCenter.x, vignetteCenter.y));
        lowp float percent = smoothstep(vignetteStart, vignetteEnd, d);
        gl_FragColor = vec4(mix(rgb.x, vignetteColor.x, percent), mix(rgb.y, vignetteColor.y, percent), mix(rgb.z, vignetteColor.z, percent), 1.0);
    }

晕影还可以通过图片叠加的方式去实现，PhotoShop中有各种图片叠加方式都可以转换到OpenGL中。

# 总结

还有一些常见效果，比如黑白，模糊等，PhotoShop中的效果都可以在OpenGL中实现，但多种处理叠加后用原始算法就性能非常差了，这时候用颜色查找表滤镜就非常合适了，一种非常好的空间换时间的方法。
