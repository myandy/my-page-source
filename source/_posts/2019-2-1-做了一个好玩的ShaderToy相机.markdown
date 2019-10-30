---
title:      "做了一个好玩的ShaderToy相机"
description:   "基于Camera API2"
date:       2019-2-1 12:00:00
author:     "安地"
tags:
      - Android
      - OpenGL
---


## 介绍

<www.shadertoy.com>是一个构建着色器的网站，可以创建着色器做出很多漂亮的效果。感觉很有意思，就想放到相机上会好玩，就做出了这个demo，给大家分享下。


## 实现

### 原理

ShaderToy上每一个效果都有对应的代码，就是片段着色器代码，有一些自定义的参数，比如时间，位置，输入源纹理等。
把第一个输入源改成相机输入，就是相机滤镜了。采用GpuImage的框架，搭起一个支持滤镜的相机app，再把ShaderToy上的着色器代码转换过来，就可以实现ShaderToy相机。

### 着色器解析

先找了一个最简单的着色器分析，<https://www.shadertoy.com/view/MddBWn>,代码如下：

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord/iResolution.xy-0.5;
    uv *= 0.7 - mod(uv.x+uv.y,uv.x*0.5)*1.5;
    fragColor = texture(iChannel0,uv+0.5);
}
```
fragColor是输出颜色，fragCoord是坐标，iChannel0是输入原纹理，iResolution是点击的位置。知道这些就可以改成OpenGLEs上着色器代码了：

```glsl
precision highp float;

uniform float               iTime;
uniform sampler2D           inputImageTexture;
uniform sampler2D           inputImageTexture2;
varying vec2                textureCoordinate;
uniform vec3                iResolution;

void main()
{
    vec2 uv = textureCoordinate/iResolution.xy-0.5;
    uv *= 0.7 - mod(uv.x+uv.y,uv.x*0.5)*1.5;
    gl_FragColor = texture2D(inputImageTexture,uv+0.5);

}
```

这个效果叫fokkkus，有点像破碎的镜子向外发射，另外非常喜欢的一个着色器：

```glsl
precision highp float;

uniform float               iTime;
uniform sampler2D           inputImageTexture;
uniform sampler2D           inputImageTexture2;
varying vec2                textureCoordinate;

void main()
{
	    vec2 uv = textureCoordinate.xy;
        vec4 c = texture2D(inputImageTexture, uv);
        vec4 c2 = texture2D(inputImageTexture, uv + vec2(0.005, 0.005));
        c = c - distance(c, c2);

        gl_FragColor = vec4(c.rgb, 1.0);
}
```

这个效果叫distance，着色器原理很简单，就是计算当前点和右上点的颜色的差值，再用当前颜色去减这个值，使图片边沿变深或者糊化，非常简单的代码但效果特别好。

### 制作ShaderToy滤镜

有了片段着色器，再加上顶点着色器再加上控制就可以做出滤镜了，用GpuImage的框架来实现。

顶点着色器就是传递一下坐标，代码如下：

``` Java
    public static final String NO_FILTER_VERTEX_SHADER = "" +
            "attribute vec4 position;\n" +
            "attribute vec4 inputTextureCoordinate;\n" +
            " \n" +
            "varying vec2 textureCoordinate;\n" +
            " \n" +
            "void main()\n" +
            "{\n" +
            "    gl_Position = position;\n" +
            "    textureCoordinate = inputTextureCoordinate.xy;\n" +
            "}";
```

ShaderToyFilter代码：

``` Java
public class ShaderToyFilter extends BaseOriginalFilter {

    private int[] inputTextureHandles = {-1};
    private int[] inputTextureUniformLocations = {-1};

    final long START_TIME = System.currentTimeMillis();

    public ShaderToyFilter(int shade) {
        super(NO_FILTER_VERTEX_SHADER, OpenGlUtils.readShaderFromRawResource(FilterSDK.sContext, shade));
    }

