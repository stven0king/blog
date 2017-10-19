---
title: 又一年对Android消息机制（Handler&Looper）的思考
date: 2017-6-25 22:39:00
tags: [Android消息机制,Handler,Looper]
categories: Android 
description: "Android消息机制对于每一个Android开发者来说都不陌生，在日常的开发中我们不可避免的要经常涉及这部分的内容。从开发角度来说，Handler是Android消息机制的上层接口，这使得在开发过程中只需要和Handler交互即可。Handler的使用过程很简单，通过它可以轻松的将一个任务切换Handler所在的线程中去执行。很多人认为Handler的作用是更新UI，这的确没错，但是更新UI仅仅是Handler的一个特殊的使用场景。具体来说是这样的；有时候需要再子线程中进行耗时的I/O操作，可能是读取文件或访问网络等。。。。。"
---
前言
----
Android消息机制对于每一个Android开发者来说都不陌生，在日常的开发中我们不可避免的要经常涉及这部分的内容。从开发角度来说，Handler是Android消息机制的上层接口，这使得在开发过程中只需要和Handler交互即可。Handler的使用过程很简单，通过它可以轻松的将一个任务切换Handler所在的线程中去执行。很多人认为Handler的作用是更新UI，这的确没错，但是更新UI仅仅是Handler的一个特殊的使用场景。具体来说是这样的；有时候需要再子线程中进行耗时的I/O操作，可能是读取文件或访问网络等。。。。。

本文是是作者在【温故而知新】时，发现自己之前做的笔记再次完善后的，希望对各位读者都有所帮助~！

