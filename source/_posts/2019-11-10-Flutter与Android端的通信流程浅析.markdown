---
title:      "Flutter与Android端的通信流程浅析"
description:   "Flutter与native三种通信方式的使用与数据传递解析"
date:       2019-11-10 14:00:00
author:     "安地"
tags:
      - Android
      - Flutter
---


## 简介

Flutter 官方提供了一种 Platform Channel 的方案，用于 Dart 和平台之间相互通信。

我们主要分析下Flutter和Android的通信过程，只分析到java层代码。


## 使用

Platform Channel提供了三种封装的调用方式，用于不同情景下的使用：

1.MethodChannel 用于Flutter主动调用Native的方法，并获取相应的返回值
2.EventChannel  用于Flutter监听Native的消息，无法回复消息
3.BasicMessageChannel 用于Flutter与native双向字符串和半结构化的数据传递

### MethodChannel


#### android端

在Java代码里面添加一个MethodChannel.MethodCallHandler用于处理方法回调，dart端调用方法会调用到这里的onMethodCall，然后根据方法名和参数做对应处理即可。
把MethodCallHandler要设置到一个MethodChannel里，对应一个独一的限定名。flutterView实际是一个BinaryMessenger。

```java
public class MyMethodChannel implements MethodChannel.MethodCallHandler {

    private Activity activity;
    private MethodChannel channel;

    public static MyMethodChannel registerWith(FlutterView flutterView) {
        MethodChannel channel = new MethodChannel(flutterView, "MyMethodChannel");
        MyMethodChannel methodChannelPlugin = new MyMethodChannel((Activity) flutterView.getContext(), channel);
        channel.setMethodCallHandler(methodChannelPlugin);
        return methodChannelPlugin;
    }

    private MyMethodChannel(Activity activity, MethodChannel channel) {
        this.channel = channel;
        this.activity = activity;
    }

    @Override
    public void onMethodCall(MethodCall methodCall, MethodChannel.Result result) {
        switch (methodCall.method) {
            case "send":
                result.success("android端收到方法：" + methodCall.arguments);
                Toast.makeText(activity, methodCall.arguments + "", Toast.LENGTH_SHORT).show();
                break;
            default:
                result.notImplemented();
                break;
        }
    }
}
```
然后在FlutterActivity创建时调用registerWith就可以

```java
public class MainActivity extends FlutterActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        GeneratedPluginRegistrant.registerWith(this);
        MyMethodChannel.registerWith(getFlutterView());
    }
}
```


#### Flutter端

dart根据MethodChannel的限定名创建MethodChannel，然后调用异步方法，使用future获取回调，即java的Result。
```dart
class _MyHomePageState extends State<MyHomePage> {

  MethodChannel _methodChannel = MethodChannel("MyMethodChannel");

  void _sendToNative() {
    Future<String> future =
    _methodChannel.invokeMethod("send", _counter);
    future.then((message) {
      setState(() {
        _counter = "返回值：" + message;
      });
    });
  }
  
}

```

### EventChannel

#### android端

dart端注册后java的onListen被调用，拿到EventSink，用于发送消息到dart，有success，error，endOfStream三个方法
```java
public class MyEventChannel implements EventChannel.StreamHandler {

    private Activity activity;
    private EventChannel channel;

    public static MyEventChannel registerWith(FlutterView flutterView) {
        EventChannel channel = new EventChannel(flutterView, "MyEventChannel");
        MyEventChannel eventChannel = new MyEventChannel((Activity) flutterView.getContext(), channel);
        channel.setStreamHandler(eventChannel);
        return eventChannel;
    }

    private MyEventChannel(Activity activity, EventChannel channel) {
        this.channel = channel;
        this.activity = activity;
    }

    @Override
    public void onListen(Object o, EventChannel.EventSink eventSink) {
        activity.getWindow().getDecorView().postDelayed(new Runnable() {
            @Override
            public void run() {
                eventSink.success("success");
                eventSink.error("failed1","failed2",-1);
                eventSink.endOfStream();
            }
        }, 1000);
    }

    @Override
    public void onCancel(Object o) {

    }
}
```

