---
title: Android系统的编舞者Choreographer
date: 2018-04-25 16:21:43
tags: [Choreographer, VSYNC]
categories: Android
description:  "Choreographer机制，用于同Vsync机制配合，实现统一调度界面绘图。
"
---

# 前言

上一篇文章 [Android的16ms和垂直同步以及三重缓存](http://dandanlove.com/2018/04/13/android-16ms/) 解释了手机流畅性的问题，并在文章中提到了在Android4.1中添加的`Vsync`。`Choreographer`机制，用于同`Vsync`机制配合，实现统一调度界面绘图。

# Choreographer的构造

Choreographer是线程级别的单例，并且具有处理当前线程消息循环队列的功能。

```java
public final class Choreographer {
    // Enable/disable vsync for animations and drawing.
    private static final boolean USE_VSYNC = SystemProperties.getBoolean(
            "debug.choreographer.vsync", true);
    
	//单例
	public static Choreographer getInstance() {
        return sThreadInstance.get();
    }

    //每个线程一个Choreographer实例
    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            return new Choreographer(looper);
        }
    };

    private Choreographer(Looper looper) {
        mLooper = looper;
        //创建handle对象，用于处理消息，其looper为当前的线程的消息队列
        mHandler = new FrameHandler(looper);
        //创建VSYNC的信号接受对象
        mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
        //初始化上一次frame渲染的时间点
        mLastFrameTimeNanos = Long.MIN_VALUE;
        //计算帧率，也就是一帧所需的渲染时间，getRefreshRate是刷新率，一般是60
        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
        //创建消息处理队列
        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
    }
}
```

> 变量USE_VSYNC用于表示系统是否是用了Vsync同步机制，该值是通过读取系统属性debug.choreographer.vsync来获取的。如果系统使用了Vsync同步机制，则创建一个FrameDisplayEventReceiver对象用于请求并接收Vsync事件，最后Choreographer创建了一个大小为3的CallbackQueue队列数组，用于保存不同类型的Callback。

# Choreographer的使用

## 注册Runnable对象

作者之前写过一篇关于**ViewRootImpl**的文章：[**ViewRootImpl**的独白，我不是一个**View**(布局篇)]([**https://www.jianshu.com/p/005afb47e00c**](https://www.jianshu.com/p/005afb47e00c))里面有涉及使用Choreographer进行View的绘制，这次我们从ViewRootImpl的绘制出发来看看Choreographer的使用。

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
    Choreographer mChoreographer;
    /***部分代码省略***/
    public ViewRootImpl(Context context, Display display) {
        /***部分代码省略***/
        mChoreographer = Choreographer.getInstance();
        /***部分代码省略***/
    }
    /***部分代码省略***/
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
        }
    }
}
```

## 注册FrameCallback对象

> 无论是注册`Runnable`还是注册`FrameCallback`对象最终都会调用`postCallbackDelayedInternal`方法往`mCallbackQueues`添加回调，区别在于`FrameCallback`的token为`FRAME_CALLBACK_TOKEN`，两者在回调的时候不相同。

```java
public final class Choreographer {
    // All frame callbacks posted by applications have this token.
    private static final Object FRAME_CALLBACK_TOKEN = new Object() {
        public String toString() { return "FRAME_CALLBACK_TOKEN"; }
    };

    private static final class CallbackRecord {
        public CallbackRecord next;
        public long dueTime;
        public Object action; // Runnable or FrameCallback
        public Object token;

        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }
    }
}
```

# Choreographer的消息处理

## Choreographer接受消息

```java
public final class Choreographer {
    //Input callback.  Runs first.
    public static final int CALLBACK_INPUT = 0;
    //Animation callback.  Runs before traversals.
    public static final int CALLBACK_ANIMATION = 1;
    // Traversal callback.  Handles layout and draw.  
    //Runs last after all other asynchronous messages have been handled.
    public static final int CALLBACK_TRAVERSAL = 2;
    private static final int CALLBACK_LAST = CALLBACK_TRAVERSAL;

