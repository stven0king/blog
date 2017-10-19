---
title: Android之NDK开发初体验
date: 2017-3-24 23:51:00
tags: [JNI,NDK]
categories: Android
description: "作为Android开发人员，没有接触过NDK开发也接触过so文件吧。其实NDK编程也没有看来的那么隐秘，今天我们来看看咱们利用NDK来写出自己的so文件"
---
记得前年开始自己在项目中使用第三方so库的时候就接触NDK编程开发了，只不过哪个时候自己是输出了"Hello Wrold~!"。如今一年多的时间过去了，回头拾起之前的代码再次翻看。

# 概念
在阅读文章之前我们首先了解几个概念
## JNI
> JNI是[Java](http://lib.csdn.net/base/javase)语言提供的Java和C/C++相互沟通的机制，Java可以通过JNI调用本地的C/C++代码，本地的C/C++的代码也可以调用java代码。JNI 是本地编程接口，Java和C/C++互相通过的接口。Java通过C/C++使用本地的代码的一个关键性原因在于C/C++代码的高效性。

## NDK
> NDK是一系列工具的集合。它提供了一系列的工具，帮助开发者快速开发C（或C++）的动态库，并能自动将so和java应用一起打包成apk。这些工具对开发者的帮助是巨大的。它集成了交叉编译器，并提供了相应的mk文件隔离CPU、平台、ABI等差异，开发人员只需要简单修改mk文件（指出“哪些文件需要编译”、“编译特性要求”等），就可以创建出so。它可以自动地将so和Java应用一起打包，极大地减轻了开发人员的打包工作。

## ARM
> 早起Android只支持ARMv5的CPU架构，而发展到现在，支持一下7种架构：
<center>![arm.jpg](http://upload-images.jianshu.io/upload_images/1319879-0b680fa969199ac1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>
世界在进步，cup在arm基础上不断升级优化。每种架构关联着一种ABI（application binary interface应用程序二进制接口），所以每一种架构都对应一个.so文件，但都兼容arm。对于我们Android开发者来说，我们的app需要能在大多数手机上运行。所以要么我们所有arm类型都兼容，要么只兼容armeabi。兼容所有CPU架构类型是在性能上比较好，但是同时它也造成了apk体积的剧增（PS：我们之前的项目因为接入so库后导致apk体积剧增，最后只支持armeabi一种类型了）。

# 搭建环境

## Java环境配置（略）
## AndroidSDK环境配置（略）
## NDK环境配置
> 本文主要讲述NDK环境配置：
- [下载对应操作系统的NDK](https://developer.android.com/ndk/downloads/index.html)
- 解压文件（windows随意解压，Ubuntu解压在bin目录下）
- windows环境下配置
<center>![windows-ndk.jpg](http://upload-images.jianshu.io/upload_images/1319879-4c5a46d281287332.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>
- Ubuntu环境下配置
修改系统环境变量
sudo gedit /etc/profile
在profile文件下面添加,保存并退出
export ANDROID_NDK= ndk路径
export PATH=$ANDROID_NDK:$PATH
source  /etc/profile

查看是否配置成功
```java
im@58user:~/StudioProjects/NDKDemo/app/src/main/java$ ndk-build -v
GNU Make 3.81
Copyright (C) 2006  Free Software Foundation, Inc.
该程序为自由软件，详情可参阅版权条款。在法律允许的范围内
我们不作任何担保，这包含但不限于任何商业适售性以及针对特
定目的的适用性的担保。

 这个程序创建为 x86_64-pc-linux-gnu
```


## Android studio环境配置

<center>![android-ndk-env-config.jpg](http://upload-images.jianshu.io/upload_images/1319879-f1f46a7692aed25a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

以上是下边使用Android studio 进行NDK开发的基础，下边我们进入真正的开发环节。

# NDK开发环节

## native方法的定义
为了方便，我直接将native方法定义在了Activity当中
```java
public class MainActivity extends AppCompatActivity {
   //加载so库，libjnilib.so文件
    static {
        System.loadLibrary("jnilib");
    }
    //定义native方法
    private native String getStringForNative();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ((TextView) findViewById(R.id.text)).setText(getStringForNative());
    }
}
```

## gradle配置
```java
android {
	/**略**/
    defaultConfig {
        applicationId "ndk.tzx.com.ndkdemo"
        minSdkVersion 19
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"
        ndk {
			//定义生成的mk文件中的model名称
            moduleName "jnilib"
        }
    }
    sourceSets {
        main {
			//引入so路径
            jni.srcDirs = ['src/main/jni']
        }
    }
   /**略**/
}
```
## 创建jni目录
<center>![new-jni.jpg](http://upload-images.jianshu.io/upload_images/1319879-956f3a2906c26310.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

## 生成C++head文件
<center>![make-.c.jpg](http://upload-images.jianshu.io/upload_images/1319879-1db370f891c06e80.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

执行完改命令会在main/jni目录下生成对应的头文件

<center>![ndk-build.cpp.jpg](http://upload-images.jianshu.io/upload_images/1319879-57305812fe749d22.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>


## native方法的实现
然后我们在main/jni目录下创建cpp文件并进行native方法的实现
> - include头问件
- 实现方法
这一步经常有好多人会遇到错误，只因方法名写错~！~！

<center>![edit.cpp.jpg](http://upload-images.jianshu.io/upload_images/1319879-22ab06e993f8c745.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

## 构建并运行出结果
<center>![arm-&-mk.jpg](http://upload-images.jianshu.io/upload_images/1319879-b5d88ab642b4aef8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

> 上图是项目build后的结果，在app/build/intermediates/ndk/debug目录下有lib文件夹，obj文件夹和Android.mk文件。
在Android.mk这个文件当中我们定义生成so的名称，生成so对应cpp文件的路径和so输出的路径。
lib目录下我们可以看到各种类型的CPU架构下的so文件。

如果以上过程都没有问题的话，那么恭喜你整个项目就可以直接运行了。

# 踩坑需要一步一步来
build项目的时候遇到下边问题：
## Android.mk生成问题
直接在gradle.properties文件尾部添加 `android.useDeprecatedNdk=true`
<center>![ndk-intergration.jpg](http://upload-images.jianshu.io/upload_images/1319879-14020e5546f73286.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

## 生成so文件问题
```java
Error:Execution failed for task ':app:compileDebugNdk'.
> com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command '/bin/android-ndk-r13b/ndk-build.cmd'' finished with non-zero exit value 2
```
使用Android.md文件生成so的时候可能会遇到这样的问题：
解决办法1:
> 将Android.mk文件copy到jni目录下和.h与.cpp文件放在同一级目录，然后在该目录下执行`ndk-build`。 <center>![ndk-build.jpg](http://upload-images.jianshu.io/upload_images/1319879-6463def52a3229ec.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

这种方法也肯能报错：

```java
Error:(15) *** Android NDK: Aborting.    .  Stop.
Android NDK: /home/im/StudioProjects/NDKDemo/app/src/main/jni/Android.mk: Cannot find module with tag 'core' in import path    
Android NDK: Are you sure your NDK_MODULE_PATH variable is properly defined ?    
Android NDK: The following directories were searched:    
Android NDK:         
make: Entering directory `/home/im/StudioProjects/NDKDemo/app/src/main/jni'
make: Leaving directory `/home/im/StudioProjects/NDKDemo/app/src/main/jni'
:app:buildNative FAILED
Error:Execution failed for task ':app:buildNative'.
>Process 'command '/bin/android-ndk-r13b/ndk-build'' finished with non-zero exit value 2
```

遇到这种情况，偶查了很多资料最后才解决（参见解决方法2）

解决方法2：
> 安装最新的ndk（^_^）

## 运行问题
整个项目可以运行安装的时候是不是很爽，但是还可能遇到下边的问题：

```java
$ adb shell am start -n "ndk.tzx.com.ndkdemo/ndk.tzx.com.ndkdemo.MainActivity" -a android.intent.action.MAIN -c android.intent.category.LAUNCHER
Error while executing: am start -n "ndk.tzx.com.ndkdemo/ndk.tzx.com.ndkdemo.MainActivity" -a android.intent.action.MAIN -c android.intent.category.LAUNCHER
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=ndk.tzx.com.ndkdemo/.MainActivity }
Error type 3
Error: Activity class {ndk.tzx.com.ndkdemo/ndk.tzx.com.ndkdemo.MainActivity} does not exist.

Error while Launching activity
```

这问题偶也整了好久，网上大多数解释为native方法名不匹配，最后重新写cpp文件也成功解决。

心好累~！~！~！复习之前的东西还是要当初做好笔记啊。

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>