<center>![handler.jpg](http://oltcsi62w.bkt.clouddn.com/image/android/handler-looper.jpg)</center>

思考上边这幅图，让我们带着自己的期待，开始从Handler的使用一步步探究到源代码的愉快之旅。

Handler
----
Handler的对象的创建
====
```java
private Handler mHandler = new Handler(){
	@Override
	public void handleMessage(Message msg) {
		super.handleMessage(msg);
	}
};
```

利用Handler发送消息
====
```java
Message message = Message.obtain();
message.setData(bundle);
message.setTarget(mHandler);
message.sendToTarget();
//或者mHandler.handleMessage(message);
```
Handler的使用基本上就是上边所展示的【创建实例、处理消息、发送消息】这些内容。为什么这么简单的使用就能实现线程间的通信呢。。。

【源码分析】Handler创建
====
上面我们介绍了Handler的使用，并且带着疑问【Handler使用的每一行代码我们都知道他的具体作用以及异步线程中是怎么传递消息的么】？

我们先从Handler的构造函数开始
```java
/**
 * Default constructor associates this handler with the {@link Looper} for the
 * current thread.
 *
 * If this thread does not have a looper, this handler won't be able to receive messages
 * so an exception is thrown.
 */
public Handler() {
	this(null, false);
}

public Handler(Callback callback) {
	this(callback, false);
}

public Handler(Looper looper) {
	this(looper, null, false);
}

public Handler(Looper looper, Callback callback) {
	this(looper, callback, false);
}

public Handler(boolean async) {
	this(null, async);
}

public Handler(Callback callback, boolean async) {
	if (FIND_POTENTIAL_LEAKS) {
		final Class<? extends Handler> klass = getClass();
		if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
				(klass.getModifiers() & Modifier.STATIC) == 0) {
			Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
				klass.getCanonicalName());
		}
	}

	mLooper = Looper.myLooper();
	if (mLooper == null) {
		throw new RuntimeException(
			"Can't create handler inside thread that has not called Looper.prepare()");
	}
	mQueue = mLooper.mQueue;
	mCallback = callback;
	mAsynchronous = async;
}

public Handler(Looper looper, Callback callback, boolean async) {
	mLooper = looper;
	mQueue = looper.mQueue;
	mCallback = callback;
	mAsynchronous = async;
}
```
我们可以通过构造函数可以看出来，Handler的构造函数实际的作用就是实例化内部的Looper类型的成员变量mLooper，和Looper内部的MessageQueue类型的变量mQueue。

【源码分析】Handler发送消息
====
发送消息有两种一种是利用Message，一种是调用Handler的sendMessage方法。
```java
/**
 * Sends this Message to the Handler specified by {@link #getTarget}.
 * Throws a null pointer exception if this field has not been set.
 */
public void sendToTarget() {
	target.sendMessage(this);
}
```
但是很显然第一种方法内部还是调用Handler#sendMessage方法，接下来我们看看该方法的具体实现。
```java
/**
 * Pushes a message onto the end of the message queue after all pending messages
 * before the current time. It will be received in {@link #handleMessage},
 * in the thread attached to this handler.
 *  
 * @return Returns true if the message was successfully placed in to the 
 *         message queue.  Returns false on failure, usually because the
 *         looper processing the message queue is exiting.
 */
public final boolean sendMessage(Message msg){
	return sendMessageDelayed(msg, 0);
}
/**
 * Enqueue a message into the message queue after all pending messages
 * before (current time + delayMillis). You will receive it in
 * {@link #handleMessage}, in the thread attached to this handler.
 *  
 * @return Returns true if the message was successfully placed in to the 
 *         message queue.  Returns false on failure, usually because the
 *         looper processing the message queue is exiting.  Note that a
 *         result of true does not mean the message will be processed -- if
 *         the looper is quit before the delivery time of the message
 *         occurs then the message will be dropped.
 */
public final boolean sendMessageDelayed(Message msg, long delayMillis){
	if (delayMillis < 0) {
		delayMillis = 0;
	}
	return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
/**
 * Enqueue a message into the message queue after all pending messages
 * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
 * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
 * Time spent in deep sleep will add an additional delay to execution.
 * You will receive it in {@link #handleMessage}, in the thread attached
 * to this handler.
 * 
 * @param uptimeMillis The absolute time at which the message should be
 *         delivered, using the
 *         {@link android.os.SystemClock#uptimeMillis} time-base.
 *         
 * @return Returns true if the message was successfully placed in to the 
 *         message queue.  Returns false on failure, usually because the
 *         looper processing the message queue is exiting.  Note that a
 *         result of true does not mean the message will be processed -- if
 *         the looper is quit before the delivery time of the message
 *         occurs then the message will be dropped.
 */
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	MessageQueue queue = mQueue;
	if (queue == null) {
		RuntimeException e = new RuntimeException(
				this + " sendMessageAtTime() called with no mQueue");
		Log.w("Looper", e.getMessage(), e);
		return false;
	}
	return enqueueMessage(queue, msg, uptimeMillis);
}
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    //将要发送的Message绑上该Handler的标签
	msg.target = this;
	if (mAsynchronous) {
		msg.setAsynchronous(true);
	}
	//将消息将入Handler构造函数中初始化的MessageQueue队列
	return queue.enqueueMessage(msg, uptimeMillis);
}
```
调用用Handler#sendMessage方法实际上是一步一步最终将消息与该Handler绑定，并放入Handler构造函数中初始化的MessageQueue队列中。消息队列有进就有出，MessageQueue不断接受Handler发送过来的消息，那么MessageQueue是怎么处理这些消息的呢？因为Handler是内的消息队列是引用的Looper中的成员变量，而消息的循环接受处理正式Looper的loop方法，接下来我们详细讲解并分析Looper类在Android的消息机制中的所扮演的角色！

Looper
----
Lopper简介
====
Looper在Android的消息机制中扮演着消息循环的角色，具体来说就是它会不停地从MessageQueue中查看是否有新消息，如果有新消息就会立即处理，否则就一直阻塞在哪里。
Looper的构造函数
====
```java
private Looper(boolean quitAllowed) {
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}
```
通过Looper的构造函数我们可以看出，Looper每个实例中都会有一个MessageQueue队列，用来循环读取消息。


Looper的主要成员变量
====
```java
// sThreadLocal.get() will return null unless you've called prepare().
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static Looper sMainLooper;  // guarded by Looper.class

final MessageQueue mQueue;
final Thread mThread;
```
> * sThreadLocal是一个ThreadLocal类型的变量，使用时可以得到当前线程的Looper实例。
* sMainLooper表示主线程的Looper实例。
* mQueue是Looper实例中的消息队列。
* mThread线程类型的对象。

通过Looper的成员变量我们可以看出来，每个线程只能有一个Loope实例。Looper是通过mQueue不断循环接受消息、分发消息的。

Looper的获取
====
为什么这里没有写Looper的创建而是Looper的获取呢？Looper的构造方法能为我们创建可用Looper实例么？仔细看上面的Looper的构造函数我们可以发现，构造函数中没有将Looper实例放入sThreadLocal中。我们看看下边prepare()和myLooper()方法的实现。

```java
/** Initialize the current thread as a looper.
  * This gives you a chance to create handlers that then reference
  * this looper, before actually starting the loop. Be sure to call
  * {@link #loop()} after calling this method, and end it by calling
  * {@link #quit()}.
  */
public static void prepare() {
	prepare(true);
}

private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread");
	}
     //Looper类会创建一个新的Looper对象,并放入全局的sThreadLocal中。
	sThreadLocal.set(new Looper(quitAllowed));
}

/**
 * Return the Looper object associated with the current thread.  Returns
 * null if the calling thread is not associated with a Looper.
 */
public static Looper myLooper() {
	return sThreadLocal.get();
}

/**
 * Initialize the current thread as a looper, marking it as an
 * application's main looper. The main looper for your application
 * is created by the Android environment, so you should never need
 * to call this function yourself.  See also: {@link #prepare()}
 */
public static void prepareMainLooper() {
	prepare(false);
	synchronized (Looper.class) {
		if (sMainLooper != null) {
			throw new IllegalStateException("The main Looper has already been prepared.");
		}
		sMainLooper = myLooper();
	}
}
/**
 * Returns the application's main looper, which lives in the main thread of the application.
 */
public static Looper getMainLooper() {
	synchronized (Looper.class) {
		return sMainLooper;
	}
}
```

在启动Activity的时候sMainLooper已经开始工作了（具体请看下边ActivityThread的main方法），所以我们要获取主线程的Looper实例，只需要调用Looper.getMainLooper。但在其他异步线程中我们要创建Looper通常我们通过以下方式获取Looper对象。

```java
Looper.prepare();
Looper looper = Looper.myLooper();
```

接下来我们看看在Framwork层中Looper是怎么使用的：ActivityThread#main
```java
public static void main(String[] args) {
	Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
	SamplingProfilerIntegration.start();

	// CloseGuard defaults to true and can be quite spammy.  We
	// disable it here, but selectively enable it later (via
	// StrictMode) on debug builds, but using DropBox, not logs.
	CloseGuard.setEnabled(false);

	Environment.initForCurrentUser();

	// Set the reporter for event logging in libcore
	EventLogger.setReporter(new EventLoggingReporter());

	// Make sure TrustedCertificateStore looks in the right place for CA certificates
	final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
	TrustedCertificateStore.setDefaultUserDirectory(configDir);

	Process.setArgV0("<pre-initialized>");
    //创建主线程的Looper对象
	Looper.prepareMainLooper();

	ActivityThread thread = new ActivityThread();
	thread.attach(false);

	if (sMainThreadHandler == null) {
		sMainThreadHandler = thread.getHandler();
	}

	if (false) {
		Looper.myLooper().setMessageLogging(new
				LogPrinter(Log.DEBUG, "ActivityThread"));
	}

	// End of event ActivityThreadMain.
	Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
	//开启循坏队列监听并接受&处理消息
	Looper.loop();

	throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
我们可以看见main方法执行的时候主线程中的Looper就已经创建并循环等待处理消息了。

Looper的loop
====
上面我们讲述了Looper对象实例的获取，那么获取完后我们就需要开启循坏队列监听并接受&处理消息，即执行Looper.loop()方法。
```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
	//获取当前线程的Looper对象实例
	final Looper me = myLooper();
	//调用loop()方法之前必须在当前线程通过Looper.prepare()方法创建Looper对象实例。
	if (me == null) {
		throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
	}
	//获取Looper实例的中的消息队列
	final MessageQueue queue = me.mQueue;

	// Make sure the identity of this thread is that of the local process,
	// and keep track of what that identity token actually is.
	Binder.clearCallingIdentity();
	final long ident = Binder.clearCallingIdentity();
	//开启循环队列-不断
	for (;;) {
		//从队列中取出消息
		Message msg = queue.next(); // might block
		if (msg == null) {
			// No message indicates that the message queue is quitting.
			return;
		}

		// This must be in a local variable, in case a UI event sets the logger
		final Printer logging = me.mLogging;
		if (logging != null) {
			logging.println(">>>>> Dispatching to " + msg.target + " " +
					msg.callback + ": " + msg.what);
		}

		final long traceTag = me.mTraceTag;
		if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
			Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
		}
		//将消息分发给注册它的handler
		try {
			msg.target.dispatchMessage(msg);
		} finally {
			if (traceTag != 0) {
				Trace.traceEnd(traceTag);
			}
		}

		if (logging != null) {
			logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
		}

		// Make sure that during the course of dispatching the
		// identity of the thread wasn't corrupted.
		final long newIdent = Binder.clearCallingIdentity();
		if (ident != newIdent) {
			Log.wtf(TAG, "Thread identity changed from 0x"
					+ Long.toHexString(ident) + " to 0x"
					+ Long.toHexString(newIdent) + " while dispatching to "
					+ msg.target.getClass().getName() + " "
					+ msg.callback + " what=" + msg.what);
		}

		msg.recycleUnchecked();
	}
}
```

到这里我们终于找到消息发送后的处理逻辑，实际上是通过每条消息传递给该消息所绑定的Handler的dispatchMessgage方法进行处理。

【源码分析】Handler的消息处理
====
```java
/**
 * Handle system messages here.
 */