    //长度为3（CALLBACK_LAST+1）的CallbackQueue类型的数组
    private final CallbackQueue[] mCallbackQueues;

    //发送回调事件
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }

    public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        if (action == null) {
            throw new IllegalArgumentException("action must not be null");
        }
        if (callbackType < 0 || callbackType > CALLBACK_LAST) {
            throw new IllegalArgumentException("callbackType is invalid");
        }

        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }

    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        /***部分代码省略***/
        synchronized (mLock) {
            //从开机到现在的毫秒数（手机睡眠的时间不包括在内）； 
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            //添加类型为callbackType的CallbackQueue（将要执行的回调封装而成）
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
            //函数执行时间
            if (dueTime <= now) {
                //立即执行
                scheduleFrameLocked(now);
            } else {
                //异步回调延迟执行
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
    /**
     * @param dueTime 任务开始时间
     * @param action 任务
     * @param token 标识
     */
    private CallbackRecord obtainCallbackLocked(long dueTime, Object action, Object token) {
        CallbackRecord callback = mCallbackPool;
        if (callback == null) {
            callback = new CallbackRecord();
        } else {
            mCallbackPool = callback.next;
            callback.next = null;
        }
        callback.dueTime = dueTime;
        callback.action = action;
        callback.token = token;
        return callback;
    }
    private final class CallbackQueue {
        private CallbackRecord mHead;
        public void addCallbackLocked(long dueTime, Object action, Object token) {
            CallbackRecord callback = obtainCallbackLocked(dueTime, action, token);
            CallbackRecord entry = mHead;
            //判断当前的是否不头节点
            if (entry == null) {
                mHead = callback;
                return;
            }
            //判断当前任务出发起始时间是不是当前所有任务的最开始时间
            if (dueTime < entry.dueTime) {
                callback.next = entry;
                mHead = callback;
                return;
            }
            //根据任务开始时间由小到大插入到链表当中
            while (entry.next != null) {
                if (dueTime < entry.next.dueTime) {
                    callback.next = entry.next;
                    break;
                }
                entry = entry.next;
            }
            entry.next = callback;
        }
    }
}
```

### CallbackQueue

```java
public final class Choreographer {
    /**
     * Callback type: Input callback.  Runs first.
     * @hide
     */
    public static final int CALLBACK_INPUT = 0;

    /**
     * Callback type: Animation callback.  Runs before traversals.
     * @hide
     */
    public static final int CALLBACK_ANIMATION = 1;

    /**
     * Callback type: Traversal callback.  Handles layout and draw.  Runs
     * after all other asynchronous messages have been handled.
     * @hide
     */
    public static final int CALLBACK_TRAVERSAL = 2;
}
```
> 三种类型不同的`CallbackRecord`链表，按照任务触发时间由小到大排列。

<center>![CallbackQueue.png](https://upload-images.jianshu.io/upload_images/1319879-3e4ccbfece97a82c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>


## FrameHandler异步处理

```java
public final class Choreographer {

    private static final int MSG_DO_FRAME = 0;
    private static final int MSG_DO_SCHEDULE_VSYNC = 1;
    private static final int MSG_DO_SCHEDULE_CALLBACK = 2;
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    //刷新当前这一帧
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    //做VSYNC的信号同步
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    //将当前任务加入执行队列
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
}
```

### doFrame

```java
public final class Choreographer {
    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; // no work to do
            }
            //当前时间
            startNanos = System.nanoTime();
            //抖动间隔
            final long jitterNanos = startNanos - frameTimeNanos;
            //抖动间隔大于屏幕刷新时间间隔（16ms）
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                //跳过了几帧！，也许当前应用在主线程做了太多的事情。
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                //最后一次的屏幕刷是lastFrameOffset之前开始的
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                if (DEBUG) {
                    Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                            + "which is more than the frame interval of "
                            + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                            + "Skipping " + skippedFrames + " frames and setting frame "
                            + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
                }
                //最后一帧的刷新开始时间
                frameTimeNanos = startNanos - lastFrameOffset;
            }
            //由于跳帧可能造成了当前展现的是之前的帧，这样需要等待下一个vsync信号
            if (frameTimeNanos < mLastFrameTimeNanos) {
                if (DEBUG) {
                    Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                            + "previously skipped frame.  Waiting for next vsync.");
                }
                scheduleVsyncLocked();
                return;
            }
            //当前画面刷新的状态置false
            mFrameScheduled = false;
            //更新最后一帧的刷新时间
            mLastFrameTimeNanos = frameTimeNanos;
        }
        //按照优先级策略进行画面刷新时间处理
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
        
        if (DEBUG) {
            final long endNanos = System.nanoTime();
            Log.d(TAG, "Frame " + frame + ": Finished, took "
                    + (endNanos - startNanos) * 0.000001f + " ms, latency "
                    + (startNanos - frameTimeNanos) * 0.000001f + " ms.");
        }
    }
}
```

### doScheduleVsync

```java
public final class Choreographer {
    //等待vsync信号
    void doScheduleVsync() {
        synchronized (mLock) {
            if (mFrameScheduled) {
                scheduleVsyncLocked();
            }
        }
    }
    //当运行在Looper线程，则立刻调度vsync
    private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }
}
```

### doScheduleCallback

```java
public final class Choreographer {
    // Enable/disable vsync for animations and drawing.
    private static final boolean USE_VSYNC = SystemProperties.getBoolean(
            "debug.choreographer.vsync", true);
    private final class CallbackQueue {
        //判断是否有能执行的任务
        public boolean hasDueCallbacksLocked(long now) {
            return mHead != null && mHead.dueTime <= now;
        }
        /***部分代码省略***/
    }
    /***部分代码省略***/
    //执行任务回调
    void doScheduleCallback(int callbackType) {
        synchronized (mLock) {
            if (!mFrameScheduled) {
                final long now = SystemClock.uptimeMillis();
                //有能执行的任务
                if (mCallbackQueues[callbackType].hasDueCallbacksLocked(now)) {
                    scheduleFrameLocked(now);
                }
            }
        }
    }

    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                if (DEBUG) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                if (isRunningOnLooperThreadLocked()) {
                    //当运行在Looper线程，则立刻调度vsync
                    scheduleVsyncLocked();
                } else {
                    //切换到主线程，调度vsync
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                //如果没有VSYNC的同步，则发送消息刷新画面
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
    //检测当前的Looper线程是不是主线程
    private boolean isRunningOnLooperThreadLocked() {
        return Looper.myLooper() == mLooper;
    }
}
```

```java
public final class Choreographer {
    // The display event receiver can only be accessed by the looper thread to which
    // it is attached.  We take care to ensure that we post message to the looper
    // if appropriate when interacting with the display event receiver.
    private final FrameDisplayEventReceiver mDisplayEventReceiver;

