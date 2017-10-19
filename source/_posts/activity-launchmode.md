---
title: 深入理解Activity启动模式之launchMode
date: 2017-7-20 10:29:00
tags: [activity,launchMode]
categories: Android 
description: "Android每个Application都是由若干个四大组件组成的。每个页面都是一个Activity，当需要打开相应页面（Activity）时系统会创建他们的实例并把他们一一放入栈中进行管理。任务栈是一种“后进先出”的栈结构，通过back键，我们可以发现这些Activity会一一出栈（PS：不断返回上一页）。如果每次启动Activity都创建一个实例，会不会很浪费资源。能不能进行Activity的复用呢？。。"
---
Android每个Application都是由若干个四大组件组成的。每个页面都是一个Activity，当需要打开相应页面（Activity）时系统会创建他们的实例并把他们一一放入栈中进行管理。任务栈是一种“后进先出”的栈结构，通过back键，我们可以发现这些Activity会一一出栈（PS：不断返回上一页）。如果每次启动Activity都创建一个实例，会不会很浪费资源。能不能进行Activity的复用呢？Android系统在设计就考虑到这个问题，所以提供了同步的Activity启动模式，在不同条件下进行Activity的复用。其中都包括：Standard、SingleTop、SingleTask、SingleInstance。
（PS：本篇文章的实验数据都基于Android7.0）

在讲述Activity启动模式之前我们先了解一些基础概念：
> Application 是组件的集合。
Process 操作系统调度的单位。
Task是指在执行特定作业时与用户交互的一系列 Activity。Task以栈的形式管理Activity集合。
- 一个应用中可以有多个Task
- 不同应用的Activity可以放在同一个Task中进行管理
- 一个应用中可以有多个Process
- Task可以运行在不同进程

standard
===
> 标准启动模式，也是activity的默认启动模式。在这种模式下启动的activity可以被多次实例化，即在同一个任务中可以存在多个activity的实例，每个实例都会处理一个Intent对象。

