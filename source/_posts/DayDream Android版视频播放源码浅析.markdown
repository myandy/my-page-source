---
title:      "DayDream Android SDK视频播放源码浅析"
description:   "上一篇讲到Android中VR播放方式，最后就用到了DayDream（GVR） Android SDK中的VrVideoView来进行视频播放，现在来简要分析下其源码。"
date:       2016-10-14 12:00:00
author:     "安地"
tags: [Android]
    
---

## 分析


上一篇讲到Android中VR播放方式，最后就用到了DayDream（GVR） Android SDK中的VrVideoView来进行视频播放，现在来简要分析下其源码。



### VrVideoView

    public class VrVideoView extends VrWidgetView {
        private static final String TAG = VrVideoView.class.getSimpleName();
        private static final boolean DEBUG = false;
        //内部播放器
        private VrVideoPlayerInternal videoPlayer;
        //Vr渲染器
        private VrVideoRenderer renderer;
    
        public VrVideoView(Context context, AttributeSet attrs) {
            super(context, attrs);
        }
    
        public VrVideoView(Context context) {
            super(context);
        }
    
       /**
         *
          创建渲染器，先初始化播放器和渲染器，然后把播放器获取到的元数据设入渲染器的球形数据中
        */
        protected VrVideoRenderer createRenderer(Context context, GLThreadScheduler glThreadScheduler, float xMetersPerPixel, float yMetersPerPixel, int screenRotation) {
            this.videoPlayer = new VrVideoPlayerInternal(context);
            this.renderer = new VrVideoRenderer(this.videoPlayer, this.getContext(), glThreadScheduler, xMetersPerPixel, yMetersPerPixel, screenRotation);
            this.videoPlayer.setVideoTexturesListener(new VideoTexturesListener() {
                public void onVideoTexturesReady() {
                    VrVideoView.this.renderer.setSphericalMetadata(VrVideoView.this.videoPlayer.getMetadata());
                }
            });
            return this.renderer;
        }
    
        // 下面是一些播放控制，通过播放器去实现
        public void pauseRendering() {
            super.pauseRendering();
            this.pauseVideo();
        }
    
        public void loadVideo(Uri uri, VrVideoView.Options options) throws IOException {
            this.videoPlayer.openUri(new Uri[]{uri}, options);
        }
    
        public void loadVideoFromAsset(String filename, VrVideoView.Options options) throws IOException {
            this.videoPlayer.openAsset(new String[]{filename}, options);
        }
        
        //设置播放监听，在videoPlayer中实现，这个后续会讲
        public void setEventListener(VrVideoEventListener eventListener) {
            super.setEventListener(eventListener);
            this.videoPlayer.setEventListener(eventListener);
        }
    
        public void playVideo() {
            VrVideoPlayerInternal var1 = this.videoPlayer;
            synchronized(this.videoPlayer) {
                this.videoPlayer.getExoPlayer().setPlayWhenReady(true);
            }
        }
    
        public void pauseVideo() {
            VrVideoPlayerInternal var1 = this.videoPlayer;
            synchronized(this.videoPlayer) {
                this.videoPlayer.getExoPlayer().setPlayWhenReady(false);
            }
        }
    
        public void seekTo(long positionMs) {
            VrVideoPlayerInternal var3 = this.videoPlayer;
            synchronized(this.videoPlayer) {
                this.videoPlayer.getExoPlayer().seekTo(positionMs);
            }
        }
    
        public long getDuration() {
            VrVideoPlayerInternal var3 = this.videoPlayer;
            synchronized(this.videoPlayer) {
                long duration = this.videoPlayer.getExoPlayer().getDuration();
                return duration;
            }
        }
    
        public long getCurrentPosition() {
            VrVideoPlayerInternal var3 = this.videoPlayer;
            synchronized(this.videoPlayer) {
                long position = this.videoPlayer.getExoPlayer().getCurrentPosition();
                return position;
            }
        }
    
        //一些设置选项， inputFormat为流格式，inputType为视频格式
        public static class Options {
            //开始标记
            private static final int FORMAT_START_MARKER = 0;
            //默认格式，MP4等
            public static final int FORMAT_DEFAULT = 1;
            //HLS (HTTP Live Streaming)直播流
            public static final int FORMAT_HLS = 2;
            //结束标记
            private static final int FORMAT_END_MARKER = 3;
            public int inputFormat = 1;
            private static final int TYPE_START_MARKER = 0;
            
            //全景
            public static final int TYPE_MONO = 1;
            //上下立体
            public static final int TYPE_STEREO_OVER_UNDER = 2;
            private static final int TYPE_END_MARKER = 3;
            public int inputType = 1;
    
            public Options() {
            }
    
            void validate() {
                String var10000;
                int var1;
                if(this.inputFormat <= 0 || this.inputFormat >= 3) {
                    var10000 = VrVideoView.TAG;
                    var1 = this.inputFormat;
                    Log.e(var10000, (new StringBuilder(40)).append("Invalid Options.inputFormat: ").append(var1).toString());
                    this.inputFormat = 1;
                }
    
                if(this.inputType <= 0 || this.inputFormat >= 3) {
                    var10000 = VrVideoView.TAG;
                    var1 = this.inputType;
                    Log.e(var10000, (new StringBuilder(38)).append("Invalid Options.inputType: ").append(var1).toString());
                    this.inputType = 1;
                }
    
            }
        }
    }