#### Flutter端

使用eventChannel注册监听，监听三个方法onEvent，onError,onDone，和java的三个方法对应。
```dart
  void _onEvent(Object event) {
    _counter = event.toString();
  }
  void _onError(Object error) {
    _counter = error.toString();
  }

  void _onDone() {
    _counter ="done";
  }

  @override
  void initState() {
    super.initState();
    _counter = "init";
    EventChannel eventChannel =  EventChannel('MyEventChannel');
    eventChannel.receiveBroadcastStream().listen(_onEvent,onError:_onError,onDone: _onDone);
  }

```

### BasicMessageChannel

#### Android端

创建了一个MessageHandler用于接受消息，用BasicMessageChannel可以发送消息，发送消息可以等待回答，接受消息可以回答，得到回答后一次消息就结束了。是Android端和flutter的双向单次通信。

```java
public class MyMessageChannel implements BasicMessageChannel.MessageHandler<String> {

    private Activity activity;
    private BasicMessageChannel channel;

    public static MyMessageChannel registerWith(FlutterView flutterView) {
        BasicMessageChannel channel = new BasicMessageChannel(flutterView, "MyMessageChannel", StringCodec.INSTANCE);
        MyMessageChannel eventChannel = new MyMessageChannel((Activity) flutterView.getContext(), channel);
        channel.setMessageHandler(eventChannel);
        return eventChannel;
    }

    public void testSendMessage(){
        channel.send("给flutter发消息了");
        channel.send("给flutter发消息了，请回答", new BasicMessageChannel.Reply() {
            @Override
            public void reply(Object o) {
                Toast.makeText(activity,o.toString(),Toast.LENGTH_LONG).show();
            }
        });
    }

    private MyMessageChannel(Activity activity, BasicMessageChannel channel) {
        this.channel = channel;
        this.activity = activity;
    }

    @Override
    public void onMessage(String s, BasicMessageChannel.Reply<String> reply) {
        Toast.makeText(activity,s,Toast.LENGTH_LONG).show();
        reply.reply("知道了");
    }
}
```

#### Flutter端

```dart
    const basicMessageChannel = const BasicMessageChannel('MyMessageChannel', StringCodec());
    basicMessageChannel.setMessageHandler(
          (String message) => Future<String>(() {
        setState(() {
          _counter = message;
        });
        return "flutter:知道了";
      }),
    );

    Future<String> future= basicMessageChannel.send("来自flutter的message");
    future.then((message) {
      setState(() {
        _counter = message;
      });
    });
```

## 原理浅析

主要分析下Android端框架的事件流程，我们从EventChannel入手

### 事件注册流程

EventChannel创建需要传入一个BinaryMessenger，然后给它设置一个EventChannel.StreamHandler.setStreamHandler：

```java
    @UiThread
    public void setStreamHandler(EventChannel.StreamHandler handler) {
        this.messenger.setMessageHandler(this.name, handler == null ? null : new EventChannel.IncomingStreamRequestHandler(handler));
    }
```

设置setStreamHandler方法，用了一个IncomingStreamRequestHandler包装StreamHandler，IncomingStreamRequestHandler就是BinaryMessengerHandler类型。

```java
    public interface BinaryReply {
        @UiThread
        void reply(@Nullable ByteBuffer var1);
    }
    
    public interface BinaryMessageHandler {
        @UiThread
        void onMessage(@Nullable ByteBuffer var1, @NonNull BinaryMessenger.BinaryReply var2);
    }
```
BinaryMessageHandler就一个方法onMessage,接收消息的方法，原始数据是ByteBuffer类型，另一个参数BinaryReply接口,可以回调回复一个ByteBuffer的消息。

IncomingStreamRequestHandler代码：