<center>![standard.png](http://upload-images.jianshu.io/upload_images/1319879-23d7d680ef04ee97.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

- AndroidManifest.xml

```java
<application
	android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:supportsRtl="true"
    android:theme="@style/AppTheme" >
    <activity android:name=".AActivity"
        android:launchMode="standard">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
    <activity android:name=".BActivity"
              android:launchMode="standard"/>
</application>
```

- Logcat输出

```java
3459-3459 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6384
3459-3459 D/$$$AActivity: onClickBtn: AActivity start BActivity
3459-3459 D/$$$BActivity: onCreate: currentActivityName:BActivity currentTaskID:6384
3459-3459 D/$$$BActivity: onClickBtn: BActivity start AActivity
3459-3459 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6384
3459-3459 D/$$$AActivity: onClickBtn: AActivity start BActivity
3459-3459 D/$$$BActivity: onCreate: currentActivityName:BActivity currentTaskID:6384
```

- 当前任务栈

```java
 Running activities (most recent first):
      TaskRecord{3efeab #6384 A=com.tzx.launchmodel U=0 StackId=1 sz=4}
        Run #4: ActivityRecord{18878b0 u0 com.tzx.launchmodel/.BActivity t6384}
        Run #3: ActivityRecord{5a7f957 u0 com.tzx.launchmodel/.AActivity t6384}
        Run #2: ActivityRecord{559bd5f u0 com.tzx.launchmodel/.BActivity t6384}
        Run #1: ActivityRecord{facb908 u0 com.tzx.launchmodel/.AActivity t6384}
```

singleTop
==
> 启动的activity的实例已经存在于任务桟的桟顶，那么再启动这个Activity时，不会创建新的实例，而是重用位于栈顶的那个实例，并且会调用该实例的onNewIntent()方法将Intent对象传递到这个实例中。

<center>![singleTop.png](http://upload-images.jianshu.io/upload_images/1319879-c7638e6221ef6e98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

- AndroidManifest.xml

```java
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:supportsRtl="true"
    android:theme="@style/AppTheme" >
    <activity android:name=".AActivity"
        android:launchMode="standard">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
    <activity android:name=".BActivity"
              android:launchMode="singleTop"/>
</application>
```

- Logcat输出

```java
14450-14450 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6386
14450-14450 D/$$$AActivity: onClickBtn: AActivity start BActivity
14450-14450 D/$$$BActivity: onCreate: currentActivityName:BActivity currentTaskID:6386
14450-14450 D/$$$BActivity: onClickBtn: BActivity start AActivity
14450-14450 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6386
14450-14450 D/$$$AActivity: onClickBtn: AActivity start BActivity
14450-14450 D/$$$BActivity: onCreate: currentActivityName:BActivity currentTaskID:6386
14450-14450 D/$$$BActivity: onClickBtn: BActivity start BActivity
14450-14450 D/$$$BActivity: onNewIntent()
14450-14450 D/$$$BActivity: onClickBtn: BActivity start BActivity
14450-14450 D/$$$BActivity: onNewIntent()
```

- 当前任务栈

```java
Running activities (most recent first):
      TaskRecord{fba1a37 #6386 A=com.tzx.launchmodel U=0 StackId=1 sz=4}
        Run #5: ActivityRecord{9609e73 u0 com.tzx.launchmodel/.BActivity t6386}
        Run #4: ActivityRecord{50fff56 u0 com.tzx.launchmodel/.AActivity t6386}
        Run #3: ActivityRecord{1c403e9 u0 com.tzx.launchmodel/.BActivity t6386}
        Run #2: ActivityRecord{3c41ba4 u0 com.tzx.launchmodel/.AActivity t6386}
```

singleTask
===
> 启动模式为singleTask，那么系统总会在一个新任务的最底部（root）启动这个activity，并且被这个activity启动的其他activity会和该activity同时存在于这个新任务中。如果系统中已经存在这样的一个activity则会重用这个实例，并且调用他的onNewIntent()方法。即，这样的一个activity在系统中只会存在一个实例。

<center>![singleTask.png](http://upload-images.jianshu.io/upload_images/1319879-e3855b9fe746f456.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

- AndroidManifest.xml

```java
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:supportsRtl="true"
    android:theme="@style/AppTheme" >
    <activity android:name=".AActivity"
        android:launchMode="standard">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
    <activity android:name=".BActivity"
              android:launchMode="singleTask"/>
    <activity android:name=".CActivity"
        android:process=":remote"/>
</application>
```

- Logcat输出

```java
19684-19684 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6394
19684-19684 D/$$$AActivity: onClickBtn: AActivity start BActivity
19684-19684 D/$$$BActivity: onCreate: currentActivityName:BActivity currentTaskID:6394
19684-19684 D/$$$BActivity: onClickBtn: BActivity start AActivity
19684-19684 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6394
19684-19684 D/$$$AActivity: onClickBtn: AActivity start CActivity
19921-19921 D/$$$CActivity: onCreate: currentActivityName:CActivity currentTaskID:6394
19921-19921 D/$$$CActivity: onClickBtn: CActivity start AActivity
19684-19684 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6394
19684-19684 D/$$$AActivity: onClickBtn: AActivity start BActivity
19684-19684 D/$$$BActivity: onNewIntent()
19684-19684 D/$$$BActivity: onClickBtn: BActivity start BActivity
19684-19684 D/$$$BActivity: onNewIntent()
```

- 当前任务栈

```java
Running activities (most recent first):
      TaskRecord{8e8fb09 #6394 A=com.tzx.launchmodel U=0 StackId=1 sz=2}
        Run #7: ActivityRecord{7489fc u0 com.tzx.launchmodel/.BActivity t6394}
        Run #6: ActivityRecord{9a5d10e u0 com.tzx.launchmodel/.AActivity t6394}
```

singleInstance
===
> 总是在新的任务中开启，并且这个新的任务中有且只有这一个实例。

<center>![singleInstance.png](http://upload-images.jianshu.io/upload_images/1319879-4adba415bb2c3fdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

- AndroidManifest.xml

```java
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:supportsRtl="true"
    android:theme="@style/AppTheme" >
    <activity android:name=".AActivity"
        android:launchMode="standard">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
    <activity android:name=".BActivity"
              android:launchMode="singleInstance"/>
    <activity android:name=".CActivity"
        android:process=":remote"/>
</application>
```

- Logcat输出

```java
25016-25016 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6396
25016-25016 D/$$$AActivity: onClickBtn: AActivity start BActivity
25016-25016 D/$$$BActivity: onCreate: currentActivityName:BActivity currentTaskID:6397
25016-25016 D/$$$BActivity: onClickBtn: BActivity start AActivity
25016-25016 D/$$$AActivity: onCreate: currentActivityName:AActivity currentTaskID:6396
25016-25016 D/$$$AActivity: onClickBtn: AActivity start CActivity
26003-26003 D/$$$CActivity: onCreate: currentActivityName:CActivity currentTaskID:6396
26003-26003 D/$$$CActivity: onClickBtn: CActivity start BActivity
25016-25016 D/$$$BActivity: onNewIntent()
```

- 当前任务栈

```java
 Running activities (most recent first):
      TaskRecord{d5934c9 #6397 A=com.tzx.launchmodel U=0 StackId=1 sz=1}
        Run #9: ActivityRecord{986e7ce u0 com.tzx.launchmodel/.BActivity t6397}
      TaskRecord{f282078 #6396 A=com.tzx.launchmodel U=0 StackId=1 sz=3}
        Run #8: ActivityRecord{d2c75de u0 com.tzx.launchmodel/.CActivity t6396}
        Run #7: ActivityRecord{2d3a2eb u0 com.tzx.launchmodel/.AActivity t6396}
        Run #6: ActivityRecord{2ca9351 u0 com.tzx.launchmodel/.AActivity t6396}
```

本文叙述性的文字非常少，主要通过其代码和其运行结果然后得出我们需要的结论。每一种设计模式我都画了一副图，可以使大家理解记忆深刻。
Activity启动模式到这里就结束了么，然而并没有。之前看过其他类似文章或者书籍的同学都知道还有Activity的Flags和 ```android:taskAffinity``` 标签对Activity的启动有影响。这些内容我们放在下一张[深入理解Activity启动模式之大结局](http://www.jianshu.com/p/8bc28d794e11)中讲述，有兴趣的同学可以点击阅读～！

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>