---
title: Android更新资源文件浅思考
date: 2018-10-27 18:17:43
tags: [hotfix,AssetManager]
categories: Android
description:  "最近在看 《深入探索Android热修复技术原理7.3Q.pdf》时，遇到一个之前没有注意过的问题：关于资源修更新的Android的版本兼容？作为程序员我们需要非常严谨的思路，是什么导致了资源的修复更新需要做版本兼容？"
---


# 前言

最近在看 [《深入探索Android热修复技术原理7.3Q.pdf》](http://p3ry9qsas.bkt.clouddn.com/Android_hotfix.pdf) 时，遇到一个之前没有注意过的问题：关于资源修更新的Android的版本兼容？作为程序员我们需要非常严谨的思路，是什么导致了资源的修复更新需要做版本兼容？

这个问题是使我写下这边文章的原因，下边我们带着问题来找答案~！~！~！

这个问题的解释网上答案比较少,在滴滴的插件化框架相关文章 [VirtualAPK 资源篇](https://www.notion.so/VirtualAPK-1fce1a910c424937acde9528d2acd537) 和 [阿里云移动热修复(Sophix)](https://www.aliyun.com/product/hotfix?spm=a2c4e.11153940.blogcont96378.11.18b377dcTy4deX) 相关文章
[Android热修复升级探索——资源更新之新思路](https://yq.aliyun.com/articles/96378) 中 都有一句概括性质的话语：

> AndroidL之后资源在初始化之后可以加载，而在AndroidL之前是不可以的。因为在Android KK及以下版本，addAssetPath只是把补丁包的路径添加到了mAssetPath中，而真正解析的资源包的逻辑是在app第一次执行AssetManager::getResTable的时候。

# FTSC

为了比较完整的对前面提出的问题做解答，下边我在老罗写的 [Android应用程序资源管理器（Asset Manager）的创建过程分析](https://blog.csdn.net/luoshengyang/article/details/8791064)  这篇文章的基础上分析。

## 跟踪getResourceText

我们都知道在Android中获取资源调用的 `Resources.getText(int id)` 内部都是在调用 `AssetManager.getResourceText(id)` 真正对资源进行管理的是 `AssetManager`。

下边我们就以 `getResourceText` 方法的调用顺序引子查找：

```java
public final class AssetManager {
    ......
    /*package*/ static AssetManager sSystem = null;
    private native final void init(boolean isSystem);
    ......
    //构造方法
    public AssetManager() {
        synchronized (this) {
            ......
            init(false);
            ......
            //每个AssetManager实例都会初始化系统的资源
            ensureSystemAssets();
        }
    }
    private static void ensureSystemAssets() {
        synchronized (sSync) {
            if (sSystem == null) {
                AssetManager system = new AssetManager(true);
                system.makeStringBlocks(false);
                sSystem = system;
            }
        }
    }
    ......
    //添加资源路径
    public final int addAssetPath(String path) {
        synchronized (this) {
            int res = addAssetPathNative(path);
            if (mStringBlocks != null) {
                makeStringBlocks(mStringBlocks);
            }
            return res;
        }
    }
    private native final int addAssetPathNative(String path);

    /*package*/ final CharSequence getResourceText(int ident) {
        synchronized (this) {
            TypedValue tmpValue = mValue;
            int block = loadResourceValue(ident, (short) 0, tmpValue, true);
            if (block >= 0) {
                if (tmpValue.type == TypedValue.TYPE_STRING) {
                    return mStringBlocks[block].get(tmpValue.data);
                }
                return tmpValue.coerceToString();
            }
        }
        return null;
    }
    //查找并加载资源
    private native final int loadResourceValue(int ident, short density, TypedValue outValue,
            boolean resolve);
}
```

AssetManager.init方法的C层实现：

```cpp
//frameworks/base/core/jni/android_util_AssetManager.cpp
static void android_content_AssetManager_init(JNIEnv* env, jobject clazz, jboolean isSystem)
{
    if (isSystem) {
        verifySystemIdmaps();
    }
    //构造C++层的AssetManager的对象
    AssetManager* am = new AssetManager();
    if (am == NULL) {
        jniThrowException(env, "java/lang/OutOfMemoryError", "");
        return;
    }
    //添加系统资源
    am->addDefaultAssets();
    ALOGV("Created AssetManager %p for Java object %p\n", am, clazz);
    env->SetLongField(clazz, gAssetManagerOffsets.mObject, reinterpret_cast<jlong>(am));
}
```

AssetManager.loadResourceValue方法的C层实现：

```
//frameworks/base/core/jni/android_util_AssetManager.cpp
static jint android_content_AssetManager_loadResourceValue(JNIEnv* env, jobject clazz, jint ident, jshort density, jobject outValue, jboolean resolve)
{
    /***部分代码省略***/
    //这行代码最重要，通过获取C层的AssetManager的成员变量ResTable来获取资源
    const ResTable& res(am->getResources());
    Res_value value;
    ResTable_config config;
    uint32_t typeSpecFlags;
    ssize_t block = res.getResource(ident, &value, false, density, &typeSpecFlags, &config);
    /***部分代码省略***/
    if (block >= 0) {
        return copyValue(env, outValue, &res, value, ref, block, typeSpecFlags, &config);
    }
    return static_cast<jint>(block);
}
//getResources实际获取的是ResTable
const ResTable& AssetManager::getResources(bool required) const
{
    const ResTable* rt = getResTable(required);
    return *rt;
}
```

我们可以看到获取资源实际是在操作 `C层的AssetManager的成员变量ResTable` 。如果没有将资源加入到 `ResTable` 那么是无法获取到的。下边我们分别看看AndroidL和AndroidL之前 `addAssetPath ` 方法的实现。

## ResTable的构造

```cpp
//frameworks/base/libs/androidfw/AssetManager.cpp
const ResTable* AssetManager::getResTable(bool required) const
{
    ResTable* rt = mResources;
    if (rt) {
        return rt;
    }
    /***部分代码省略***/
    const size_t N = mAssetPaths.size();
    for (size_t i=0; i<N; i++) {
        //遍历mAssetPaths将其ResTable中
        /***部分代码省略***/
    }
    /***部分代码省略***/
    return rt;
}
```

如果已经创建那么直接返回，如果没有那么将 `mAssetPaths` 集合中的资源路径全部添加到 `ResTable` 中后返回。下边我们继续看看资源的路径是怎么被插入到 `mAssetPaths` 中的，或者是怎么被直接插入到 `ResTable` 中的。

## addAssetPath的差异

```cpp
//frameworks/base/libs/androidfw/AssetManager.cpp
bool AssetManager::addDefaultAssets()
{
    const char* root = getenv("ANDROID_ROOT");
    LOG_ALWAYS_FATAL_IF(root == NULL, "ANDROID_ROOT not set");
    String8 path(root);
    path.appendPath(kSystemAssets);
    String8 pathCM(root);
    pathCM.appendPath(kCMSDKAssets);
    return addAssetPath(path, NULL) & addAssetPath(pathCM, NULL);
}


bool AssetManager::addAssetPath(const String8& path, int32_t* cookie)
{
    AutoMutex _l(mLock);
    asset_path ap;
    String8 realPath(path);
    if (kAppZipName) {
        realPath.appendPath(kAppZipName);
    }
    /***部分代码省略***/
    //将path添加到mAssetPaths中
    mAssetPaths.add(ap);
    if (mResources != NULL) {
        size_t index = mAssetPaths.size() - 1;
        //添加到ResTable中
        appendPathToResTable(ap, &index);
    }
    // new paths are always added at the end
    if (cookie) {
        *cookie = static_cast<int32_t>(mAssetPaths.size());
    }
    /***部分代码省略***/
    return true;
}
```

以上是Android5.1的源码，我们发现无论是否初始化过 `ResTable` 我们都可以直接调用 `addAssetPath ` 是可以添加资源。


下边我们看看Android4.4的源码：

```cpp
//frameworks/base/libs/androidfw/AssetManager.cpp
bool AssetManager::addAssetPath(const String8& path, void** cookie)
{
    AutoMutex _l(mLock);
    asset_path ap;
    String8 realPath(path);
    if (kAppZipName) {
        realPath.appendPath(kAppZipName);
    }
    /***部分代码省略***/
    //将path添加到mAssetPaths中
    mAssetPaths.add(ap);
    // new paths are always added at the end
    if (cookie) {
        *cookie = (void*)mAssetPaths.size();
    }
    /***部分代码省略***/
    return true;
}
```

我们发现 `addAssetPath` 只是将 `path` 添加到了 `mAssetPaths` 里面，但是并没法添加到 `ResTable` 中。
如果AndroidL之前调用 `addAssetPath` 没有初始化 `ResTable` 那么这次添加就是有效的，否则添加无效。下边我们接着看看 `ResTable` 的初始化时机。

## ResTable的初始化时机

```java
public final class AssetManager {
    /***部分代码省略***/
    //构造方法
    public AssetManager() {
        synchronized (this) {
            ......
            init(false);
            ......
            //每个AssetManager实例都会初始化系统的资源
            ensureSystemAssets();
        }
    }
    private static void ensureSystemAssets() {
        synchronized (sSync) {
            if (sSystem == null) {
                AssetManager system = new AssetManager(true);
                //初始化系统的字符串块
                system.makeStringBlocks(false);
                sSystem = system;
            }
        }
    }
    /*package*/ final void makeStringBlocks(StringBlock[] seed) {
        final int seedNum = (seed != null) ? seed.length : 0;
        final int num = getStringBlockCount();
        mStringBlocks = new StringBlock[num];
        if (localLOGV) Log.v(TAG, "Making string blocks for " + this
                + ": " + num);
        for (int i=0; i<num; i++) {
            if (i < seedNum) {
                mStringBlocks[i] = seed[i];
            } else {
                mStringBlocks[i] = new StringBlock(getNativeStringBlock(i), true);
            }
        }
    }
    private native final int getStringBlockCount();
    /***部分代码省略***/
}
```

避免大家往上翻看，我又将 `AssetManager` 的沟通方法摘出来放大家看看。我们在初始化系统的资源时调用了 `AssetManager` 的 `makeStringBlocks` 方法，最后调用了 `C层` 的 `getStringBlockCount` 方法。

AssetManager.getStringBlockCount的C层的实现：

```cpp
//frameworks/base/core/jni/android_util_AssetManager.cpp
static jint android_content_AssetManager_getStringBlockCount(JNIEnv* env, jobject clazz)
{
    AssetManager* am = assetManagerForJavaObject(env, clazz);
    if (am == NULL) {
        return 0;
    }
   //初次调用资源，进行初始化ResTable
    return am->getResources().getTableCount();
}
```

好了，我们找到了 `ResTable` 的时机，它是发生在java层的 `AssetManager` 构造的时候。

# 结论

>- Android中进行资源管理的是 `AssetManager`；
>- 资源由C层 `AssetManager` 的 `ResTable` 提供；
>- `ResTable` 构造是遍历 `mAssetPaths` 中的资源路径；
>- AndroidL `addAssetPath` 方法可以直接将资源路添加到 `ResTable` 中使用；
>- AndroidL之前 `addAssetPath` 方法只是将资源路径添加到了`mAssetPaths` 中；
>- `ResTable` 构造包含在Java层 `AssetManager` 的构造中的；

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：


<center>>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>