```java
  private final class IncomingStreamRequestHandler implements BinaryMessageHandler {
        private final EventChannel.StreamHandler handler;
        private final AtomicReference<EventChannel.EventSink> activeSink = new AtomicReference((Object)null);

        IncomingStreamRequestHandler(EventChannel.StreamHandler handler) {
            this.handler = handler;
        }

        public void onMessage(ByteBuffer message, BinaryReply reply) {
            MethodCall call = EventChannel.this.codec.decodeMethodCall(message);
            if (call.method.equals("listen")) {
                this.onListen(call.arguments, reply);
            } else if (call.method.equals("cancel")) {
                this.onCancel(call.arguments, reply);
            } else {
                reply.reply((ByteBuffer)null);
            }

        }

        private void onListen(Object arguments, BinaryReply callback) {
            EventChannel.EventSink eventSink = new EventChannel.IncomingStreamRequestHandler.EventSinkImplementation();
            EventChannel.EventSink oldSink = (EventChannel.EventSink)this.activeSink.getAndSet(eventSink);
            if (oldSink != null) {
                try {
                    this.handler.onCancel((Object)null);
                } catch (RuntimeException var7) {
                    Log.e("EventChannel#" + EventChannel.this.name, "Failed to close existing event stream", var7);
                }
            }

            try {
                this.handler.onListen(arguments, eventSink);
                callback.reply(EventChannel.this.codec.encodeSuccessEnvelope((Object)null));
            } catch (RuntimeException var6) {
                this.activeSink.set((Object)null);
                Log.e("EventChannel#" + EventChannel.this.name, "Failed to open event stream", var6);
                callback.reply(EventChannel.this.codec.encodeErrorEnvelope("error", var6.getMessage(), (Object)null));
            }

        }

        private void onCancel(Object arguments, BinaryReply callback) {
            EventChannel.EventSink oldSink = (EventChannel.EventSink)this.activeSink.getAndSet((Object)null);
            if (oldSink != null) {
                try {
                    this.handler.onCancel(arguments);
                    callback.reply(EventChannel.this.codec.encodeSuccessEnvelope((Object)null));
                } catch (RuntimeException var5) {
                    Log.e("EventChannel#" + EventChannel.this.name, "Failed to close event stream", var5);
                    callback.reply(EventChannel.this.codec.encodeErrorEnvelope("error", var5.getMessage(), (Object)null));
                }
            } else {
                callback.reply(EventChannel.this.codec.encodeErrorEnvelope("error", "No active stream to cancel", (Object)null));
            }

        }
   }
```

onMessage分发下去三个方法，开始监听和取消监听，名称不对就立即回复空消息。
onListen中创建EventChannel.EventSink，并回调给EventChannel.StreamHandler，就是我们自己写的继承方法了，然后在reply回复操作成功，可以看出每个消息都必须有回复。

### 事件调用流程

事件调用由EventSink发起，看EventSinkImplementation：

```java
 private final class EventSinkImplementation implements EventChannel.EventSink {
            final AtomicBoolean hasEnded;

            private EventSinkImplementation() {
                this.hasEnded = new AtomicBoolean(false);
            }

            @UiThread
            public void success(Object event) {
                if (!this.hasEnded.get() && IncomingStreamRequestHandler.this.activeSink.get() == this) {
                    EventChannel.this.messenger.send(EventChannel.this.name, EventChannel.this.codec.encodeSuccessEnvelope(event));
                }
            }

            @UiThread
            public void error(String errorCode, String errorMessage, Object errorDetails) {
                if (!this.hasEnded.get() && IncomingStreamRequestHandler.this.activeSink.get() == this) {
                    EventChannel.this.messenger.send(EventChannel.this.name, EventChannel.this.codec.encodeErrorEnvelope(errorCode, errorMessage, errorDetails));
                }
            }

            @UiThread
            public void endOfStream() {
                if (!this.hasEnded.getAndSet(true) && IncomingStreamRequestHandler.this.activeSink.get() == this) {
                    EventChannel.this.messenger.send(EventChannel.this.name, (ByteBuffer)null);
                }
            }
        }
```

