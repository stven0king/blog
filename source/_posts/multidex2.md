---
title: Multidex记录二：缺陷&解决
date: 2018-10-17 18:50:43
tags: [multidex]
categories: Android
description:  "为什么要用记录呢，因为我从开始接触Android时我们的项目就在65535的边缘。不久Google就出了multidex的解决方案。我们也已经接入multidex好多年，但我自己还没有接入，所以本博文只是作者自己对multidex接入中产生的问题以及解决方案做理解和记录。"
---

[Multidex记录一：介绍和使用](http://dandanlove.com/2018/10/16/multidex1/)
[Multidex记录二：缺陷&解决](http://dandanlove.com/2018/10/17/multidex2)
[Multidex记录三：源码解析](http://dandanlove.com/2018/10/18/multidex3)

# 记录Multidex缺陷&解决

为什么要用记录呢，因为我从开始接触Android时我们的项目就在65535的边缘。不久Google就出了multidex的解决方案。我们也已经接入multidex好多年，但我自己还没有接入，所以本博文只是作者自己对multidex接入中产生的问题以及解决方案做理解和记录。

## Multidex的缺陷
[Multidex介绍和使用](https://www.jianshu.com/p/9d9c2dbba223) 中已经说了一部分`multidex`的局限性：

- 1、在冷启动时因为需要安装DEX文件，如果DEX文件过大时，处理时间过长，很容易引发ANR（Application Not Responding）；
- 2、采用MultiDex方案的应用可能不能在低于Android 4.0 (API level 14) 机器上启动，这个主要是因为Dalvik linearAlloc的一个bug（问题 [22586](http://b.android.com/22586?hl=zh-cn)） ;
- 3、采用MultiDex方案的应用因为需要申请一个很大的内存，在运行时可能导致程序的崩溃，这个主要是因为Dalvik linearAlloc 的一个限制问题 [78035](http://b.android.com/78035?hl=zh-cn)），这个限制在 Android 4.0 (API level 14)已经增加了, 应用也有可能在低于 Android 5.0 (API level 21)版本的机器上触发这个限制。

Google官方给解决办法就是混淆、混淆~！~！

## Dalvik LinearAlloc

局限`2和3`都与`Dalvik LinearAlloc`，我们先来看一下`Dalvik LinearAlloc`是什么：

> 线性内存分配器LinearAlloc的目的在于简单、快速地分配只写一次(write-once)的内存（即分配并完成初始化写入后一般不会再改变，保持只读性质）

> LinearAlloc它主要用来管理Dalvik中类加载时的内存，因为类加载后通常是只读属性，而不需要去改变且在程序的整个运行周期都是有效的，同时它还有共享的特性，一个应用加载后其它进程可以共享使用这些已加载的类从而加快程序的启动和运行速度。

在Android版本不同分别经历了`4M/5M/8M/16M`限制，目前主流4.2.x系统上可能都已到16M， 在`Gingerbread`或者以下系统LinearAllocHdr分配空间只有5M大小的， 高于`Gingerbread`的系统提升到了8M。Dalvik linearAlloc是一个固定大小的缓冲区。在应用的安装过程中，系统会运行一个名为dexopt的程序为该应用在当前机型中运行做准备。dexopt使用LinearAlloc来存储应用的方法信息。Android 2.2和2.3的缓冲区只有5MB，Android 4.x提高到了8MB或16MB。当方法数量过多导致超出缓冲区大小时，会造成dexopt崩溃。

## LinearAlloc解决方法

这个问题实质上是dex过大的问题，因为我们使用的`multidex`，dx命令就已经支持：--multi-dex 参数来直接自动分包。

我们查看`dx`命令：

<center>![dx.png](https://upload-images.jianshu.io/upload_images/1319879-88ce8bec026b43be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)</center>

> multidex相关参数说明：
>- --multi-dex：多 dex 打包的开关
>- --main-dex-list=<file>：参数是一个类列表的文件，在该文件中的类会被打包在第一个 dex 中
>- --minimal-main-dex：只有在--main-dex-list 文件中指定的类被打包在第一个 dex，其余的都在第二个 dex 文件中。

发现并没有控制`dex`中方法数的参数，那么继续查看`dx`的源码，我们找到一个`maxNumberOfIdxPerDex`变量用来指定`dex`的最大方法数。

```
//65536
private int maxNumberOfIdxPerDex = DexFormat.MAX_MEMBER_IDX + 1;
```

同时又一个隐藏的`--set-max-idx-number`参数可以用来修改`maxNumberOfIdxPerDex` 的值：

<center>![--set-max-idx-number=.png](https://upload-images.jianshu.io/upload_images/1319879-6e14d5d56e970c01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)</center>

我们修改项目的`build.gradle`脚本：

```groovy
android.applicationVariants.all {
    variant ->
        dex.doFirst{
            dex->
            if (dex.additionalParameters == null) {
                dex.additionalParameters = []
            }
                dex.additionalParameters += '--set-max-idx-number=48000'
 
       }
}
```

`--set-max-idx-number=`用于控制每一个dex的最大方法个数，写小一点可以产生好几个dex。为了避免2.3机型runtime 的linearAlloclimit ,最好保持每一个dex体积<4M ,刚才的的`value<=48000`。

## Application Not Responding解决：

`Multidex`的安装是比较耗时的，所以如果放在主线程中就会产生ANR。

目前有两类解决办法：
> 放在异步线程；
> 放在其他进程（我们使用的是第二种，下边详细讲解）；

### 异步线程执`MultiDex.install

最有名的是美团的方案：精简主dex+异步加载secondary.dex 。对异步化执行速度的不确定性，他们的解决方案是重写Instrumentation execStartActivity 方法，hook跳转Activity的总入口做判断，如果当前secondary.dex 还没有加载完成，就弹一个loading Activity等待加载完成，如果已经加载完成那最好不过了。

局限性：第一个dex必须包含所有可能启动之后ClassLoader的类，不然一定会产生`NoClassDefFoundError `异常。Application的启动入口有多重，点击桌面icon只不过是其中的一种，而且有些时候启动Application不一定会打开Activity。

### 放在其他进程

微信团队的方案：
流程图：

<center>![泡在网上的日子](https://upload-images.jianshu.io/upload_images/1319879-aa1f994deb5c9c3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

>- 对现有代码改动量最小；
>- 该方案不关注Application被哪个组件启动。Activity ，Service ，Receiver ，ContentProvider 都满足（与美团方案都相同的问题，假如打开的不是Activity。这个时候弹出一个过渡的Activity就非常尴尬）；
>- 该方案不限制 Application ，Activity ，Service ，Receiver ，ContentProvider 继续新增业务；

实现代码：
[泡在网上的日子:其实你不知道MultiDex到底有多坑](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/1218/3789.html)

单独说一下`waitForDexopt`这个方法，这里设置的10s（Honeycomb之前20s）的轮询之后执行了`MultiDex.install`。此时在`mini`进程中`Multidex`可能还未完成安装（我们项目目前一共3个dex，`Multidex`的安装耗时大概20s）。

```java
public void waitForDexopt(Context base) {
    Intent intent = new Intent();
    ComponentName componentName = new
            ComponentName( "com.zongwu", LoadResActivity.class.getName());
    intent.setComponent(componentName);
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    base.startActivity(intent);
    long startWait = System.currentTimeMillis ();
    long waitTime = 10 * 1000 ;
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB_MR1 ) {
        waitTime = 20 * 1000 ;//实测发现某些场景下有些2.3版本有可能10s都不能完成optdex
    }
    while (needWait(base)) {
        try {
            long nowWait = System.currentTimeMillis() - startWait;
            LogUtils.d("loadDex" , "wait ms :" + nowWait);
            if (nowWait >= waitTime) {
                return;
            }
            Thread.sleep(200 );
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

当启动`:mini`进程后，主进程就会切换为后台进程所以不存在ANR的问题。我们可以一直轮询`needWait`，直到`Multidex`加载完成。

```java
public void waitForDexopt(Context base) {
    /***部分代码省略***/
    while (needWait(base)) {
        try {
            //long nowWait = System.currentTimeMillis() - startWait;
            //LogUtils.d("loadDex" , "wait ms :" + nowWait);
            //if (nowWait >= waitTime) {
            //    return;
            //}
            Thread.sleep(200 );
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 参考资料：
[泡在网上的日子:其实你不知道MultiDex到底有多坑](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/1218/3789.html)
[尼古拉斯_赵四:Android关于Dex拆分(MultiDex)技术详解](https://blog.csdn.net/jiangwei0910410003/article/details/50799573)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>