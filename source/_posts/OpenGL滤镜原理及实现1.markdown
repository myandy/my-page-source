---
title:      OpenGL滤镜原理及实现1
description: 颜色查找表原理及实现
date:       2017-1-14 12:00:00
author:     "安地"
tags:
          - Android
          - OpenGL
---

## Opengl着色器语言（glsl）

着色器(Shader)是运行在GPU上的小程序。着色器程序包括顶点着色器和片段着色器。

顶点着色器。对于发送给GPU的每一个顶点，都要执行一次顶点着色器。其功能是把每个顶点在虚拟空间中的三维坐标变换为可以在屏幕上显示的二维坐标，并带有用于z-buffer的深度信息。顶点着色器可以操作的属性有：位置、颜色、纹理坐标，但是不能创建新的顶点。

片元着色器。片元着色器计算每个像素的颜色和其它属性。它通过应用光照值、凹凸贴图，阴影，镜面高光，半透明等处理来计算像素的颜色并输出。它也可改变像素的深度(z-buffering)或在多个渲染目标被激活的状态下输出多种颜色。一个片元着色器不能产生复杂的效果，因为它只在一个像素上进行操作，而不知道场景的几何形状。

用顶点着色器和片段着色器可以写出各种各种样的opengl程序。

## 颜色查找表(Color Lookup Table)


![Alt text](/img/post_filter_1.png)

在影像处理的领域中，当我们想要调整一个影像的色彩时，时常会用到颜色查找表（简称ColorLUT）的技术。
举个简单的例子，如果我们想要让影像中每个像素的0.3倍，最基本的作法就是把每一个像素的R值乘以0.3，假设影像的大小为1024x768，那么总共要786432次浮点数乘法。
如果我们一开始先建一张表，把所有色彩值经过处理（R值变为0.3倍）之后的结果记录起来，然后把每个像素的色彩值拿去查表，得到处理之后的色彩值，那么我们只要做786432查看动作，会比浮点运算快上许多。实际上大部分色彩调整的演算法都比这个例子复杂许多，因此更多凸显出查表法的高效率。
以RGB 24位为例，每个像素占3字节，而总共有16777216（256x256x256）种色彩，总共占48MB （256x256x256x3字节），看来并不小。因此实务上我们并不会记下所有的色彩，而是只记下部分的色彩，其他不在表内的色彩，用内插法取得处理后的结果。上图就是颜色转换的立方体。

![Alt text](/img/post_filter_2.png)


把立方体展开成二维图片就如上图，这是一张8x8的图片，分别包含了4个4x4的正方形，以右上角正方形为例，这代表Z = 0的平面，而X轴由左至右，Y轴为由上至下 ，左上角第一个像素代表位于（0,0,0）的点，第二个像素代表位（85,0,0）的点，以此类推，由于这些像素代表的是未处理前的颜色 ，因此第一个像素的RGB值为0,0,0，第二的像素的RGB值为85,0,0。


![Alt text](/img/post_filter_3.png)
实际使用得最多的是512*512的颜色查找表，对应为64*64*64，共64个小正方形，每个小正方形为64*64像素。

![Alt text](/img/post_filter_4.png)
还有一种常用的是256*3的颜色查找表，只能单独对每一种RGB颜色进行处理转换。

## 背景叠加图

叠加图实际是根据位置进行颜色改变。

![Alt text](/img/post_filter_5.png)
通过上图blackboard图获取到对应位置的RGB颜色，作为X坐标，

Y坐标是原图的RGB，通过这个XY坐标去取下图overlayMap图对应的颜色。

![Alt text](/img/post_filter_6.png)

这样两步就把覆盖颜色叠加上去了。

## 滤镜实例

这个实例是Hudson滤镜，blackboard结合overlayMap实现了背景叠加，map实现了颜色滤镜，使用的是上面的255*3的颜色查找表，三种颜色中点分别是1/6,1/2，5/6，故在这些位置上取对应颜色。

### 顶点着色器

    attribute vec4 position;
    attribute vec4 inputTextureCoordinate;
    varying vec2 textureCoordinate;

    void main()
    {
        gl_Position = position;
        textureCoordinate = inputTextureCoordinate.xy;
    }


### 片段着色器

    precision mediump float;
    varying mediump vec2 textureCoordinate;
    uniform sampler2D inputImageTexture;   //orginalImage
    uniform sampler2D inputImageTexture2; //blackboard;
    uniform sampler2D inputImageTexture3; //overlayMap;
    uniform sampler2D inputImageTexture4; //map
    uniform float strength;

    void main()
     {
         vec4 originColor = texture2D(inputImageTexture, textureCoordinate);
         vec4 texel = texture2D(inputImageTexture, textureCoordinate);
         vec3 bbTexel = texture2D(inputImageTexture2, textureCoordinate).rgb;
         texel.r = texture2D(inputImageTexture3, vec2(bbTexel.r, texel.r)).r;
         texel.g = texture2D(inputImageTexture3, vec2(bbTexel.g, texel.g)).g;
         texel.b = texture2D(inputImageTexture3, vec2(bbTexel.b, texel.b)).b;

         vec4 mapped;
         mapped.r = texture2D(inputImageTexture4, vec2(texel.r, .16666)).r;
         mapped.g = texture2D(inputImageTexture4, vec2(texel.g, .5)).g;
         mapped.b = texture2D(inputImageTexture4, vec2(texel.b, .83333)).b;
         mapped.a = 1.0;

         mapped.rgb = mix(originColor.rgb, mapped.rgb, strength);
         gl_FragColor = mapped;

     }

## 效果图

![Alt text](/img/post_filter_7.png)

    从左到右分别为原图，颜色处理图，背景处理图，全处理图。
## 总结

    颜色查找表是滤镜里面最常用的处理方法，在glsl中使用就可以达到各种各样的效果。

    颜色查找表部分内容来自:
<http://huangtw-blog.logdown.com/posts/176980-ios-quickly-made-using-a-cicolorcube-filter>