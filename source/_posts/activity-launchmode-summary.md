---
title: 深入理解Activity启动模式之大结局
date: 2017-7-21 10:30:00
tags: [activity,launchMode]
categories: Android 
description: "谈起Activity的启动模式必不可少的要是launchMode、Flags、taskAffinity这三块知识点，上一篇文章 [深入理解Activity启动模式之launchMode]看过的同学都知道该文章对launchMode做了非常详细的讲解，所以本片文章承接上一篇文章对剩余的Flags、taskAffinity这两块做讲述，希望看完此片文章的同学们此后遇到Activity的启动模式相关问题或使用场景再也不用查资料^_^。"
---
<center>![lastday.jpg](http://upload-images.jianshu.io/upload_images/1319879-e481b89f30352afc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

谈起Activity的启动模式必不可少的要是launchMode、Flags、taskAffinity这三块知识点，上一篇文章 [深入理解Activity启动模式之launchMode](http://www.jianshu.com/p/51a28a380c6a) 看过的同学都知道该文章对launchMode做了非常详细的讲解，所以本片文章承接上一篇文章对剩余的Flags、taskAffinity这两块做讲述，希望看完此片文章的同学们此后遇到Activity的启动模式相关问题或使用场景再也不用查资料^_^。
（PS：本篇文章的实验数据都基于Android7.0）

Activity启动模式之Flags
---
先来看看常用Flags：
> - Intent.FLAG_ACTIVITY_SINGLE_TOP 
  该标志位表示使用singleTop模式来启动一个Activity，与在清单文件指定android:launchMode="singleTop"效果相同。
- Intent.FLAG_ACTIVITY_CLEAR_TOP 
  该标志位表示使用singleTask模式来启动一个Activity，与在清单文件指定android：launchMode="singleTask"效果相同。
- Intent.FLAG_ACTIVITY_NO_HISTORY 
  使用该模式来启动Activity，当该Activity启动其他Activity后，该Activity就被销毁了，不会保留在任务栈中。如A-B,B中以这种模式启动C，C再启动D，则任务栈只有ABD。
- Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS 
  使用该标识位启动的Activity不添加到最近应用列表，也即我们从最近应用里面查看不到我们启动的这个activity。与属性android:excludeFromRecents="true"效果相同。
- Intent.FLAG_ACTIVITY_NEW_TASK
该标志位表示使用一个新的Task来启动一个Activity，相当于在清单文件中给Activity指定“singleTask”启动模式。通常我们在Service启动Activity时，由于Service中并没有Activity任务栈，所以必须使用该Flag来创建一个新的Task。

使用也非常简单：

```java
public static void startActivity(Context context) {
    Intent intent = new Intent(context, HomeActivity.class);
    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    context.startActivity(new Intent(context, HomeActivity.class));
}
```

android:taskAffinity
---
我们重点来看看taskAffinity这个标签
> [android:taskAffinity 官网解释](https://developer.android.com/guide/topics/manifest/activity-element.html#reparent)
与 Activity 有着亲和关系的任务。从概念上讲，具有相同亲和关系的 Activity 归属同一任务（从用户的角度来看，则是归属同一“应用”）。 任务的亲和关系由其根 Activity 的亲和关系确定。
亲和关系确定两件事 - Activity 更改到的父项任务（请参阅 allowTaskReparenting 属性）和通过 FLAG_ACTIVITY_NEW_TASK 标志启动 Activity 时将用来容纳它的任务。

> 默认情况下，应用中的所有 Activity 都具有相同的亲和关系。您可以设置该属性来以不同方式组合它们，甚至可以将在不同应用中定义的 Activity 置于同一任务内。 要指定 Activity 与任何任务均无亲和关系，请将其设置为空字符串。

> 如果未设置该属性，则 Activity 继承为应用设置的亲和关系（请参阅 <application> 元素的 taskAffinity 属性）。 应用默认亲和关系的名称是 <manifest> 元素设置的软件包名称。

我们来总结一下上边文字包含的信息：
- TASK的taskAffinity是有它的根Activity的taskAffinity决定的；
- Activity的taskAffinity默认值为所在应用程序的包名；
- taskAffinity影响Activity的任务（TASK）从属；
> 我们先来看看截取的一段堆栈信息：
```java
Running activities (most recent first):
      TaskRecord{f51ba40 #6608 A=com.tzx.launchmodel U=0 StackId=1 sz=2}
        Run #3: ActivityRecord{6158b8a u0 com.tzx.launchmodel/.BActivity t6608}
      TaskRecord{3e0d8d2 #6609 A=com.tzx.single U=0 StackId=1 sz=1}
        Run #2: ActivityRecord{d387da3 u0 com.tzx.launchmodel/.CActivity t6609}
      TaskRecord{f51ba40 #6608 A=com.tzx.launchmodel U=0 StackId=1 sz=2}
        Run #1: ActivityRecord{a0bfd79 u0 com.tzx.launchmodel/.AActivity t6608}
//该堆栈信息对应的AndroidManifest.xml:
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.tzx.launchmodel" >
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme" >
        <activity android:name=".AActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity android:name=".BActivity"/>
        <activity android:name=".CActivity"
                  android:taskAffinity="com.tzx.single"
            android:launchMode="singleInstance"/>
    </application>
</manifest>
```
> 其中TaskRecord中的A的值就是taskAffinity。

- taskAffinity一班与singleTask、singleInstance和allowTaskReparenting标签搭配使用。
> 这句话又怎么理解呢？
在解释这句时我先想和大家先聊聊Android手机在使用过程中为了手机软件运行的更加流畅，我们一般都会清理后台任务。这个任务列表是我们最能直观的看到任务的存在痕迹。在这里我们会看到很多情况：一个应用程序的所有Activity一个任务，多个应用程序的不同Activity一个任务，一个应用程序不同Activity在不同任务中。此时是不是感觉很懵逼，没关系接下来我会讲述为什么会这样：
<center>![taskAffinity.jpg](http://upload-images.jianshu.io/upload_images/1319879-0ea2325edccb02da.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>
> - 一个应用程序的所有Activity一个任务：一班情况下都这样（不设置taskAffinity）。
> - 多个应用程序的不同Activity一个任务：一个应用程序启动另外一个应用程序的Activity并且这里只能是standard或者singleTop模式。因为我们得出的第三个结论taskAffinity只会对singleTask、singleInstance有影响，所以在启动standard或者singleTop模式的Activity时不会对比taskAffinity，会直接在启动它的TASK中启动。而启动singleTask、singleInstance模式的Activity只能在对应的taskAffinity的TASK中。
> - 一个应用程序不同Activity在不同任务中：当在应用程序中我们启动singleTask模式的Activity时会寻找与该Activity的taskAffinity相同的TASK当中启动，如果没有则会新建一个TASK并且这个TASK在任务列表里展示。这时可能比较细心的同学会问singleInstance模式的Activity呢？它每次都会启动一个TASK，那么任务列表里面会展示这个TASK么？如果任务列表中不存在TASK与该singleInstance模式的Activity的taskAffinity
相同，那么该TASK出现中在任务列表。如果有那么不出现在任务列表。

android:allowTaskReparenting
---
android:allowTaskReparenting这个标签我们单独抽出来讲一下，为什么呢？因为我感觉android:allowTaskReparenting和taskAffinity没啥关系，至与launchMode有关！！！
>  [android:allowTaskReparenting 官网解释](https://developer.android.com/guide/topics/manifest/activity-element.html#reparent)
> 当启动 Activity 的任务接下来转至前台时，Activity 是否能从该任务转移至与其有亲和关系的任务 —“true”表示它可以转移，“false”表示它仍须留在启动它的任务处。
如果未设置该属性，则对 Activity 应用由 <application> 元素的相应 allowTaskReparenting 属性设置的值。 默认值为“false”。

> 正常情况下，当 Activity 启动时，会与启动它的任务关联，并在其整个生命周期中一直留在该任务处。您可以利用该属性强制 Activity 在其当前任务不再显示时将其父项更改为与其有亲和关系的任务。该属性通常用于使应用的 Activity 转移至与该应用关联的主任务。

> 例如，如果电子邮件包含网页链接，则点击链接会调出可显示网页的 Activity。 该 Activity 由浏览器应用定义，但作为电子邮件任务的一部分启动。 如果将其父项更改为浏览器任务，它会在浏览器下一次转至前台时显示，当电子邮件任务再次转至前台时则会消失。

> Activity 的亲和关系由 taskAffinity 属性定义。 任务的亲和关系通过读取其根 Activity 的亲和关系来确定。因此，按照定义，根 Activity 始终位于具有相同亲和关系的任务之中。 由于具有“singleTask”或“singleInstance”启动模式的 Activity 只能位于任务的根，因此更改父项仅限于“standard”和“singleTop”模式。 （另请参阅 launchMode 属性。）

上面文字叙述总结为一句话：android:allowTaskReparenting可以让Activity在TASK中转移，但该Activity时能是“standard”和“singleTop”模式。
至于为什么在讲taskAffinity的时候已经介绍清楚了。实用场景官网的描述中也有，大家可以参考使用。


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>