播放调用非常简单，设置好Options后加载视频然后播放就行了，这里播放会在准备好时播放，所以顺序无所谓。

        public void loadOnLineNormalVrVideo(String path) throws IOException {
            VrVideoView.Options mOptions = new Options();
            mOptions.inputFormat = Options.FORMAT_DEFAULT;
            mOptions.inputType = Options.TYPE_MONO;
            playVideo();
            loadVideo(Uri.parse(path), mOptions);
        }

VrVideoView还是比较简单的，不过要注意一下周期控制，对播放器做对应操作就行了，看官方demo就很清楚，不再细说了。如果只是使用的话看到这里就够了，下面继续分析播放器。

### 播放器VrVideoPlayerInternal

    class VrVideoPlayerInternal implements Listener {
        private static final boolean DEBUG = false;
        private static final int MAX_NUM_RENDERERS = 3;
        private static final String TAG;
        private Context context;
        private VrVideoEventListener eventListener;
        protected boolean isLooping;
        private boolean isPreparing;
        private SphericalMetadata metadata;
        //真正的播放器
        protected ExoPlayer player;
        //渲染队列
        private TrackRenderer[] videoRenderers;
        //纹理队列
        private VideoTexture[] videoTextures;
        private VideoTexturesListener videoTexturesListener;
    
        //视频帧刷新通知器，被调用后post到主线程，由eventListener回调出去，用于刷新视频播放器进度状态
        class NewFrameNotifier implements Runnable {
            private Handler mainHandler;
    
            NewFrameNotifier() {
                this.mainHandler = new Handler(Looper.getMainLooper());
            }
           
            public void onNewFrame() {
                this.mainHandler.post(this);
            }
    
            public void run() {
                if (VrVideoPlayerInternal.this.eventListener != null) {
                    VrVideoPlayerInternal.this.eventListener.onNewFrame();
                }
            }
        }
        
        //视频状态监听器，监听ExoPlayer，又回调给eventListener
        private class VideoLooperListener implements ExoPlayer.Listener {
            private VideoLooperListener() {
            }
    
            public void onPlayerError(ExoPlaybackException error) {
                Log.e(TAG, hashCode() + ".onPlayerError", error);
                if (VrVideoPlayerInternal.this.eventListener != null) {
                    VrVideoPlayerInternal.this.eventListener.onLoadError(error.getMessage());
                }
            }
    
            public void onPlayerStateChanged(boolean playWhenReady, int playbackState) {
                if (playbackState == 2) {
                    VrVideoPlayerInternal.this.isPreparing = true;
                } else if (playbackState == 4) {
                    if (VrVideoPlayerInternal.this.isPreparing && VrVideoPlayerInternal.this.eventListener != null) {
                        VrVideoPlayerInternal.this.isPreparing = DEBUG;
                        VrVideoPlayerInternal.this.eventListener.onLoadSuccess();
                    }
                } else if (playWhenReady && playbackState == 5) {
                    if (VrVideoPlayerInternal.this.eventListener != null) {
                        VrVideoPlayerInternal.this.eventListener.onCompletion();
                    }
                    if (VrVideoPlayerInternal.this.isLooping) {
                        synchronized (VrVideoPlayerInternal.this) {
                            VrVideoPlayerInternal.this.player.seekTo(0);
                        }
                    }
                }
            }
    
            public void onPlayWhenReadyCommitted() {
            }
        }
    
        public static interface VideoTexturesListener {
            void onVideoTexturesReady();
        }
    
        static {
            TAG = VrVideoPlayerInternal.class.getSimpleName();
        }
    
        //新建渲染队列和纹理队列
        public VrVideoPlayerInternal(Context context) {
            this.videoRenderers = new TrackRenderer[0];
            this.videoTextures = new VideoTexture[0];
            this.isPreparing = false;
            init(context);
        }
    
        protected VrVideoPlayerInternal() {
            this.videoRenderers = new TrackRenderer[0];
            this.videoTextures = new VideoTexture[0];
            this.isPreparing = false;
        }
        
        //初始化，工厂模式新建Exo播放器，设入最大渲染数（3）
        protected void init(Context context) {
            this.context = context;
            this.player = Factory.newInstance(MAX_NUM_RENDERERS, 0, 0);
            this.player.addListener(new VideoLooperListener());
            this.player.setPlayWhenReady(true);
        }
        
        //加载视频到真正的播放器
        private void loadVideoIntoPlayer(Uri[] uris, Options options) {
            //选项没有设置就new一个，有就验证下，确保合法
            if (options == null) {
                options = new Options();
            } else {
                options.validate();
            }
            if (uris.length > 2) {
                Log.e(TAG, "Try to load more streams than we support. Maximum is 2");
                return;
            }
            //通过选项选择渲染器创建者
            RendererBuilder builder;
            if (options.inputFormat == 2) {
                if (uris.length > 1) {
                    Log.w(TAG, "Only first Uri will be used with HLS format.");
                }
                builder = new HlsRendererBuilder(this.context, uris[0]);
            } else {
                builder = new ExtractorRendererBuilder(this.context, uris);
            }
            //VrVideoPlayerInternal实现了RendererBuilder的listener，把this放入创建RendererBuilder，完成后回调下面的onRenderersReady方法
            builder.buildRenderers(this);
        }
        
        //渲染器准备好的回调
        public synchronized void onRenderersReady(TrackRenderer audioRenderer, TrackRenderer[] videoRenderers) {
            int i;
            this.videoRenderers = new TrackRenderer[videoRenderers.length];
            System.arraycopy(videoRenderers, 0, this.videoRenderers, 0, videoRenderers.length);
            TrackRenderer[] allRenderers = new TrackRenderer[3];
            allRenderers[0] = audioRenderer;
            System.arraycopy(videoRenderers, 0, allRenderers, 1, videoRenderers.length);
            for (i = videoRenderers.length + 1; i < 3; i++) {
                allRenderers[i] = new DummyTrackRenderer();
            }
            //准备所有渲染器
            this.player.prepare(allRenderers);
            //新建纹理
            this.videoTextures = new VideoTexture[videoRenderers.length];
            for (i = 0; i < videoRenderers.length; i++) {
                this.videoTextures[i] = new VideoTexture();
            }
            //设置帧刷新通知
            this.videoTextures[0].setFrameNotifier(new NewFrameNotifier());
            if (this.videoTexturesListener != null) {
                this.videoTexturesListener.onVideoTexturesReady();
            }
        }
        
        //渲染器准备失败的回调
        public void onRenderersError(String errorMessage) {
            Log.e(TAG, new StringBuilder(String.valueOf(errorMessage).length() + 29).append(hashCode()).append(".onRenderersError ").append(errorMessage).toString());
            if (this.eventListener != null) {
                this.eventListener.onLoadError(errorMessage);
            }
        }
        
        //设置eventListener，在VrVideoView中调用的
        public void setEventListener(VrVideoEventListener eventListener) {
            this.eventListener = eventListener;
        }
    
        //下面两个方法都是打开播放源，最后调用前面的loadVideoIntoPlayer
        
        public void openAsset(String[] filenames, Options options) throws IOException {
            this.metadata = new SphericalMetadata();
            if (options != null) {
                this.metadata = createMetadataFromOptions(options);
            } else {
                this.metadata = parseMetadataFromVideoInputStream(this.context.getAssets().open(filenames[0]));
            }
            Uri[] videoUris = new Uri[filenames.length];
            for (int i = 0; i < filenames.length; i++) {
                String str = "file:///android_asset/";
                String valueOf = String.valueOf(filenames[i]);
                videoUris[i] = Uri.parse(valueOf.length() != 0 ? str.concat(valueOf) : new String(str));
            }
            loadVideoIntoPlayer(videoUris, options);
        }
    
        public void openUri(Uri[] uris, Options options) throws IOException {
            this.metadata = new SphericalMetadata();
            if (options != null) {
                this.metadata = createMetadataFromOptions(options);
            } else {
                String scheme = uris[0].getScheme();
                if (scheme == null || !scheme.startsWith("http")) {
                    this.metadata = parseMetadataFromVideoInputStream(new FileInputStream(uris[0].getPath()));
                }
            }
            loadVideoIntoPlayer(uris, options);
        }
    
        public SphericalMetadata getMetadata() {
            return this.metadata;
        }
    
        public byte[] getMetadataBytes() {
            return SphericalMetadata.toByteArray(this.metadata);
        }
    
        //绑定纹理，在渲染器中调用的
        public synchronized int[] bindTexture() {
            int[] textureIds;
            if (this.videoTextures.length == 0) {
                throw new IllegalStateException("openXXX() should be called successfully first.");
            }
            textureIds = new int[this.videoTextures.length];
            for (int i = 0; i < this.videoTextures.length; i++) {
                //从纹理队列取出一个纹理
                VideoTexture videoTexture = this.videoTextures[i];
                //没有初始化就初始化
                if (!videoTexture.getIsTextureSet()) {
                    videoTexture.init();
                }
                //播放器刷新纹理，并且进度跳转
                this.player.sendMessage(this.videoRenderers[i], 1, new Surface(videoTexture.getSurfaceTexture()));
                this.player.seekTo(this.player.getCurrentPosition() + 1);
                //设置纹理Id
                textureIds[i] = videoTexture.getTextureId();
            }
            //返回纹理Id列表
            return textureIds;
        }
        
        //准备帧，也是在渲染器中调用的
        public synchronized boolean prepareFrame() {
            boolean isReady;
            int i = 0;
            synchronized (this) {
                if (this.videoTextures.length > 0) {
                    isReady = true;
                } else {
                    isReady = false;
                }
                VideoTexture[] videoTextureArr = this.videoTextures;
                int length = videoTextureArr.length;
                while (i < length) {
                    VideoTexture videoTexture = videoTextureArr[i];
                    if (videoTexture.getIsTextureSet()) {
                        videoTexture.updateTexture();
                    } else {
                        isReady = DEBUG;
                    }
                    i++;
                }
            }
            return isReady;
        }
        
        //视图断开后，把渲染器和纹理释放掉
        public synchronized void onViewDetach() {
            for (int i = 0; i < this.videoRenderers.length; i++) {
                this.player.blockingSendMessage(this.videoRenderers[i], 1, null);
                this.videoTextures[i].release();
            }
        }
    
        //关闭，也要释放纹理列表
        public synchronized void shutdown() {
            this.player.stop();
            this.player.release();
            for (VideoTexture videoTexture : this.videoTextures) {
                videoTexture.release();
            }
        }
    
        public synchronized long getCurrentPositionMs() {
            return this.player.getCurrentPosition();
        }
    
        public ExoPlayer getExoPlayer() {
            return this.player;
        }
    
        public void setVideoTexturesListener(VideoTexturesListener videoTexturesListener) {
            this.videoTexturesListener = videoTexturesListener;
        }
        
        //新建metadata，根据option设置全景还是上下立体
        private static SphericalMetadata createMetadataFromOptions(Options options) {
            SphericalMetadata metadata = new SphericalMetadata();
            switch (options.inputType) {
                case SphericalMetadata.STEREO_FRAME_LAYOUT_MONO /*1*/:
                    metadata.frameLayoutMode = 1;
                    break;
                case SphericalMetadata.STEREO_FRAME_LAYOUT_TOP_BOTTOM /*2*/:
                    metadata.frameLayoutMode = 2;
                    break;
                default:
                    throw new IllegalArgumentException("Unexpected options.inputType " + options.inputType);
            }
            return metadata;
        }
        
        //根据视频流转换出球面metadata，在打开播放源时调用的
        private static SphericalMetadata parseMetadataFromVideoInputStream(InputStream videoInputStream) throws IOException {
            SphericalMetadata sphericalMetadata = new SphericalMetadata();
            sphericalMetadata = SphericalMetadataParser.parse(SphericalMetadataMP4.extract(videoInputStream));
            videoInputStream.close();
            return sphericalMetadata;
        }
    }
    