所有的事件发送都由BinaryMessenger转发，由MethodCodec编码成ByteBuffer类型数据。
BinaryMessenger实际是FlutterView：

```java
    @UiThread
    public void send(String channel, ByteBuffer message) {
        this.send(channel, message, (BinaryReply)null);
    }
    @UiThread
    public void send(String channel, ByteBuffer message, BinaryReply callback) {
        if (!this.isAttached()) {
            Log.d("FlutterView", "FlutterView.send called on a detached view, channel=" + channel);
        } else {
            this.mNativeView.send(channel, message, callback);
        }
    }
```

FlutterNativeView：

```java
    @UiThread
    public void send(String channel, ByteBuffer message, BinaryReply callback) {
        if (!this.isAttached()) {
            Log.d("FlutterNativeView", "FlutterView.send called on a detached view, channel=" + channel);
        } else {
            this.dartExecutor.send(channel, message, callback);
        }
    }
```

DartExecutor:
```java
    @UiThread
    public void send(@NonNull String channel, @Nullable ByteBuffer message, @Nullable BinaryReply callback) {
        this.messenger.send(channel, message, callback);
    }
```

DartMessenger：

```java
    @UiThread
    public void send(@NonNull String channel, @NonNull ByteBuffer message) {
        Log.v("DartMessenger", "Sending message over channel '" + channel + "'");
        this.send(channel, message, (BinaryReply)null);
    }

    public void send(@NonNull String channel, @Nullable ByteBuffer message, @Nullable BinaryReply callback) {
        Log.v("DartMessenger", "Sending message with callback over channel '" + channel + "'");
        int replyId = 0;
        if (callback != null) {
            replyId = this.nextReplyId++;
            this.pendingReplies.put(replyId, callback);
        }

        if (message == null) {
            this.flutterJNI.dispatchEmptyPlatformMessage(channel, replyId);
        } else {
            this.flutterJNI.dispatchPlatformMessage(channel, message, message.position(), replyId);
        }

    }
```
DartMessenger设置replayId,直接线性递增计数，BinaryReply会保存在map结构pendingReplies中，replyId和BinaryReply一一绑定。

EventChannel是单向通信，不需要BinaryReply，可以猜测MethodChannel和BaseMessageChannel需要用到BinaryReply，BinaryReply在哪里创建的呢？

DratMessage接受消息：

```java
    public void handleMessageFromDart(@NonNull String channel, @Nullable byte[] message, int replyId) {
        Log.v("DartMessenger", "Received message from Dart over channel '" + channel + "'");
        BinaryMessageHandler handler = (BinaryMessageHandler)this.messageHandlers.get(channel);
        if (handler != null) {
            try {
                Log.v("DartMessenger", "Deferring to registered handler to process message.");
                ByteBuffer buffer = message == null ? null : ByteBuffer.wrap(message);
                handler.onMessage(buffer, new DartMessenger.Reply(this.flutterJNI, replyId));
            } catch (Exception var6) {
                Log.e("DartMessenger", "Uncaught exception in binary message listener", var6);
                this.flutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
            }
        } else {
            Log.v("DartMessenger", "No registered handler for message. Responding to Dart with empty reply message.");
            this.flutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
        }

    }

    public void handlePlatformMessageResponse(int replyId, @Nullable byte[] reply) {
        Log.v("DartMessenger", "Received message reply from Dart.");
        BinaryReply callback = (BinaryReply)this.pendingReplies.remove(replyId);
        if (callback != null) {
            try {
                Log.v("DartMessenger", "Invoking registered callback for reply from Dart.");
                callback.reply(reply == null ? null : ByteBuffer.wrap(reply));
            } catch (Exception var5) {
                Log.e("DartMessenger", "Uncaught exception in binary message reply handler", var5);
            }
        }

    }
```
接受dart消息时会创建DartMessenger.Reply，然后回调给BinaryMessageHandler。处理回调消息时根据replayId移除BinaryReply，然后再回调reply方法，所以reply是能且仅能调用一次。

