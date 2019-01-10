---
title: Android之Service学习笔记
date: 2016-12-28 19:14:00
tags: [Service]
categories: Android
description: "本来想学习学习Binder通信机制，在学习的过程中又接触AIDL并开始学习，在AIDL学习过程中看到bindService，接着就想回顾一下Service的一些知识。希望温故可以知新，也算是年末最后一篇笔记，给自己明年有一个好的开头~！~！"
---
前言
===
本来想学习学习Binder通信机制，在学习的过程中又接触AIDL并开始学习，在AIDL学习过程中看到bindService，接着就想回顾一下Service的一些知识。希望温故可以知新，也算是年末最后一篇笔记，给自己明年有一个好的开头~！~！

介绍
====
[Service](https://developer.android.com/reference/android/app/Service.html) 是一个可以在后台执行长时间运行操作而不提供用户界面的应用组件。服务可由其他应用组件启动，而且即使用户切换到其他应用，服务仍将在后台继续运行。 此外，组件可以绑定到服务，以与之进行交互，甚至是执行进程间通信 (IPC)。 例如，服务可以处理网络事务、播放音乐，执行文件 I/O 或与内容提供程序交互，而所有这一切均可在后台进行。

创建Service
====
Service的创建可以分为两种分别是：启动和绑定。

启动：
> 当应用组件（如 Activity）通过调用 [startService()](https://developer.android.com/reference/android/content/Context.html#startService(android.content.Intent)) 启动服务时，服务即处于“启动”状态。一旦启动，服务即可在后台无限期运行，即使启动服务的组件已被销毁也不受影响。 已启动的服务通常是执行单一操作，而且不会将结果返回给调用方。例如，它可能通过网络下载或上传文件。 操作完成后，服务会自行停止运行。

绑定：
> 当应用组件通过调用 [bindService()](https://developer.android.com/reference/android/content/Context.html#bindService(android.content.Intent, android.content.ServiceConnection, int)) 绑定到服务时，服务即处于“绑定”状态。绑定服务提供了一个客户端-服务器接口，允许组件与服务进行交互、发送请求、获取结果，甚至是利用进程间通信 (IPC) 跨进程执行这些操作。 仅当与另一个应用组件绑定时，绑定服务才会运行。 多个组件可以同时绑定到该服务，但全部取消绑定后，该服务即会被销毁。

同名的Service只能存在一个，但运行方式可以两种并存。也就是说，它既可以是启动服务（以无限期运行），也允许绑定。问题只是在于您是否实现了一组回调方法：[onStartCommand()](https://developer.android.com/reference/android/app/Service.html#onStartCommand(android.content.Intent, int, int)) （允许组件启动服务）和 [onBind()](https://developer.android.com/reference/android/app/Service.html#onBind(android.content.Intent)) （允许绑定服务）。

> 需要注意的是Service它是运行在主线程中的，如果服务执行的CPU密集型或阻塞性质的操作，那么应该在服务内创建新的线程去工作。通过使用单独的线程，可以降低发生“应用无响应”(ANR) 错误的风险，而应用的主线程仍可继续专注于运行用户与 Activity 之间的交互。


Service之基础
====
下边主要介绍Service的一些方法以及其生命周期。
* onCreate()

> 首次创建服务时，系统将调用此方法来执行一次性设置程序（在调用 [onStartCommand()](https://developer.android.com/reference/android/app/Service.html#onStartCommand(android.content.Intent, int, int)) 或[onBind()](https://developer.android.com/reference/android/app/Service.html#onBind(android.content.Intent)) 之前）。如果服务已在运行，则不会调用此方法。

* onStartCommand()

> 当另一个组件（如 Activity）通过调用 [startService()](https://developer.android.com/reference/android/content/Context.html#startService(android.content.Intent)) 请求启动服务时，系统将调用此方法。一旦执行此方法，服务即会启动并可在后台无限期运行。
关于onStartCommand返回值可以查看[Service之onStartCommand剖析笔记](http://dandanlove.com/2016/12/28/onStartCommand/)

* onBind()

> 当另一个组件想通过调用 [bindService()](https://developer.android.com/reference/android/content/Context.html#bindService(android.content.Intent, android.content.ServiceConnection, int)) 与服务绑定（例如执行 RPC）时，系统将调用此方法。在此方法的实现中，您必须通过返回 [IBinder](https://developer.android.com/reference/android/os/IBinder.html)
 提供一个接口，供客户端用来与服务进行通信。请务必实现此方法，但如果您并不希望允许绑定，则应返回 null。

* onRebind()

> 如果之前有断开连接的时候调用onUnbind方法，并且其返回值为ture。那么在新的客户端需要进行和服务进行连接的时候会调用该方法。

* onUnbind()

> 和onRebind()对应，如果想在新的客户端连接的时候收到通知那么onUnbind()的返回值设置为ture，但改方法的默认返回值为false。

* onDestroy()
> 当服务不再使用且将被销毁时，系统将调用此方法。服务应该实现此方法来清理所有资源，如线程、注册的侦听器、接收器等。 这是服务接收的最后一个调用。

Service的声明周期
===
![Service生命周期](https://developer.android.com/images/service_lifecycle.png)

该图分别表示了startService和bindService的声明周期，那么当Service即有startService又有bindService时呢？

![Service的生命周期.png](http://upload-images.jianshu.io/upload_images/1319879-75e2dd7f28a2ded2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该图是作者使用同一个Service和多个不同的client去bindService和startService得出的结论。

> * Service类似于单例，无论启动多少次onCreate只执行一次，除非执行了onDestory或者stopSelf；
* 当client进行startService的时候，Service会调用onStartCommand方法。所以我们一般会在这个方法法中处理传递事件；
* 当client在进行stopService的时候，如果此时没有任何其他的client与其绑定（startService | bindService）那么Service才会执行onDestory方法；
* 当client在进行bindService的时候，如果Service没有被bind过那么Server会调用它的onBind方法。因为当Service的onBind方法被调用过后Ibinder已经被AMS获取到，那么在client进行bindService的时候会先判断是否Service的onUnbind方法已经被调用过，如果没有那么直接返回该Ibinder，否则根据onUnbind的返回值判断是否调用onRebind方法；
* 当client进行onbindService的时候，如果此时没有任何client在bind状态，那么就会调用Service的onUnbind，然后再判断是否有其他的client与Service绑定（startService），如果没有的话，那么Service就会调用onDestroy方法；

下边是源码的方法执行流程：
===

![startService.png](http://upload-images.jianshu.io/upload_images/1319879-7184a2c6402fe367.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![bindService.png](http://upload-images.jianshu.io/upload_images/1319879-c7cb6e59eee2de3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![unbindService.png](http://upload-images.jianshu.io/upload_images/1319879-57eda9c3c8bf69ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![stopService&stopSelf.png](http://upload-images.jianshu.io/upload_images/1319879-a4e636e86d0134cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面流程图只是方法的调用，关于具体方法的实现还没有进行仔细的研究所以这里就不贴了。
看到这里关于Service的基础知识应该回顾的差不多，我们一起期待新的一年的开始吧~！~！~！

想阅读作者的更多文章，可以查看我的公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>