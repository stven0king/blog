---
title: Multidex记录一：介绍和使用
date: 2018-10-16 18:50:43
tags: [multidex]
categories: Android
description:  "为什么要用记录呢，因为我从开始接触Android时我们的项目就在65535的边缘。不久Google就出了multidex的解决方案。我们也已经接入multidex好多年，但我自己还没有接入，所以本博文只是作者自己对multidex接入整理记录，大部分来源于Android官网~！~！"
---


[Multidex记录一：介绍和使用](http://dandanlove.com/2018/10/16/multidex1/)
[Multidex记录二：缺陷&解决](http://dandanlove.com/2018/10/17/multidex2)
[Multidex记录三：源码解析](http://dandanlove.com/2018/10/18/multidex3)

# 记录Multidex介绍和使用

为什么要用记录呢，因为我从开始接触Android时我们的项目就在65535的边缘。不久Google就出了multidex的解决方案。我们也已经接入multidex好多年，但我自己还没有接入，所以本博文只是作者自己对multidex接入整理记录其中大部分来源于[Google官网](https://developer.android.com/studio/build/multidex?hl=zh-cn)。

<center>![image.png](https://upload-images.jianshu.io/upload_images/1319879-3ab8f197df6e81e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/415)</center>

## 背景
随着 Android 平台的持续成长，Android 应用的大小也在增加。当您的应用及其引用的库达到特定大小时，您会遇到构建错误，指明您的应用已达到 Android 应用构建架构的极限。早期版本的构建系统按如下方式报告这一错误：

```
Conversion to Dalvik format failed:Unable to execute dex: method ID not in [0, 0xffff]: 65536
```

> 超过最大方法数限制的问题，是由于DEX文件格式限制，一个DEX文件中method个数采用使用原生类型short来索引文件中的方法，也就是2个字节共计最多表达65536个method，field/class的个数也均有此限制。对于DEX文件，则是将工程所需全部class文件合并且压缩到一个DEX文件期间，也就是Android打包的DEX过程中， 单个DEX文件可被引用的方法总数（自己开发的代码以及所引用的Android框架、类库的代码）被限制为65536。

Google官方：[配置方法数超过 64K 的应用](https://developer.android.com/studio/build/multidex?hl=zh-cn)

### Android 5.0 之前版本的 Dalvik 可执行文件分包支持
> Android 5.0（API 级别 21）之前的平台版本使用 Dalvik 运行时来执行应用代码。默认情况下，Dalvik 限制应用的每个 APK 只能使用单个 classes.dex 字节码文件。要想绕过这一限制，您可以使用`multidex`，然后管理对其他 DEX 文件及其所包含代码的访问。

### Android 5.0 及更高版本的 Dalvik 可执行文件分包支持

> Android 5.0（API 级别 21）及更高版本使用名为 ART 的运行时，后者原生支持从 APK 文件加载多个 DEX 文件。ART 在应用安装时执行预编译，扫描 `classesN.dex` 文件，并将它们编译成单个 `.oat` 文件，供 Android 设备执行。因此，如果您的 `minSdkVersion `为 21 或更高值，则不需要 Dalvik 可执行文件分包支持库。

现在的Android设备市场还有大部分的`Android5.0`一下的手机，所以我们要使用`multidex`来解决应用在这些设备上的`65535`。

## 配置您的应用进行 Dalvik 可执行文件分包

将您的应用项目设置为使用 Dalvik 可执行文件分包配置需要对您的应用项目进行以下修改，具体取决于应用支持的最低 Android 版本。

### 修改gradle配置文件


如果您的`minSdkVersion` 设置为 21 或更高值，您只需在模块级 `build.gradle` 文件中将` multiDexEnabled` 设置为 true，如此处所示：

```groovy
android {
    defaultConfig {
        ...
        minSdkVersion 21 
        targetSdkVersion 26
        multiDexEnabled true
    }
    ...
}
```

但是，如果您的 minSdkVersion 设置为 20 或更低值，则Gradle 构建脚本依赖关系标识符如下所示：

```
compile 'com.android.support:multidex:1.0.2'
```

### 修改Application

- 如果您没有替换 `Application` 类，请编辑清单文件，按如下方式设置 `<application>` 标记中的 `android:name`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp">
    <application
            android:name="android.support.multidex.MultiDexApplication" >
        ...
    </application>
</manifest>
```
- 如果您替换了 `Application` 类，请按如下方式对其进行更改以扩展 `MultiDexApplication`（如果可能）：
```java
public class MyApplication extends MultiDexApplication { ... }
```
- 或者，如果您替换了 `Application` 类，但无法更改基本类，则可以改为替换 `attachBaseContext() `方法并调用 `MultiDex.install(this) `来启用 Dalvik 可执行文件分包：
```java
public class MyApplication extends SomeOtherApplication {
  @Override
  protected void attachBaseContext(Context base) {
     super.attachBaseContext(base);
     MultiDex.install(this);
  }
}
```

构建应用后，Android 构建工具会根据需要构建主 DEX 文件 (`classes.dex`) 和辅助 DEX 文件（`classes2.dex` 和 `classes3.dex` 等）。然后，构建系统会将所有 DEX 文件打包到您的 APK 中。

运行时，Dalvik 可执行文件分包 API 使用特殊的类加载器来搜索适用于您的方法的所有 DEX 文件（而不是仅在主 classes.dex 文件中搜索）。

## Dalvik 可执行文件分包支持库的局限性

- 在冷启动时因为需要安装DEX文件，如果DEX文件过大时，处理时间过长，很容易引发ANR（Application Not Responding）；
- 采用MultiDex方案的应用可能不能在低于Android 4.0 (API level 14) 机器上启动，这个主要是因为Dalvik linearAlloc的一个bug ;
- 采用MultiDex方案的应用因为需要申请一个很大的内存，在运行时可能导致程序的崩溃，这个主要是因为Dalvik linearAlloc 的一个限制，这个限制在 Android 4.0 (API level 14)已经增加了, 应用也有可能在低于 Android 5.0 (API level 21)版本的机器上触发这个限制。

## java.lang.NoClassDefFoundError

为 Dalvik 可执行文件分包构建每个 DEX 文件时，构建工具会执行复杂的决策制定来确定主要 DEX 文件中需要的类，以便应用能够成功启动。如果启动期间需要的任何类未在主 DEX 文件中提供，那么您的应用将崩溃并出现错误 `java.lang.NoClassDefFoundError`。

该情况不应出现在直接从应用代码访问的代码上，因为构建工具能识别这些代码路径，但可能在代码路径可见性较低（如使用的库具有复杂的依赖项）时出现。例如，如果代码使用自检机制或从原生代码调用 Java 方法，那么这些类可能不会被识别为主 DEX 文件中的必需项。

因此，如果您收到 `java.lang.NoClassDefFoundError`，则必须使用构建类型中的 `multiDexKeepFile` 或 `multiDexKeepProguard` 属性声明它们，以手动将这些其他类指定为主 DEX 文件中的必需项。如果类在 `multiDexKeepFile` 或 `multiDexKeepProguard` 文件中匹配，则该类会添加至主 DEX 文件。


### multiDexKeepFile 属性
您在 `multiDexKeepFile` 中指定的文件应该每行包含一个类，并且采用 `com/example/MyClass.class` 的格式。例如，您可以创建一个名为 `multidex-config.txt` 的文件，如下所示：

```
com/example/MyClass.class
com/example/MyOtherClass.class
```

然后，您可以按以下方式针对构建类型声明该文件：

```groovy
android {
    buildTypes {
        release {
            multiDexKeepFile file 'multidex-config.txt'
            ...
        }
    }
}
```
请记住，Gradle 会读取相对于 build.gradle 文件的路径，因此如果 multidex-config.txt 与 build.gradle 文件在同一目录中，以上示例将有效。

### multiDexKeepProguard 属性
`multiDexKeepProguard` 文件使用与 `Proguard` 相同的格式，并且支持整个 Proguard 语法。如需了解有关 `Proguard 格式和语法的详细信息，请参阅 Proguard 手册中的 [Keep Options](http://proguard.sourceforge.net/manual/usage.html#keepoptions) 一节。

您在 `multiDexKeepProguard` 中指定的文件应该在任何有效的 ProGuard 语法中包含 `-keep` 选项。例如，`-keep com.example.MyClass.class`。您可以创建一个名为 `multidex-config.pro` 的文件，如下所示：

```
-keep class com.example.MyClass
-keep class com.example.MyClassToo
```

如果您想要指定包中的所有类，文件将如下所示：

```
-keep class com.example.** { *; } // All classes in the com.example package
```

然后，您可以按以下方式针对构建类型声明该文件：



```groovy
android {
    buildTypes {
        release {
            multiDexKeepProguard 'multidex-config.pro'
            ...
        }
    }
}
```


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：


<center>>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>