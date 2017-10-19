---
title: linkToDeath机制了解和使用
date: 2016-12-21 14:25:00
tags: [Binder,AIDL]
categories: Android
description: "往往是由于服务端进程意外停止了，这时我们需要重新连接服务。
那么我们可以使用linkToDeath机制，如果使用bindService那么还可以通过ServiceConnection.onServiceDisconnected方法进行重连。"
---

在学习Binder和AIDL的过程中遇到的一些有意思的事情~！
linkToDeath机制，我们先看看官网如何介绍：
> When working with remote objects, you often want to find out when they are no longer valid. There are three ways this can be determined:
The [transact()](https://developer.android.com/reference/android/os/IBinder.html#transact(int, android.os.Parcel, android.os.Parcel, int))
 method will throw a [RemoteException](https://developer.android.com/reference/android/os/RemoteException.html)
 exception if you try to call it on an IBinder whose process no longer exists.
The [pingBinder()](https://developer.android.com/reference/android/os/IBinder.html#pingBinder())
 method can be called, and will return false if the remote process no longer exists.
The [linkToDeath()](https://developer.android.com/reference/android/os/IBinder.html#linkToDeath(android.os.IBinder.DeathRecipient, int))
 method can be used to register a [IBinder.DeathRecipient](https://developer.android.com/reference/android/os/IBinder.DeathRecipient.html)
 with the IBinder, which will be called when its containing process goes away.

总结：我们可以通过三种方式来检测远程对象是否存活。
* 调用远程方法的时候捕获RemoteException(DeadObjectException)；
* 调用IBinder的pingBinder()进行检测；
* 实现IBinder.DeathRecipient接口回调；

Binder意外中断
====
往往是由于服务端进程意外停止了，这时我们需要重新连接服务。
那么我们可以使用linkToDeath机制，如果使用bindService那么还可以通过ServiceConnection.onServiceDisconnected方法进行重连。

捕获RemoteException
----
在调用远程服务的时候，如果服务挂掉，那么我们客户端会接受到抛出的RemoteException异常，监听该异常进行处理。
```java
android.os.DeadObjectException
	at android.os.BinderProxy.transact(Native Method)
	at com.tzx.aidlinout.aidl.IBookManager$Stub$Proxy.addInBook(IBookManager.java:159)
	at com.tzx.aidlinout.MainActivity.onClick(MainActivity.java:117)
	at android.view.View.performClick(View.java:3514)
	at android.view.View$PerformClick.run(View.java:14125)
	at android.os.Handler.handleCallback(Handler.java:605)
	at android.os.Handler.dispatchMessage(Handler.java:92)
	at android.os.Looper.loop(Looper.java:137)
	at android.app.ActivityThread.main(ActivityThread.java:4439)
	at java.lang.reflect.Method.invokeNative(Native Method)
	at java.lang.reflect.Method.invoke(Method.java:511)
	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:787)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:554)
	at dalvik.system.NativeStart.main(Native Method)
```

pingBinder进行检测
----
pingBinder()方法会返回当前远程服务的状态（true|false）

IBinder.DeathRecipient
----
实现了IBinder.DeathRecipient接口的参数调用linkToDeath()方法，可以在binderDied方法中处理中断逻辑。
```java
binder.linkToDeath(new IBinder.DeathRecipient() {
	@Override
	public void binderDied() {
		Log.d("binder", "binderDied calling~!");
	}
}, 0);
```

总结
====
先看下边的log，从中我们可以总结出以上4中Binder中断处理方法的执行顺序：
> linkToDeath > onServiceDisconnected > pingBinder > transact()
```java
//调用远程服务
binder.pingBinder = true
Binder Thread #2	//远程服务的当前线程名称
//kill远程服务存在的进程
binderDied calling~!
onServiceDisconnected
binder.pingBinder = false
android.os.DeadObjectException
	at android.os.BinderProxy.transact(Native Method)
	at com.tzx.aidlinout.aidl.IBookManager$Stub$Proxy.addInBook(IBookManager.java:159)
	at com.tzx.aidlinout.MainActivity.onClick(MainActivity.java:117)
	at android.view.View.performClick(View.java:3514)
	at android.view.View$PerformClick.run(View.java:14125)
	at android.os.Handler.handleCallback(Handler.java:605)
	at android.os.Handler.dispatchMessage(Handler.java:92)
	at android.os.Looper.loop(Looper.java:137)
	at android.app.ActivityThread.main(ActivityThread.java:4439)
	at java.lang.reflect.Method.invokeNative(Native Method)
	at java.lang.reflect.Method.invoke(Method.java:511)
	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:787)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:554)
	at dalvik.system.NativeStart.main(Native Method)
```

扩展
====
这里讲一个Android源码中的类：RemoteCallbackList。
它在内部对列表中的每一个数据实现了Callback。而Callback实现了IBinder.DeathRecipient接口。
```java
public class RemoteCallbackList<E extends IInterface> {
	ArrayMap<IBinder, Callback> mCallbacks
            = new ArrayMap<IBinder, Callback>();
	private Object[] mActiveBroadcast;
    private int mBroadcastCount = -1;
    private boolean mKilled = false;

    private final class Callback implements IBinder.DeathRecipient {
        final E mCallback;
        final Object mCookie;
        
        Callback(E callback, Object cookie) {
            mCallback = callback;
            mCookie = cookie;
        }

        public void binderDied() {
            synchronized (mCallbacks) {
                mCallbacks.remove(mCallback.asBinder());
            }
            onCallbackDied(mCallback, mCookie);
        }
    }
    /******其他代码省略*******/
}
```

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>