Reply代码：
```java
    private static class Reply implements BinaryReply {
        @NonNull
        private final FlutterJNI flutterJNI;
        private final int replyId;
        private final AtomicBoolean done = new AtomicBoolean(false);

        Reply(@NonNull FlutterJNI flutterJNI, int replyId) {
            this.flutterJNI = flutterJNI;
            this.replyId = replyId;
        }

        public void reply(@Nullable ByteBuffer reply) {
            if (this.done.getAndSet(true)) {
                throw new IllegalStateException("Reply already submitted");
            } else {
                if (reply == null) {
                    this.flutterJNI.invokePlatformMessageEmptyResponseCallback(this.replyId);
                } else {
                    this.flutterJNI.invokePlatformMessageResponseCallback(this.replyId, reply, reply.position());
                }

            }
        }
    }
```

通过AtomicBoolean使reply只能调用一次，多次调用会抛出"Reply already submitted"异常。

FlutterJNI：
```java
    @UiThread
    public void dispatchPlatformMessage(@NonNull String channel, @Nullable ByteBuffer message, int position, int responseId) {
        this.ensureRunningOnMainThread();
        if (this.isAttached()) {
            this.nativeDispatchPlatformMessage(this.nativePlatformViewId, channel, message, position, responseId);
        } else {
            Log.w("FlutterJNI", "Tried to send a platform message to Flutter, but FlutterJNI was detached from native C++. Could not send. Channel: " + channel + ". Response ID: " + responseId);
        }
    }
    
    private native void nativeDispatchPlatformMessage(long var1, @NonNull String var3, @Nullable ByteBuffer var4, int var5, int var6);

  // Called by native.
  @SuppressWarnings("unused")
  private void handlePlatformMessage(@NonNull final String channel, byte[] message, final int replyId) {
    if (platformMessageHandler != null) {
      platformMessageHandler.handleMessageFromDart(channel, message, replyId);
    }
  }
```

看到通过层层转调后到native方法nativeDispatchPlatformMessage，参数分别是还是channel名称，ByteBuffer数据，数据长度，回答id。

### 总结

每个消息都有一个来回的传递，完成一个闭环。先分析从dart到android的调用，EventChannel的启动监听就是一个（BasicMessageChannel的dart消息，MethodChannel的方法调用也是），我们分析其流程：

1.Dart调用listen方法通过 Platform Channel 机制调用到FlutterJNI的handlePlatformMessage，并随赠一个replyId
2.FlutterJNI调用DartMessenger的handleMessageFromDart
3.DartMessenger创建reply并转发到BinaryMessageHandler
4.IncomingStreamRequestHandler接受到listen消息创建EventSink，reply回复成功或者失败
5.Reply通过FlutterJNI回复消息给Platform Channel，对应之前的replyId


EventChannel发送消息不需要回复，MethodChannel不能发送消息（只能接受方法调用然后回复），BasicMessageChannel可以发送消息并等待回复，分析其流程：

1.BasicMessageChannel发送消息,需要回调就创建一个IncomingReplyHandler的BinaryReply
2.FlutterView发送消息，层层向下转发到DartMessenger
3.DartMessenger发送消息，有BinaryReply就replyId自增并绑定BinaryReply到map中
4.flutterJNI发送消息到Platform Channel并随带replyId
5.Dart层回复通过Platform Channel调用到FlutterJNI的handlePlatformMessageResponse返回replyId
6.DartMessenger收到消息回复则移除replyId，调用BinaryReply的回复方法


两套流程里面都有replyId，但两者却是不冲突的，replyId由发送者创建，每次通信都有方向，只能回复一次，所以不会有id冲突问题。android发送的replyId和dart发送的replyId可以一样，但底层知道方向，维持了两套，不会冲突。