    @Override
    protected void onInitialized() {
        super.onInitialized();

        inputTextureUniformLocations[0] = GLES20.glGetUniformLocation(mGLProgId, "inputImageTexture2");
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                int iResolutionLocation = GLES20.glGetUniformLocation(mGLProgId, "iResolution");
                GLES20.glUniform3fv(iResolutionLocation, 1,
                        FloatBuffer.wrap(new float[]{(float) 1.0, (float) 1, 1.0f}));
                inputTextureHandles[0] = OpenGlUtils.loadTexture(FilterSDK.sContext, R.raw.edge_png, new int[2]);
            }
        });
    }

    @Override
    protected void onDrawArraysPre() {
        super.onDrawArraysPre();
        float time = ((float) (System.currentTimeMillis() - START_TIME)) / 1000.0f;
        int iTimeLocation = GLES20.glGetUniformLocation(mGLProgId, "iTime");
        GLES20.glUniform1f(iTimeLocation, time);

        GLES20.glActiveTexture(GLES20.GL_TEXTURE0 + (3));
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, inputTextureHandles[0]);
        GLES20.glUniform1i(inputTextureUniformLocations[0], 3);
    }
}
```

最上继承的是GpuImageFilter,弄了一张纹理图作为inputImageTexture2，少数着色器会用到，位置iResolution传的默认，iTime是时间相对值。滤镜层就这些内容了。

### 相机控制

相机部分使用api2，最新的总是更好的。

CameraFragment中的打开相机预览代码：

```Java
  private void createCameraPreviewSession() {
        try {
            SurfaceTexture texture = mCameraSurfaceView.getSurfaceTexture();
            assert texture != null;

            // We configure the size of default buffer to be the size of camera preview we want.
            texture.setDefaultBufferSize(mPreviewSize.getWidth(), mPreviewSize.getHeight());

            // This is the output Surface we need to start preview.
            Surface surface = new Surface(texture);

            // We set up a CaptureRequest.Builder with the output Surface.
            mPreviewRequestBuilder
                    = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
            mPreviewRequestBuilder.addTarget(surface);

            // Here, we create a CameraCaptureSession for camera preview.
            mCameraDevice.createCaptureSession(Arrays.asList(surface, mImageReader.getSurface()),
                    new CameraCaptureSession.StateCallback() {

                        @Override
                        public void onConfigured(@NonNull CameraCaptureSession cameraCaptureSession) {
                            // The camera is already closed
                            if (null == mCameraDevice) {
                                return;
                            }

                            // When the session is ready, we start displaying the preview.
                            mCaptureSession = cameraCaptureSession;
                            try {
                                // Auto focus should be continuous for camera preview.
                                mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_MODE,
                                        CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE);
                                // Flash is automatically enabled when necessary.
                                setAutoFlash(mPreviewRequestBuilder);

                                // Finally, we start displaying the camera preview.
                                mPreviewRequest = mPreviewRequestBuilder.build();
                                mCaptureSession.setRepeatingRequest(mPreviewRequest,
                                        mCaptureCallback, mBackgroundHandler);
                            } catch (CameraAccessException e) {
                                e.printStackTrace();
                            }
                        }

                        @Override
                        public void onConfigureFailed(
                                @NonNull CameraCaptureSession cameraCaptureSession) {
                            showToast("Failed");
                        }
                    }, null
            );
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
    }
```
一个是通过CameraSurfaceView拿到SurfaceTexture再生成Surface，再把Surface传给CameraDevice。实际数据的流向是Camera的数据传给SurfaceTexture中，CameraSurfaceView那到纹理做处理再显示出来。
打开相机通过CameraDevice创建会话createCaptureSession方法。

### 相机显示

真正显示的地方是CameraSurfaceView，看这部分代码：

```Java
public class CameraSurfaceView extends BaseSurfaceView {

    private OesTextureFilter mOesTextureFilter;

    public SurfaceTexture getSurfaceTexture() {
        return mSurfaceTexture;
    }

    private SurfaceTexture mSurfaceTexture;

    private int mRatioWidth = 0;
    private int mRatioHeight = 0;

    public CameraSurfaceView(Context context) {
        this(context, null);
    }

    public CameraSurfaceView(Context context, AttributeSet attrs) {
        super(context, attrs);
        this.getHolder().addCallback(this);
        scaleType = ScaleType.CENTER_CROP;
        gLTextureBuffer.put(TextureRotationUtil.getRotation(Rotation.NORMAL, false, true))
                .position(0);
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        super.onSurfaceCreated(gl, config);
        if (mOesTextureFilter == null)
            mOesTextureFilter = new OesTextureFilter();
        mOesTextureFilter.init();
        textureId = OpenGlUtils.getExternalOESTextureID();
        if (textureId != OpenGlUtils.NO_TEXTURE) {
            mSurfaceTexture = new SurfaceTexture(textureId);
            if (mSurfaceTextureListener != null) {
                mSurfaceTextureListener.onAvailable();
            }
            mSurfaceTexture.setOnFrameAvailableListener(onFrameAvailableListener);
        }
    }