    private Choreographer(Looper looper) {
        /***部分代码省略***/
        //在Choreographer的构造函数中，我们使用USE_VSYNC则会有FrameDisplayEventReceiver做为与显示器时间进行交互
        mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
    }
    /***部分代码省略***/
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        //构造函数需要传入当前的looper队列  
        public FrameDisplayEventReceiver(Looper looper) {
            super(looper);
        }
        /***部分代码省略***/  
    }
}
```

```java
public abstract class DisplayEventReceiver {
    private static native void nativeScheduleVsync(long receiverPtr);
    /**
     * Creates a display event receiver.
     *
     * @param looper The looper to use when invoking callbacks.
     */
    public DisplayEventReceiver(Looper looper) {
        if (looper == null) {
            throw new IllegalArgumentException("looper must not be null");
        }
        mMessageQueue = looper.getQueue();
        //接受数量多少等于looper中消息的多少
        mReceiverPtr = nativeInit(this, mMessageQueue);
        mCloseGuard.open("dispose");
    }
    /**
     * Schedules a single vertical sync pulse to be delivered when the next
     * display frame begins.
     */
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }
}
```

## Choreographer流程汇总

<center>![choreographer.png](https://upload-images.jianshu.io/upload_images/1319879-15dd68b1b499b02c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>


# native端的消息处理

文件路径：` frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp`


## NativeDisplayEventReceiver类结构

```cpp
//NativeDisplayEventReceiver类的定义
class NativeDisplayEventReceiver : public LooperCallback {
public://对象公共方法
    //构造函数
    NativeDisplayEventReceiver(JNIEnv* env,
            jobject receiverObj, const sp<MessageQueue>& messageQueue);
    status_t initialize();  //初始化方法
    void dispose();
    status_t scheduleVsync();//获取下一个VSYNC信号

protected:
    virtual ~NativeDisplayEventReceiver();//析构函数

private:
    jobject mReceiverObjGlobal;//java层的DisplayEventReceiver的全局引用
    sp<MessageQueue> mMessageQueue;//looper的消息队列
    DisplayEventReceiver mReceiver;//frameworks/nivate/libs/gui/DisplayEventReceiver.cpp
    bool mWaitingForVsync;//默认为false