public void dispatchMessage(Message msg) {
	if (msg.callback != null) {
		handleCallback(msg);
	} else {
		if (mCallback != null) {
			if (mCallback.handleMessage(msg)) {
				return;
			}
		}
		handleMessage(msg);
	}
}
```
这个就是为什么我们在创建Handler对象的时候都需要复写它的handleMessage方法。

总结
----
看完整篇文章我们就能理解Android中的消息机制了吧？可能有些同学还是有些小疑惑，我貌似看到了并理解了Handler对消息的处理【Handler发送消息并添加到队列中，Looper循环将队列里的消息发给Handler处理】，但是好像对Handler是怎么实现多线程异步通信还有些不清楚？

多线程异步通信，Handler它到底是怎么实现的？
====

首先我们需要明确三点：
> * 线程之间内存是共享的；
* 每个线程最多只能有一个Looper对象实例；
* Handler创建的时候必须绑定Looper对象（默认为当前线程的Looper实例对象）；

确定以上三点，我们就容易理解多了。多个线程编程中，当多个线程只拥有一个Handler实例时，无论消息在哪个线程里发送出来，消息都会在这个Handler的handleMessage方法中做处理（举例：若在主线程中创建一个Handler对象并且该Handler对象绑定的是主线程的Looper对象实例，那么我们在异步线程中使用该handler发送的消息，最终都会将在主线程中进行处理。eg：网络请求，文件读写。。）。

现在我们终于可以清空我们所有的疑问愉快的结束本文章的阅读啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>
![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>