    public void setSurfaceTextureListener(SurfaceTextureListener surfaceTextureListener) {
        mSurfaceTextureListener = surfaceTextureListener;
    }

    private SurfaceTextureListener mSurfaceTextureListener;

    public void setViewPortSize(int width, int height) {
        imageWidth = width;
        imageHeight = height;
    }

    public interface SurfaceTextureListener {
        void onAvailable();
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        super.onSurfaceChanged(gl, width, height);
    }

    protected void onFilterChanged() {
        super.onFilterChanged();
        mOesTextureFilter.onInputSizeChanged(imageWidth, imageHeight);
        mOesTextureFilter.onDisplaySizeChanged(surfaceWidth, surfaceHeight);
        mOesTextureFilter.initFrameBuffers(imageWidth, imageHeight);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        super.onDrawFrame(gl);
        if (mSurfaceTexture == null)
            return;
        mSurfaceTexture.updateTexImage();
        float[] mtx = new float[16];
        mSurfaceTexture.getTransformMatrix(mtx);
        mOesTextureFilter.setTextureTransformMatrix(mtx);
        int id;
        if (mFilter == null) {
            mOesTextureFilter.onDrawFrame(textureId, gLCubeBuffer, gLTextureBuffer);
        } else {
            id = mOesTextureFilter.onDrawToTexture(textureId);
            mFilter.onDrawFrame(id, gLCubeBuffer, gLTextureBuffer);
        }
    }

    private SurfaceTexture.OnFrameAvailableListener onFrameAvailableListener = new SurfaceTexture.OnFrameAvailableListener() {

        @Override
        public void onFrameAvailable(SurfaceTexture surfaceTexture) {
            requestRender();
        }
    };

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int width = MeasureSpec.getSize(widthMeasureSpec);
        int height = MeasureSpec.getSize(heightMeasureSpec);
        if (0 == mRatioWidth || 0 == mRatioHeight) {
            setMeasuredDimension(width, height);
        } else {
            if (width < height * mRatioWidth / mRatioHeight) {
                setMeasuredDimension(width, width * mRatioHeight / mRatioWidth);
            } else {
                setMeasuredDimension(height * mRatioWidth / mRatioHeight, height);
            }
        }
    }
}
```
OesTextureFilter是一个简单的接受外部纹理的Filter，其片段着色器如下：

``` glsl
#extension GL_OES_EGL_image_external : require
precision mediump float;
uniform samplerExternalOES sTexture;
varying vec2 textureCoordinate;

void main () {
    gl_FragColor = texture2D(sTexture, textureCoordinate);
}
```

samplerExternalOES定义扩展的纹理取样器，用于接受相机传入的纹理。

CameraSurfaceView最上继承自GLSurfaceView,先看关键onSurfaceCreated方法：

```Java
        textureId = OpenGlUtils.getExternalOESTextureID();
        if (textureId != OpenGlUtils.NO_TEXTURE) {
            mSurfaceTexture = new SurfaceTexture(textureId);
            if (mSurfaceTextureListener != null) {
                mSurfaceTextureListener.onAvailable();
            }
            mSurfaceTexture.setOnFrameAvailableListener(onFrameAvailableListener);
        }
```

在onSurfaceCreated时，创建一个OESTextureId,用于创建SurfaceTexture,mSurfaceTexture在刷新时使GLSurfaceView刷新，从而调用onDrawFrame方法：

```Java
    @Override
    public void onDrawFrame(GL10 gl) {
        super.onDrawFrame(gl);
        if (mSurfaceTexture == null)
            return;
        mSurfaceTexture.updateTexImage();
        float[] mtx = new float[16];
        mSurfaceTexture.getTransformMatrix(mtx);
        mOesTextureFilter.setTextureTransformMatrix(mtx);
        int id;
        if (mFilter == null) {
            mOesTextureFilter.onDrawFrame(textureId, gLCubeBuffer, gLTextureBuffer);
        } else {
            id = mOesTextureFilter.onDrawToTexture(textureId);
            mFilter.onDrawFrame(id, gLCubeBuffer, gLTextureBuffer);
        }
    }