    virtual int handleEvent(int receiveFd, int events, void* data);
    bool processPendingEvents(nsecs_t* outTimestamp, int32_t* id, uint32_t* outCount);
    void dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count);
    void dispatchHotplug(nsecs_t timestamp, int32_t id, bool connected);
};

//ststem/core/include/utils/Looper.h
/**
 * A looper callback.
 */
//NativeDisplayEventReceiver的父类，用与looper中消息的回调
class LooperCallback : public virtual RefBase {
protected:
    virtual ~LooperCallback() { }

public:
    virtual int handleEvent(int fd, int events, void* data) = 0;
};
```

## NativeDisplayEventReceiver初始化

```cpp
//初始化native的消息队列
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverObj,
        jobject messageQueueObj) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }
    //构造NativeDisplayEventReceiver对象
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverObj, messageQueue);
    status_t status = receiver->initialize();
    if (status) {
        String8 message;
        message.appendFormat("Failed to initialize display event receiver.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
        return 0;
    }

    receiver->incStrong(gDisplayEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}
//NativeDisplayEventReceiver的构造函数
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env,
        jobject receiverObj, const sp<MessageQueue>& messageQueue) :
        mReceiverObjGlobal(env->NewGlobalRef(receiverObj)),
        mMessageQueue(messageQueue), mWaitingForVsync(false) {
    ALOGV("receiver %p ~ Initializing input event receiver.", this);
}
//receiver内部数据的初始化
status_t NativeDisplayEventReceiver::initialize() {
    status_t result = mReceiver.initCheck();
    if (result) {
        ALOGW("Failed to initialize display event receiver, status=%d", result);
        return result;
    }
    //监听mReceiver的所获取的文件句柄。
    int rc = mMessageQueue->getLooper()->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
            this, NULL);
    if (rc < 0) {
        return UNKNOWN_ERROR;
    }
    return OK;
}
```

## NativeDisplayEventReceiver请求VSYNC的同步

```cpp
//java层调用DisplayEventReceiver的scheduleVsync请求VSYNC的同步
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    status_t status = receiver->scheduleVsync();
    if (status) {
        String8 message;
        message.appendFormat("Failed to schedule next vertical sync pulse.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
    }
}

status_t NativeDisplayEventReceiver::scheduleVsync() {
    if (!mWaitingForVsync) {
        ALOGV("receiver %p ~ Scheduling vsync.", this);

        // Drain all pending events.
        nsecs_t vsyncTimestamp;
        int32_t vsyncDisplayId;
        uint32_t vsyncCount;
        processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount);
        //请求下一次Vsync信息处理
        status_t status = mReceiver.requestNextVsync();
        if (status) {
            ALOGW("Failed to request next vsync, status=%d", status);
            return status;
        }

        mWaitingForVsync = true;
    }
    return OK;
}