看完VrVideoPlayerInternal的代码，都很清晰了，打开播放源，管理纹理和渲染，设入真正的播放器ExoPlayer，回调事件监听方法等。再看一下最重要的渲染器。

### 渲染器VrVideoRenderer

    class VrVideoRenderer extends VrWidgetRenderer {
        private static final String TAG = VrVideoRenderer.class.getSimpleName();
        private static final boolean DEBUG = false;
        private VrVideoPlayerInternal player;
        private SphericalMetadata metadata;
        private float frameRate;
        private int elapsedFramesSinceMeasurement;
        private long lastFrameTimeMs;
        private volatile VrVideoRenderer.LoadVideoRequest lastVideoRequest;
    
        public VrVideoRenderer(VrVideoPlayerInternal player, Context context, GLThreadScheduler glThreadScheduler, float xMetersPerPixel, float yMetersPerPixel, int screenRotation) {
            super(context, glThreadScheduler, xMetersPerPixel, yMetersPerPixel, screenRotation);
            this.player = player;
            System.loadLibrary("pano_video_renderer");
        }
        
        //设置元数据，在GL线程中执行LoadVideoRequest
        public void setSphericalMetadata(SphericalMetadata metadata) {
            this.lastVideoRequest = new VrVideoRenderer.LoadVideoRequest(metadata);
            this.postApiRequestToGlThread(this.lastVideoRequest);
        }
        
        //一帧绘制后更新时间
        public void onDrawFrame(GL10 gl) {
            if(this.player.prepareFrame()) {
                super.onDrawFrame(gl);
            }
    
            if(this.lastFrameTimeMs == 0L) {
                this.lastFrameTimeMs = SystemClock.elapsedRealtime();
            }
    
            ++this.elapsedFramesSinceMeasurement;
        }
    
        public void shutdown() {
            this.player.shutdown();
            super.shutdown();
        }
    
        protected void onViewDetach() {
            this.player.onViewDetach();
        }
    
        public void onSurfaceCreated(GL10 gl, EGLConfig config) {
            super.onSurfaceCreated(gl, config);
            if(this.lastVideoRequest != null) {
                this.executeApiRequestOnGlThread(this.lastVideoRequest);
            }
    
        }
    
        float getCurrentFramerate() {
            return this.frameRate;
        }
    
        protected native long nativeCreate(ClassLoader var1, Context var2, float var3);
    
        protected native void nativeResize(long var1, int var3, int var4, float var5, float var6, int var7);
    
        protected native void nativeRenderFrame(long var1);
    
        protected native void nativeSetStereoMode(long var1, boolean var3);
    
        protected native void nativeDestroy(long var1);
    
        protected native void nativeOnPause(long var1);
    
        protected native void nativeOnResume(long var1);
    
        protected native void nativeOnPanningEvent(long var1, float var3, float var4);
    
        protected native void nativeGetHeadRotation(long var1, float[] var3);
    
        private native void nativeSetSphericalMetadata(long var1, byte[] var3);
    
        private native long nativeSetVideoTexture(long var1, int[] var3);
    
        //调用的native方法去获取渲染器，然后调用播放器的bindTexture
        private class LoadVideoRequest implements ApiRequest {
            private final SphericalMetadata metadata;
    
            public LoadVideoRequest(SphericalMetadata metadata) {
                this.metadata = metadata;
            }
    
            public void execute() {
                VrVideoRenderer.access$100(VrVideoRenderer.this, VrVideoRenderer.this.getNativeRenderer(), SphericalMetadata.toByteArray(this.metadata));
                VrVideoRenderer.access$400(VrVideoRenderer.this, VrVideoRenderer.this.getNativeRenderer(), VrVideoRenderer.this.player.bindTexture());
            }
        }
    }
    
VrVideoRenderer继承了VrWidgetRenderer，VrWidgetRenderer继承了Renderer，里面都比较简单就不说了。VR显示的关键部分就是这里了，可惜关键代码写在native层，没法解析了。

## 总结

播放视图 VrVideoView ->播放器 VrVideoPlayerInternal ->渲染器 VrVideoRenderer ->渲染器底层实现 pano_video_renderer.so

差不多一个这样的流程，其中播放器底层由ExoPlayer实现，ExoPlayer是google开源的播放器，介于现有MediaPlayer和自定义媒体播放器之间的。VrVideoView只支持VR播放，可以自己写增加一个普通渲染器支持非VR的播放。