```

onDrawFrame时mOesTextureFilter获取到SurfaceTexture的位置转换，保持和相机显示的一直，如果没有滤镜直接使用OesTextureFilter绘制textureId，如果有滤镜则用OesTextureFilter绘制textureId到一个新的纹理上，再使用滤镜mFilter绘制这个新的id。
BaseSurfaceView中主要是基础控制，位置坐标方向渲染模式等。

### 相机拍照

相机拍照有两种方式：
1.录屏：  使用OpenGL的readPixel方法，所见即所得，图片最大是屏幕大小，画质较差
2.调用底层拍照： 像素高，有滤镜等修改的话需要加后处理

我这里用的是方法2，主要是为了尝试新的东西，如果具体产品的要根据需求来，一般三方相机各种贴纸滤镜美颜处理用录屏的多，能所见即所得，也更快捷。

CameraFragment中请求拍照代码：

``` Java
    private void lockFocus() {
        try {
            // This is how to tell the camera to lock focus.
            mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_TRIGGER,
                    CameraMetadata.CONTROL_AF_TRIGGER_START);
            // Tell #mCaptureCallback to wait for the lock.
            mState = STATE_WAITING_LOCK;
            mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback,
                    mBackgroundHandler);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
    }
```

相机api2增加了状态，可以锁住预览然后进行拍照，api1预览，拍照，录像各是一种模式，需要切换。
拍照完成后在监听里回调：

```Java
    private final ImageReader.OnImageAvailableListener mOnImageAvailableListener
            = new ImageReader.OnImageAvailableListener() {

        @Override
        public void onImageAvailable(ImageReader reader) {
            mFile = new File(getActivity().getExternalFilesDir(null), "pic-" + System.currentTimeMillis() / 1000 + ".jpg");
            mBackgroundHandler.post(new ImageSaver(reader.acquireNextImage(), mFile, mCameraSurfaceView.getFilter()));
        }

    };
```
这里生成了一个新的File，通过reader.acquireNextImage()拿到了一个image对象，里面有我们要的原始图片，然后发一个任务去保存照片：

```Java
 private static class ImageSaver implements Runnable {

        /**
         * The JPEG image
         */
        private final Image mImage;
        /**
         * The file we save the image into.
         */
        private final File mFile;
        private final GPUImageFilter mFilter;

        ImageSaver(Image image, File file, GPUImageFilter filter) {
            mImage = image;
            mFile = file;
            mFilter = filter;
        }

        @Override
        public void run() {
            ByteBuffer buffer = mImage.getPlanes()[0].getBuffer();
            byte[] bytes = new byte[buffer.remaining()];
            buffer.get(bytes);
            if (mFilter != null) {
                Bitmap bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
                mImage.close();
                SaveUtils.saveBitmap(FilterSDK.sContext, bitmap, mFilter, mFile);
            } else {
                FileOutputStream output = null;
                try {
                    output = new FileOutputStream(mFile);
                    output.write(bytes);
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    mImage.close();
                    if (null != output) {
                        try {
                            output.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }

    }
```

解析出原图后，如果没有滤镜直接保存了，有滤镜则需要做滤镜处理再保存：

```Java
    public static void saveBitmap(Context context, Bitmap bitmap, GPUImageFilter filter, File file) {
        GPUImage gpuImage = new GPUImage(context);
        gpuImage.setFilter(filter);
        bitmap = Bitmaps.ensureBitmapSize(bitmap);
        gpuImage.setImage(bitmap);
        bitmap = gpuImage.getBitmapWithFilterApplied(true);
        FileOutputStream output = null;
        try {
            output = new FileOutputStream(file);
            bitmap.compress(Bitmap.CompressFormat.JPEG, 100, output);
            output.flush();
            bitmap.recycle();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (null != output) {
                try {
                    output.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```
使用GPUImage做滤镜处理，使用getBitmapWithFilterApplied获取到新的Bitmap。GPUImage内部用的是离屏渲染，做了一次OpenGL绘制然后ReadPixel，
OpenGL处理有大小限制，做了一次ensureBitmapSize,以前的手机经常不能超过4096，现在发现都到16384，自用的小米Note3试的。

### 源码下载

demo里有二十多种Shader的效果，还实现了图片加滤镜的效果，图片保存等。更多细节可以在github看源码，还录了一个gif图展示效果在github上，欢迎star。
<https://github.com/myandy/shadertoyCamera>

## 总结

这个demo各方面挺完整了，相机API2的使用，滤镜GPUImage架构，还翻译了二十多个ShaderToy的着色器，半年前写的，这次总结梳理下，温故而知新，也希望大家看了能有所收获。
以后博客会更新频繁些，会写更多方面的东西。
