---
title: Multidex记录三：源码解析
date: 2018-10-18 18:50:43
tags: [multidex]
categories: Android
description:  "为什么要用记录呢，因为我从开始接触Android时我们的项目就在65535的边缘。不久Google就出了multidex的解决方案。我们也已经接入multidex好多年，但我自己还没有接入，知其然而不知其所以然。出了问题只能自己找源码来分析。"
---

[Multidex记录一：介绍和使用](http://dandanlove.com/2018/10/16/multidex1/)
[Multidex记录二：缺陷&解决](http://dandanlove.com/2018/10/17/multidex2)
[Multidex记录三：源码解析](http://dandanlove.com/2018/10/18/multidex3)

# 记录Multidex源码解析

为什么要用记录呢，因为我从开始接触Android时我们的项目就在65535的边缘。不久Google就出了multidex的解决方案。我们也已经接入multidex好多年，但我自己还没有接入，知其然而不知其所以然。出了问题只能自己找源码来分析。前两篇文章 [Multidex记录一：介绍和使用](https://www.jianshu.com/p/9d9c2dbba223) 和 [Multidex记录二：缺陷&解决](https://www.jianshu.com/p/bf59ae5af9b9) 分别讲述了怎么接入和接入时遇到的问题，本博文只是对multidex源码学习过程中的分析和理解的记录。

关于`Multidex`的相关知识点前两章已经讲的差不多了，这篇文章只分析`Multidex`的安装。

## 流程图

<center>![multidex-flowchart.png](https://upload-images.jianshu.io/upload_images/1319879-4e2349ad6d44d94e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/420)</center>

## 源码分析

我们先来看看`MultiDex`的安装日志：

```java
I/MultiDex: VM with version 1.6.0 does not have multidex support
    Installing application
    MultiDexExtractor.load(/data/app/com.xxx.xxx-1.apk, false, )
I/MultiDex: Blocking on lock /data/data/com.xxx.xxx/code_cache/secondary-dexes/MultiDex.lock
    /data/data/com.xxx.xxx/code_cache/secondary-dexes/MultiDex.lock locked
    Detected that extraction must be performed.
I/MultiDex: Extraction is needed for file /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes2.zip
    Extracting /data/data/com.xxx.xxx/code_cache/secondary-dexes/tmp-com.xxx.xxx-1.apk.classes1415547735.zip
I/MultiDex: Renaming to /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes2.zip
    Extraction succeeded - length /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes2.zip: 4238720 - crc: 2971858359
    Extraction is needed for file /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes3.zip
    Extracting /data/data/com.xxx.xxx/code_cache/secondary-dexes/tmp-com.xxx.xxx-1.apk.classes-1615165740.zip
I/MultiDex: Renaming to /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes3.zip
    Extraction succeeded - length /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes3.zip: 3106018 - crc: 3138243730
    Extraction is needed for file /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes4.zip
    Extracting /data/data/com.xxx.xxx/code_cache/secondary-dexes/tmp-com.xxx.xxx-1.apk.classes-469912688.zip
I/MultiDex: Renaming to /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes4.zip
    Extraction succeeded - length /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx-1.apk.classes4.zip: 2163715 - crc: 1148318293
    load found 3 secondary dex files
I/MultiDex: install done
```

第二次启动时的日志：

```java
I/MultiDex: VM with version 1.6.0 does not have multidex support
    Installing application
    MultiDexExtractor.load(/data/app/com.xxx.xxx-1.apk, false, )
    Blocking on lock /data/data/com.xxx.xxx/code_cache/secondary-dexes/MultiDex.lock
    /data/data/com.xxx.xxx/code_cache/secondary-dexes/MultiDex.lock locked
I/MultiDex: loading existing secondary dex files
    load found 3 secondary dex files
    install done
```

### 初始化信息

```java
public final class MultiDex {
    static final String TAG = "MultiDex";
    //老版本dex文件存放路径
    private static final String OLD_SECONDARY_FOLDER_NAME = "secondary-dexes";
    //dex文件存放路径 code_cache/secondary-dexes
    private static final String SECONDARY_FOLDER_NAME;
    //Multidex最高支持的版本，大于20Android系统已支持
    private static final int MAX_SUPPORTED_SDK_VERSION = 20;
    //Multidex最低支持的版本
    private static final int MIN_SDK_VERSION = 4;
    //vm的版本信息
    private static final int VM_WITH_MULTIDEX_VERSION_MAJOR = 2;
    private static final int VM_WITH_MULTIDEX_VERSION_MINOR = 1;
    //apk路径
    private static final Set<String> installedApk;
    //是否支持Multidex
    private static final boolean IS_VM_MULTIDEX_CAPABLE;

    static {
    	//SECONDARY_FOLDER_NAME=code_cache/secondary-dexes
        SECONDARY_FOLDER_NAME = "code_cache" + File.separator + "secondary-dexes";
        installedApk = new HashSet();
        //VM是否已经支持自动Multidex
        IS_VM_MULTIDEX_CAPABLE = isVMMultidexCapable(System.getProperty("java.vm.version"));
    }
```

### Multidex安装

```java
public static void install(Context context) {
    //VM是否已经支持自动Multidex
    if (IS_VM_MULTIDEX_CAPABLE) {
        Log.i("MultiDex", "VM has multidex support, MultiDex support library is disabled.");
    } else if (VERSION.SDK_INT < 4) {
        //Multidex最低支持的版本
        throw new RuntimeException("Multi dex installation failed. SDK " + VERSION.SDK_INT + " is unsupported. Min SDK version is " + 4 + ".");
    } else {
        try {
            /***部分代码省略***/
            //多线程锁
            synchronized(installedApk) {
                //apkPath = data/data/com.xxx.xxx/
                String apkPath = applicationInfo.sourceDir;
                if (installedApk.contains(apkPath)) {
                    return;
                }
                installedApk.add(apkPath);
                /***部分代码省略***/
                //清除 /data/data/com.xxx.xxx/files/secondary-dexes 目录下的文件
                try {
                    clearOldDexDir(context);
                } catch (Throwable var8) {
                    Log.w("MultiDex", "Something went wrong when trying to clear old MultiDex extraction, continuing without cleaning.", var8);
                }
                //data/data/com.xxx.xxx/code_cache/secondary-dexes
                File dexDir = new File(applicationInfo.dataDir, SECONDARY_FOLDER_NAME);
                //解压apk，获得dex的zip文件列表
                List<File> files = MultiDexExtractor.load(context, applicationInfo, dexDir, false);
                //校验zip文件
                if (checkValidZipFiles(files)) {
                    //安装dex文件
                    installSecondaryDexes(loader, dexDir, files);
                } else {
                    //校验失败，重新执行解压（解压失败直接抛出异常）和安装
                    Log.w("MultiDex", "Files were not valid zip files.  Forcing a reload.");
                    files = MultiDexExtractor.load(context, applicationInfo, dexDir, true);
                    if (!checkValidZipFiles(files)) {
                        throw new RuntimeException("Zip files were not valid.");
                    }

                    installSecondaryDexes(loader, dexDir, files);
                }
            }
        }
        /***部分代码省略***/
    }
}
```

### Multidex获取dex文件

加载dex文件

```java
/**
 * @param context
 * @param applicationInfo
 * @param dexDir /data/data/com.xxx.xxx/code_cache/secondary-dexes/
 * @param forceReload 是否强制重新加载
 * @return 包含dex的zip文件列表
 * @throws IOException
 *             if an error occurs when writing the file.
 */
static List<File> load(Context context, ApplicationInfo applicationInfo, File dexDir, boolean forceReload) throws IOException {
    //data/data/com.xxx.xxx/
    File sourceApk = new File(applicationInfo.sourceDir);
    //apk的循环冗余校验码
    long currentCrc = getZipCrc(sourceApk);
    List files;
    //是否强制执行reload和是否已经解压过apk
    if (!forceReload && !isModified(context, sourceApk, currentCrc)) {
        try {
            //是否已存在zip文件
            files = loadExistingExtractions(context, sourceApk, dexDir);
        } catch (IOException var9) {
            files = performExtractions(sourceApk, dexDir);
            putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1);
        }
    } else {
        //获取apk中的classes2.dex并且压缩为zip文件
        files = performExtractions(sourceApk, dexDir);
        //存储当前apk的信息，作为下次有效缓存的明证
        putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1);
    }
    return files;
}
/**
 * @param context
 * @param timeStamp apk的最后一次修改时间
 * @param crc apk的循环冗余校验码
 * @param totalDexNumber apk中一共有几个dex
 */
private static void putStoredApkInfo(Context context, long timeStamp, long crc, int totalDexNumber) {
    SharedPreferences prefs = getMultiDexPreferences(context);
    Editor edit = prefs.edit();
    edit.putLong("timestamp", timeStamp);
    edit.putLong("crc", crc);
    edit.putInt("dex.number", totalDexNumber);
    apply(edit);
}
```

获取已经存在的dex的压缩包

```java
/**
 * @param context
 * @param dexDir /data/data/com.xxx.xxx/code_cache/secondary-dexes/
 * @param sourceApk data/app/com.xxx.xxx-1.apk
 * @return 包含dex的zip文件列表
 * @throws IOException
 *             if an error occurs when writing the file.
 */
private static List<File> loadExistingExtractions(Context context, File sourceApk, File dexDir) throws IOException {
    //extractedFilePrefix = com.xxx.xxx.apk.classes
    String extractedFilePrefix = sourceApk.getName() + ".classes";
    //totalDexNumber=apk中dex的数量
    int totalDexNumber = getMultiDexPreferences(context).getInt("dex.number", 1);
    List<File> files = new ArrayList(totalDexNumber);
    //主dex已经加载过了，加载class2.dex,class3.dex......
    for(int secondaryNumber = 2; secondaryNumber <= totalDexNumber; ++secondaryNumber) {
        //fileName = com.xxx.xxx.apk.classes2.zip
        String fileName = extractedFilePrefix + secondaryNumber + ".zip";
        //extractedFile = data/data/com.xxx.xxx/code_cache/secondary-dexes/com.wuba.bangjob-1.apk.classes2.zip
        File extractedFile = new File(dexDir, fileName);
        if (!extractedFile.isFile()) {
            throw new IOException("Missing extracted secondary dex file '" + extractedFile.getPath() + "'");
        }

        files.add(extractedFile);
        //校验zip文件是否完整
        if (!verifyZipFile(extractedFile)) {
            Log.i("MultiDex", "Invalid zip file: " + extractedFile);
            throw new IOException("Invalid ZIP file.");
        }
    }

    return files;
}
```

生成dex的压缩zip文件

```java
/**
 * @param sourceApk data/app/com.xxx.xxx-1.apk
 * @param dexDir /data/data/com.xxx.xxx/code_cache/secondary-dexes/
 * @return 包含dex的zip文件列表
 * @throws IOException
 *             if an error occurs when writing the file.
 */
private static List<File> performExtractions(File sourceApk, File dexDir) throws IOException {
    //extractedFilePrefix = com.xxx.xxx.apk.classes
    String extractedFilePrefix = sourceApk.getName() + ".classes";
    //删除data/data/com.xxx.xxx/code_cache/secondary-dexes/目录下的文件
    prepareDexDir(dexDir, extractedFilePrefix);
    List<File> files = new ArrayList();
    ZipFile apk = new ZipFile(sourceApk);
    try {
        //从class2.dex开始
        int secondaryNumber = 2;
        for(ZipEntry dexFile = apk.getEntry("classes" + secondaryNumber + ".dex"); dexFile != null; dexFile = apk.getEntry("classes" + secondaryNumber + ".dex")) {
            //fileName = com.xxx.xxx.apk.classes2.zip
            String fileName = extractedFilePrefix + secondaryNumber + ".zip";
            File extractedFile = new File(dexDir, fileName);
            files.add(extractedFile);
            //最多三次尝试生成dex的zip文件
            int numAttempts = 0;
            //标识是否生成dex的zip文件
            boolean isExtractionSuccessful = false;
            while(numAttempts < 3 && !isExtractionSuccessful) {
                ++numAttempts;
                extract(apk, dexFile, extractedFile, extractedFilePrefix);
                //校验压缩文件是否完整，否则删除重来
                isExtractionSuccessful = verifyZipFile(extractedFile);
                if (!isExtractionSuccessful) {
                    extractedFile.delete();
                    if (extractedFile.exists()) {
                        Log.w("MultiDex", "Failed to delete corrupted secondary dex '" + extractedFile.getPath() + "'");
                    }
                }
            }

            if (!isExtractionSuccessful) {
                throw new IOException("Could not create zip file " + extractedFile.getAbsolutePath() + " for secondary dex (" + secondaryNumber + ")");
            }

            ++secondaryNumber;
        }
    }/***部分代码省略***/
    return files;
}
```

将classes2.dex放入zip文件中

```java
/**
 * @param apk apk的压缩包
 * @param dexFile apk中的classes2.dex文件
 * @param extractTo /data/data/com.xxx.xxx/code_cache/secondary-dexes/com.xxx.xxx.apk.classes2.zip
 * @param extractedFilePrefix  com.wuba.bangjob-1.apk.classes
 * @throws IOException
 *             if an error occurs when writing the file.
 */
private static void extract(ZipFile apk, ZipEntry dexFile, File extractTo, String extractedFilePrefix) throws IOException, FileNotFoundException {
    InputStream in = apk.getInputStream(dexFile);
    ZipOutputStream out = null;
    ///data/data/com.wuba.bangjob/code_cache/secondary-dexes/com.xxx.xxx.apk.classes1415547735.zip
    File tmp = File.createTempFile(extractedFilePrefix, ".zip", extractTo.getParentFile());
    try {
        out = new ZipOutputStream(new BufferedOutputStream(new FileOutputStream(tmp)));
        try {
            //将classes2.dex进行压缩内容名称为classes.dex
            ZipEntry classesDex = new ZipEntry("classes.dex");
            classesDex.setTime(dexFile.getTime());
            out.putNextEntry(classesDex);
            byte[] buffer = new byte[16384];
            for(int length = in.read(buffer); length != -1; length = in.read(buffer)) {
                out.write(buffer, 0, length);
            }

            out.closeEntry();
        } finally {
            out.close();
        }
        //重命名文件为com.xxx.xxx.apk.classes2.zip
        if (!tmp.renameTo(extractTo)) {
            throw new IOException("Failed to rename \"" + tmp.getAbsolutePath() + "\" to \"" + extractTo.getAbsolutePath() + "\"");
        }
    }/***部分代码省略***/
}
```

### dex文件的装载

将含有加载含有dex的压缩包进行夹杂，相关知识点参考：[Android类加载之PathClassLoader和DexClassLoader](https://www.jianshu.com/p/4b4f1fa6633c)。
为什么需要做版本的区分，就是因为版本见类加载的实现是有些差异的。

```java
/**
 * @param loader
 * @param dexDir /data/data/xxx.xxx.xxx/code_cache/secondary-dexes/
 * @param files dex的压缩zip文件
 */
private static void installSecondaryDexes(ClassLoader loader, File dexDir, List<File> files) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException, IOException {
    if (!files.isEmpty()) {
        if (VERSION.SDK_INT >= 19) {//KitKat(19)
            MultiDex.V19.install(loader, files, dexDir);
        } else if (VERSION.SDK_INT >= 14) {//IceCreamSandwich(14,15),JellyBean(16,17,18)
            MultiDex.V14.install(loader, files, dexDir);
        } else {
            MultiDex.V4.install(loader, files);
        }
    }
}
```

[Android类加载之PathClassLoader和DexClassLoader](https://www.jianshu.com/p/4b4f1fa6633c) 这篇文章将的是`VERSION.SDK_INT >= 19` 那么我们先看看第一个case：

```java
private static final class V19 {
    private V19() {
    }
    /**
     * @param loader PathClassLoader
     * @param additionalClassPathEntries dex压缩包路径
     * @param optimizedDirectory opt之后的dex文件目录
     */
    private static void install(ClassLoader loader, List<File> additionalClassPathEntries, File optimizedDirectory) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
        //PathClassLoader的DexPathList类型变量pathList
        Field pathListField = MultiDex.findField(loader, "pathList");
        Object dexPathList = pathListField.get(loader);
        ArrayList<IOException> suppressedExceptions = new ArrayList();
        //进行dex的opt并合并dex的Element
        MultiDex.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory, suppressedExceptions));
        //处理异常
        if (suppressedExceptions.size() > 0) {
            Iterator i$ = suppressedExceptions.iterator();
            while(i$.hasNext()) {
                IOException e = (IOException)i$.next();
                Log.w("MultiDex", "Exception in makeDexElement", e);
            }
            //一直感觉Google源码这块有些问题，loader应该改为dexPathList。因为dexElementsSuppressedExceptions变量是属于DexPathList的成员
            Field suppressedExceptionsField = MultiDex.findField(loader, "dexElementsSuppressedExceptions");
            IOException[] dexElementsSuppressedExceptions = (IOException[])((IOException[])suppressedExceptionsField.get(loader));
            if (dexElementsSuppressedExceptions == null) {
                dexElementsSuppressedExceptions = (IOException[])suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
            } else {
                IOException[] combined = new IOException[suppressedExceptions.size() + dexElementsSuppressedExceptions.length];
                suppressedExceptions.toArray(combined);
                System.arraycopy(dexElementsSuppressedExceptions, 0, combined, suppressedExceptions.size(), dexElementsSuppressedExceptions.length);
                dexElementsSuppressedExceptions = combined;
            }
            suppressedExceptionsField.set(loader, dexElementsSuppressedExceptions);
        }

    }
    //调用DexPathList的makeDexElements方法进行dex的opt
    private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files, File optimizedDirectory, ArrayList<IOException> suppressedExceptions) throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {
        Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements", ArrayList.class, File.class, ArrayList.class);
        return (Object[])((Object[])makeDexElements.invoke(dexPathList, files, optimizedDirectory, suppressedExceptions));
    }
}
```

`19 > VERSION.SDK_INT >= 14`的处理支持比`VERSION.SDK_INT >= 19`少了异常处理。是因为DexPathList的`makeDexElements`方法有了修改。

```java
/**
 * VERSION.SDK_INT >= 19
 */
private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory,
        ArrayList<IOException> suppressedExceptions);

/**
 * 19 > VERSION.SDK_INT >= 14
 */
private static Element[] makeDexElements(ArrayList<File> files,File optimizedDirectory);
```

`VERSION_SDK_INT < 14` 的情况和前面的都不相同是因为`PathClassLoader `的是实现就不相同。。。。

```java
/**
 * VERSION.SDK_INT == 13
 */
public class PathClassLoader extends ClassLoader {

    private final String path;
    private final String libPath;

    /*
     * Parallel arrays for jar/apk files.
     *
     * (could stuff these into an object and have a single array;
     * improves clarity but adds overhead)
     */
    private final String[] mPaths;
    private final File[] mFiles;
    private final ZipFile[] mZips;
    private final DexFile[] mDexs;
    /***部分代码省略***/
}
```

### apk解压结果预览

<center>![image.png](https://upload-images.jianshu.io/upload_images/1319879-08ade64f6cfb7fee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)</center>




文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