//frameworks/native/libs/gui/DisplayEventReceiver.cpp
//通过IDisplayEventConnection接口来请求Vsync信号，IDisplayEventConnection实现了Binder通信框架，可以跨进程调用。
//因为Vsync信号请求进程和Vsync产生进程有可能不在同一个进程空间，因此这里就借助IDisplayEventConnection接口来实现。
status_t DisplayEventReceiver::requestNextVsync() {
    if (mEventConnection != NULL) {
        mEventConnection->requestNextVsync();
        return NO_ERROR;
    }
    return NO_INIT;
}
```

## NativeDisplayEventReceiver处理消息

```cpp
//NativeDisplayEventReceiver处理消息
int NativeDisplayEventReceiver::handleEvent(int receiveFd, int events, void* data) {
    ...
    nsecs_t vsyncTimestamp;
    int32_t vsyncDisplayId;
    uint32_t vsyncCount;
    //过滤出最后一次的vsync
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
        mWaitingForVsync = false;
        //分发Vsync，调用到native的android/view/DisplayEventReceiver.class的dispatchVsync方法
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }
    return 1;
}
```

## DisplayEventReceiver分发VSYNC信号

```java
public abstract class DisplayEventReceiver {
    /***部分代码省略***/
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
    }
    // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
        onVsync(timestampNanos, builtInDisplayId, frame);
    }
}

private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
    private boolean mHavePendingVsync;
    private long mTimestampNanos;
    private int mFrame;
    /***部分代码省略***/
    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        //忽略来自第二显示屏的Vsync
        if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
            scheduleVsync();
            return;
        }
        /***部分代码省略***/
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        //该消息的callback为当前对象FrameDisplayEventReceiver
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        //DisplayEventReceiver消息处理
        doFrame(mTimestampNanos, mFrame);
    }
}
```

## DisplayEventReceiver消息处理

> 参见`4.2.1、doFrame`介绍

# Choreographer处理回调

## Choreographer触发可执行任务的回调

> 这里为什么说可执行任务呢？因为每个任务都有自己的触发时间，`Choreographer`只选择它能触发的任务。

```java
public final class Choreographer {
    //进行回调的标识
    private boolean mCallbacksRunning;
    /***部分代码省略***/
    void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            //找到当前能触发的回调链表
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(now);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;
        }
        try {
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                //循环遍历，回调所有的任务
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
        }
    }
    //回收回调任务资源
    private void recycleCallbackLocked(CallbackRecord callback) {
        callback.action = null;
        callback.token = null;
        callback.next = mCallbackPool;
        mCallbackPool = callback;
    }
    private final class CallbackQueue {
        public CallbackRecord extractDueCallbacksLocked(long now) {
            CallbackRecord callbacks = mHead;
            //当链表头部的任务触发事件都比当前时间晚，那么整个链表则没有任务需要触发
            if (callbacks == null || callbacks.dueTime > now) {
                return null;
            }

            CallbackRecord last = callbacks;
            CallbackRecord next = last.next;
            //找到当前时间之前需要触发任务链表，将该链表截断并返回
            while (next != null) {
                if (next.dueTime > now) {
                    last.next = null;
                    break;
                }
                last = next;
                next = next.next;
            }
            //mHead重置为原始链表截断的头部
            mHead = next;
            return callbacks;
        }
    }
}
```

## 处理Choreographer回调

`3、Choreographer的使用`部分讲述了`ViewRootImpl`使用`Choreographer`的使用，那么我们现在来看一下`ViewRootImpl`对`Choreographer`回调时间的处理。

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
    Choreographer mChoreographer;
    /***部分代码省略***/
    public ViewRootImpl(Context context, Display display) {
        /***部分代码省略***/
        mChoreographer = Choreographer.getInstance();
        /***部分代码省略***/
    }
    /***部分代码省略***/
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
        }
    }
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            //开始View的测量、布局、绘制
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
}
```

# 总结

整片文章单独看起来留下的印象不是很深刻，以前阅读过 [Android的16ms和垂直同步以及三重缓存](http://dandanlove.com/2018/04/13/android-16ms/) 这篇文章之后就会知道本文章是对  [Android的16ms和垂直同步以及三重缓存](http://dandanlove.com/2018/04/13/android-16ms/)  这篇文章其中的一些疑问进行解答。从代码的角度讲述了android的屏幕绘制部分知识。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>