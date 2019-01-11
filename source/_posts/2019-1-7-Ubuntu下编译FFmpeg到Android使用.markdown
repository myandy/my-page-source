---
title:      "Ubuntu下编译FFmpeg到Android使用"
description:   "自己动手编译并使用FFmpeg"
date:       2019-1-7 12:00:00
author:     "安地"
tags:
      - 音视频
      - FFmpeg
---

# 正文

最近学习音视频在底层的使用，那就必须研究FFmpeg,再次动手编译试试，用linux系统，可以事半功倍.
FFmpeg官网介绍是完全跨平台的音视频编解码解决方案。

## FFmpeg编译
### 开发环境
Ubuntu 14.04
FFmpeg源码（3.2.1）
ndk(android-ndk-r14b)
AndroidStudio3.0

### 下载FFmpeg源码

在官网 https://ffmpeg.org/download.html#releases 下载源码解压到本地，先试了4.1版，编译时发现问题过多，按推荐使用了3.2.1版。3.X版应该都没有问题。

### 修改configure文件

这个是因为编译so的名称修改，android识别必须后缀为so。不改的话生成的so需要手动重命名。

    SLIBNAME_WITH_MAJOR='$(SLIBNAME).$(LIBMAJOR)'
    LIB_INSTALL_EXTRA_CMD='$$(RANLIB)"$(LIBDIR)/$(LIBNAME)"'
    SLIB_INSTALL_NAME='$(SLIBNAME_WITH_VERSION)'
    SLIB_INSTALL_LINKS='$(SLIBNAME_WITH_MAJOR)$(SLIBNAME)'

替换成

    SLIBNAME_WITH_MAJOR='$(SLIBPREF)$(FULLNAME)-$(LIBMAJOR)$(SLIBSUF)'
    LIB_INSTALL_EXTRA_CMD='$$(RANLIB)"$(LIBDIR)/$(LIBNAME)"'
    SLIB_INSTALL_NAME='$(SLIBNAME_WITH_MAJOR)'
    SLIB_INSTALL_LINKS='$(SLIBNAME)'

### 编写脚本生成类库

在ffmpeg中创建一个build_android.sh的脚本，并赋予可执行的权限（chmod +x）

```
#!/bin/bash
make clean
#填写你具体的ndk解压目录
export NDK=/home/mi/Android/android-ndk-r14b
export SYSROOT=$NDK/platforms/android-9/arch-arm/
export TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64
export CPU=arm
#编译后的文件会放置在 当前路径下的android/arm／下
export PREFIX=$(pwd)/android/$CPU
export ADDI_CFLAGS="-marm"


#./configure 即为ffmpeg 根目录下的可执行文件configure
#你可以在ffmpeg根目录下使用./configure --hellp 查看 ./configure后可填入的参数。

./configure --target-os=linux \
        --prefix=$PREFIX --arch=arm \
        --disable-doc \
        --enable-shared \
        --disable-static \
        --disable-yasm \
        --disable-symver \
        --enable-gpl \
        --disable-ffmpeg \
        --disable-ffplay \
        --disable-ffprobe \
        --disable-ffserver \
        --disable-doc \
        --disable-symver \
        --cross-prefix=$TOOLCHAIN/bin/arm-linux-androideabi- \
        --enable-cross-compile \
        --sysroot=$SYSROOT \
        --extra-cflags="-Os -fpic $ADDI_CFLAGS" \
        --extra-ldflags="$ADDI_CFLAGS" \
        $ADDITIONAL_CONFIGURE_FLAG
make clean
make
make install
```

在当前目录./build_android.sh运行脚本，即可等待生成。最后成功的话可以在android/arm下看到include和lib，分别是ffmpeg的头文件和lib库。

## 在android工程中使用

### 创建Android工程
创建一个支持C++的ffmpegdemo工程

### 修改native-lib.cpp

这里就添加一个FFmpeg的注册方法

```
#include <jni.h>
#include <string>

extern "C"
{
#include "libavformat/avformat.h"
}
extern "C"
JNIEXPORT jstring

JNICALL
Java_com_anddymao_ffmpegdemo_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {

    av_register_all();
    std::string hello = "Hello,FFmpeg!";
    return env->NewStringUTF(hello.c_str());
}
```

### 添加lib
在app/src/main下创建jniLibs目录，再创建include和armeabi文件夹，把头文件拷贝到include，把lib/arm下的so拷贝到armeabi。
一共是8个so文件，下面会在CMake中配置编译，都是一一对应。

### 修改CMakeList

修改CMake中的编译信息，把ffmpeg添加的lib都添加为share library并链接进来

```
cmake_minimum_required(VERSION 3.4.1)

add_library( # Sets the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp
           )

find_library( # Sets the name of the path variable.
              log-lib
              log )


#声明导入文件更目录变量ARM_DIR
set(ARM_DIR /home/mi/mediapro/FFmpegForAndroidDemo/app/src/main/jniLibs)

#导入头文件
include_directories(src/main/jniLibs/include)

#添加动态库
add_library(avcodec-57
             SHARED
             IMPORTED)
#设置动态库路径
set_target_properties(avcodec-57
                      PROPERTIES IMPORTED_LOCATION
                      ${ARM_DIR}/armeabi/libavcodec-57.so
                        )

add_library(avdevice-57
            SHARED
            IMPORTED)
set_target_properties(avdevice-57
                      PROPERTIES IMPORTED_LOCATION
                      ${ARM_DIR}/armeabi/libavdevice-57.so)
add_library(avformat-57
            SHARED
            IMPORTED)
set_target_properties(avformat-57
                      PROPERTIES IMPORTED_LOCATION
                      ${ARM_DIR}/armeabi/libavformat-57.so)
add_library(avutil-55
            SHARED
            IMPORTED)
set_target_properties(avutil-55
                      PROPERTIES IMPORTED_LOCATION
                      ${ARM_DIR}/armeabi/libavutil-55.so)
add_library(postproc-54
            SHARED
            IMPORTED)
set_target_properties(postproc-54
                      PROPERTIES IMPORTED_LOCATION
                      ${ARM_DIR}/armeabi/libpostproc-54.so)
add_library(swresample-2
             SHARED
             IMPORTED)
set_target_properties(swresample-2
                       PROPERTIES IMPORTED_LOCATION
                       ${ARM_DIR}/armeabi/libswresample-2.so)
add_library(swscale-4
              SHARED
              IMPORTED)
set_target_properties(swscale-4
                        PROPERTIES IMPORTED_LOCATION
                        ${ARM_DIR}/armeabi/libswscale-4.so)
add_library(avfilter-6
              SHARED
              IMPORTED)
set_target_properties(avfilter-6
                        PROPERTIES IMPORTED_LOCATION
                        ${ARM_DIR}/armeabi/libavfilter-6.so)

#其他so库与上相同格式添加
#链接库

target_link_libraries(
                       native-lib
                       avcodec-57
                       avdevice-57
                       avformat-57
                       avfilter-6
                       avutil-55
                       postproc-54
                       swresample-2
                       swscale-4
                       ${log-lib} )
```

这里要注意native-lib只能放在第一个。

### 修改build.gradle

abiFilters设置armeabi平台
```
        externalNativeBuild {
            cmake {
                cppFlags "-frtti -fexceptions"
                abiFilters 'armeabi'
            }
        }
```

然后可以编译运行了，在android上看到"Hello,FFmpeg！",说明编译成功了。