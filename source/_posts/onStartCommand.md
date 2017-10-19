---
title: Service之onStartCommand剖析笔记
date: 2016-12-28 9:14:00
tags: [Service]
categories: Android
description: "Service是我们学习Android的基石之一，它在移动应用程序中使用非常广泛。比如应用定位，push消息，内存流量监听等等。记得大四那年在公司实习的时候，我做的第一个调研就是怎么让接受服务器push的Service不被kill掉（或kill后实现重新启动）。"
---
Service是我们学习Android的基石之一，它在移动应用程序中使用非常广泛。比如应用定位，push消息，内存流量监听等等。
记得大四那年在公司实习的时候，我做的第一个调研就是怎么让接受服务器push的Service不被kill掉（或kill后实现重新启动）。在调研的过程中就了解到如果Service的onStartCommand方法返回值为START_STICKY时，那么Service在不久后就会尝试重启。。。。现在自己重温Service知识点，想将其记录一下让自己对其印象更加深刻。

我们常用的onStartCommand返回值有START_STICKY、START_NOT_STICKY、START_REDELIVER_INTENT等等。
START_STICKY：
=======
> Constant to return from onStartCommand(Intent, int, int): if this service's process is killed while it is started (after returning from onStartCommand(Intent, int, int)), then leave it in the started state but don't retain this delivered intent. Later the system will try to re-create the service. Because it is in the started state, it will guarantee to call onStartCommand(Intent, int, int) after creating the new service instance; if there are not any pending start commands to be delivered to the service, it will be called with a null intent object, so you must take care to check for this.
This mode makes sense for things that will be explicitly started and stopped to run for arbitrary periods of time, such as a service performing background music playback.

> * 如果在onStartCommand(Intent, int, int)返回恒为START_STICKY。那么假设这个服务所在的进程被杀掉，那么开始在启动onStartCommand(Intent, int, int)方方法是传过来的Intent不会被保留。稍后系统会尝试重新创建这个service，并保证开始在创建新的service实例后调用onStartCommand(Intent, int, int)方法。如果这里没有任何挂起的服务调用service，那么系统会通过空Intent调用onStartCommand(Intent, int, int)方法。此模式对于明确启动和停止运行任意时间段（例如执行背景音乐播放的服务）的事物有意义。
* getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.ECLAIR时默认为START_STICKY


START_STICKY_COMPATIBILITY：
====
> Constant to return from onStartCommand(Intent, int, int): compatibility version of START_STICKY that does not guarantee that onStartCommand(Intent, int, int) will be called again after being killed.
* 对START_STICKY的兼容，不保证service杀掉后调用onStartCommand(Intent, int, int)。
* getApplicationInfo().targetSdkVersion < Build.VERSION_CODES.ECLAIR时默认为START_STICKY_COMPATIBILITY


START_NOT_STICKY：
====
> Constant to return from onStartCommand(Intent, int, int): if this service's process is killed while it is started (after returning from onStartCommand(Intent, int, int)), and there are no new start intents to deliver to it, then take the service out of the started state and don't recreate until a future explicit call to Context.startService(Intent). The service will not receive a onStartCommand(Intent, int, int) call with a null Intent because it will not be re-started if there are no pending Intents to deliver.

> 如果在onStartCommand(Intent, int, int)返回恒为START_STICKY。那么假设这个服务所在的进程被杀掉，而且没有一个新的intent启动它。那么服务的状态将不会被保留，直到一个新的显示调用 Context.startService(Intent)。如果没有一个挂起的Intent要传递，否则系统不会重建服务。

START_REDELIVER_INTENT：
====
> Constant to return from onStartCommand(Intent, int, int): if this service's process is killed while it is started (after returning from onStartCommand(Intent, int, int)), then it will be scheduled for a restart and the last delivered Intent re-delivered to it again via onStartCommand(Intent, int, int). This Intent will remain scheduled for redelivery until the service calls stopSelf(int) with the start ID provided to onStartCommand(Intent, int, int). The service will not receive a onStartCommand(Intent, int, int) call with a null Intent because it will will only be re-started if it is not finished processing all Intents sent to it (and any such pending events will be delivered at the point of restart).

> 如果在onStartCommand(Intent, int, int)返回恒为START_STICKY。那么假设这个服务所在的进程被杀掉,那么这个服务将会重启并且将最后一个传递的Intent再次传递给onStartCommand(Intent, int, int)。这个Intent将会一直保留直到当前service调用stopSelf(int)

总结：
====
* 如果希望Service一直存活并且保留上次启动它的intent的数据，那么return START_REDELIVER_INTENT；
* 如果只希望Service一直存活不需要intent中的数据，那么return START_STICKY；
* 如果希望Service执行完指定的任务后销毁，那么return START_NOT_STICKY；
* 如果没有什么要求那么直接return super.onStartCommand；

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>