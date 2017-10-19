---
title: Android类加载之PathClassLoader和DexClassLoader
date: 2017-2-23 15:08:02
tags: [PathClassLoader,DexClassLoader]
categories: Android
description: " 上一章节我们讨论了JVM的类加载机制，那么Android中的类加载是什么样的呢？两者有什么相同和不同？"
---
![北京的初雪.jpg](http://upload-images.jianshu.io/upload_images/1319879-10c40968b6c6edcf.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上一篇文章 [自定义ClassLoader和双亲委派机制](/2017/02/23/java-classloader/) 讲述了JVM中的类的加载机制，Android也是类JVM虚拟机那么它的类加载机制是什么呢，我们来探究一下(PS：文章源码为Android5.1)。

前言
---
Android的Dalvik虚拟机和Java虚拟机的运行原理相同都是将对应的java类加载在内存中运行。而Java虚拟机是加载class文件，也可以将一段二进制流通过defineClass方法生产Class进行加载（PS: [自定义ClassLoader和双亲委派机制](/2017/02/23/java-classloader/) 文章后面的自定义类加载器就是通过这种方式实现的）。Dalvik虚拟机加载的dex文件。dex文件是Android对与Class文件做的优化，以便于提高手机的性能。可以想象dex为class文件的一个压缩文件。dex在Android中的加载和class在jvm中的相同都是基于双亲委派模型，都是调用ClassLoader的loadClass方法加载类。

Android系统中类加载的双亲委派机制
---
- Android5.1源码中ClassLoader的loadClass方法
```java
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);

        if (clazz == null) {
            ClassNotFoundException suppressed = null;
            try {
                //先让父类加载器加载
                clazz = parent.loadClass(className, false);
            } catch (ClassNotFoundException e) {
                suppressed = e;
            }
            //当所有父类节点的类加载器都没有找到该类时，当前加载器调用findClass方法加载。
            if (clazz == null) {
                try {
                    clazz = findClass(className);
                } catch (ClassNotFoundException e) {
                    e.addSuppressed(suppressed);
                    throw e;
                }
            }
        }
```

- 想要动态加载类，可以用 [自定义ClassLoader和双亲委派机制](/2017/02/23/java-classloader/) 中自定义ClassLoader的方法加载自己定义的class文件么？看看Android源码中的ClassLoader的findClass方法：
```java
protected Class<?> findClass(String className) throws ClassNotFoundException {
        throw new ClassNotFoundException(className);
    }
```
这个方法直接抛出了“ClassNotFoundException”异常，所以在Android中想通过这种方式实现类的加载时不行的。

Android系统中的类加载器
---
- Android系统屏蔽了ClassLoader的findClass加载方法，那么它自己的类加载时通过什么样的方式实现的呢？
 - Android系统中有两个类加载器分别为PathClassLoader和DexclassLoader。
 - PathClassLoader和DexClassLoader都是继承与BaseDexClassLoader，BaseDexClassLoader继承与ClassLoader。

提出问题
===
在这里我们先提一个问题Android为什么会将自己的类加载器派生出两个不同的子类，它们各自有什么用？

BaseDexClassLoader类加载
---
- 作为ClassLoader的子类，复写了父类的findClass方法。
```java
@Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        //在自己的成员变量DexPathList中寻找，找不到抛异常
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
```

- DexPathList的findClass方法
```java
public Class findClass(String name, List<Throwable> suppressed) {
        //循环便利成员变量dexElements，调用DexFile.loadClassBinaryName加载class
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;

            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
```

通过以上两段代码我们可以看出，虽然Android中的ClassLoader的findClass方法的实现被取消了，但是ClassLoader的基类BaseDexClassLoader实现了findClass方法取加载指定的Class。

PathClassLoader和DexClassLoader比较
---

- PathClassLoader
```java
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```

- DexClassLoader
```java
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```

- BaseDexClassLoader的构造函数
```java
    /**
     * Constructs an instance.
     *
     * @param dexPath the list of jar/apk files containing classes and
     * resources, delimited by {@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     * should be written; may be {@code null}
     * @param libraryPath the list of directories containing native
     * libraries, delimited by {@code File.pathSeparator}; may be
     * {@code null}
     * @param parent the parent class loader
     */
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
```
> - dexPath:指定的是dex文件地址，多个地址可以用":"进行分隔
 - optimizedDirectory:制定输出dex优化后的odex文件，可以为null
 - libraryPath:动态库路径（将被添加到app动态库搜索路径列表中）
 - parent:制定父类加载器，以保证双亲委派机制从而实现每个类只加载一次。

可以看出 PathClassLoader和DexClassLoader的区别就在于构造函数中optimizedDirectory这个参数。PathClassLoader中optimizedDirectory为null，DexClassLoader中为new File(optimizedDirectory)。

- optimizedDirectory的干活
BaseDexClassLoader的构造函数利用optimizedDirectory创建了一个DexPathList，看看DexPathList中optimizedDirectory:

```java
public DexPathList(ClassLoader definingContext, String dexPath,
        String libraryPath, File optimizedDirectory) {
    /******部分代码省略******/
    this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                       suppressedExceptions);
    /******部分代码省略******/
}
private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory,
                                         ArrayList<IOException> suppressedExceptions) {
   /******部分代码省略******/
    for (File file : files) {
        /******部分代码省略******/
        if (file.isDirectory()) {
           /******部分代码省略******/
        } else if (file.isFile()){
            if (name.endsWith(DEX_SUFFIX)) {
                // Raw dex file (not inside a zip/jar).
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ex) {
                    System.logE("Unable to load dex file: " + file, ex);
                }
            } else {
                zip = file;
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException suppressed) {
                    suppressedExceptions.add(suppressed);
                }
            }
        } else {
            System.logW("ClassLoader referenced unknown path: " + file);
        }
        if ((zip != null) || (dex != null)) {
            elements.add(new Element(file, false, zip, dex));
        }
    }
    return elements.toArray(new Element[elements.size()]);
}

private static DexFile loadDexFile(File file, File optimizedDirectory)
        throws IOException {
    if (optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
    }
}
```

从这里可以看出optimizedDirectory不同生产的DexFile对象不同，我们继续看看optimizedDirectory在DexFile中的作用：

```java
public DexFile(File file) throws IOException {
    this(file.getPath());
}

/**
 * Opens a DEX file from a given filename. This will usually be a ZIP/JAR
 * file with a "classes.dex" inside.
 *
 * The VM will generate the name of the corresponding file in
 * /data/dalvik-cache and open it, possibly creating or updating
 * it first if system permissions allow.  Don't pass in the name of
 * a file in /data/dalvik-cache, as the named file is expected to be
 * in its original (pre-dexopt) state.
 *
 * @param fileName
 *            the filename of the DEX file
 *
 * @throws IOException
 *             if an I/O error occurs, such as the file not being found or
 *             access rights missing for opening it
 */
public DexFile(String fileName) throws IOException {
    mCookie = openDexFile(fileName, null, 0);
    mFileName = fileName;
    guard.open("close");
    //System.out.println("DEX FILE cookie is " + mCookie + " fileName=" + fileName);
}

/**
 * Opens a DEX file from a given filename, using a specified file
 * to hold the optimized data.
 *
 * @param sourceName
 *  Jar or APK file with "classes.dex".
 * @param outputName
 *  File that will hold the optimized form of the DEX data.
 * @param flags
 *  Enable optional features.
 */
private DexFile(String sourceName, String outputName, int flags) throws IOException {
    if (outputName != null) {
        try {
            String parent = new File(outputName).getParent();
            if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
                throw new IllegalArgumentException("Optimized data directory " + parent
                        + " is not owned by the current user. Shared storage cannot protect"
                        + " your application from code injection attacks.");
            }
        } catch (ErrnoException ignored) {
            // assume we'll fail with a more contextual error later
        }
    }

    mCookie = openDexFile(sourceName, outputName, flags);
    mFileName = sourceName;
    guard.open("close");
    //System.out.println("DEX FILE cookie is " + mCookie + " sourceName=" + sourceName + " outputName=" + outputName);
}

static public DexFile loadDex(String sourcePathName, String outputPathName,
    int flags) throws IOException {
    return new DexFile(sourcePathName, outputPathName, flags);
}
```

从注释当中就可以看到`new DexFile(file)`的dex输出路径只能为`/data/dalvik-cache`，而`DexFile.loadDex()`的dex输出路径为自己输入的optimizedDirectory路径。

![dalvik-cache.jpg](http://upload-images.jianshu.io/upload_images/1319879-897567743fc325af.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决疑问
===
我们在文章开始提出的问题就这样一步步得到了答案。
> DexClassLoader：能够加载自定义的jar/apk/dex
 PathClassLoader：只能加载系统中已经安装过的apk
所以Android系统默认的类加载器为PathClassLoader，而DexClassLoader可以像JVM的ClassLoader一样提供动态加载。

总结
---
- ClassLoader的loadClass方法保证了双亲委派机。
- BaseDexClassLoader提供了两种派生类使我们可以加载自定义类。


想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>