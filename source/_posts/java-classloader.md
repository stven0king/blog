---
title: 自定义ClassLoader和双亲委派机制
date: 2017-2-23 14:25:02
tags: [ClassLoader,双亲委派机制]
categories: Java
description: "对于类的加载机制你了解多少？"
---
博文主要讲classloader的模型、作用和使用，内容是作者学习java反射机制有关知识时记录的笔记。
ClassLoader
---
ClassLoad：类加载器（class loader）用来加载 Java 类到 Java 虚拟机中。Java 源程序（.java 文件）在经过 Java 编译器编译之后就被转换成 Java 字节代码（.class 文件）。类加载器负责读取 Java 字节代码，并转换成 java.lang.Class 类的一个实例。 
双亲委派机制
===
某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。

自己对JVM中的ClassLoader和双亲委派机制的一些理解：
===
* java虚拟机中的class其实都是通过classloader来装载的
* 只有当你使用该class的时候才会去装载，一个classloader只会装载同一个class一次。 
* 不同的类加载器的实例所加载的字节码文件，其通过反射获取的对象不是相同类型（相互赋值会抛出类型强转异常）。即：判断两个类是否为同一对象的标准里面有一条是类加载器必须为相同。
* 双亲委派机制能在很大程度上防止内存中出现多个相同的字节码文件。
* 在加载类的时候默认会使用当前类的ClassLoader进行加载（类A中引用了类B，JVM会用类A的类加载器加载类B）。
* 在线程中加载一个类的时候：当前线程的类加载器可以通过Thread类的getContextClassLoader()获得，也可以通过setContextClassLoader()自己设置类加载器（	PS：自己没有试验过）。
 
ClassLoader体系结构图
---
![ClassLoader体系结构图](http://upload-images.jianshu.io/upload_images/1319879-037f2e2c51de5bb5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
package com.tzx.reflection;

public class MyClassLoader {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		System.out.println(Thread.currentThread().getContextClassLoader());
		System.out.println(ClassLoaderTest.class.getClassLoader());
		System.out.println(System.class.getClassLoader());
		System.out.println(ClassLoader.getSystemClassLoader());
		System.out.println(ClassLoader.getSystemClassLoader().getParent());
		System.out.println(ClassLoader.getSystemClassLoader().getParent().getParent());
	}

}
```
运行结果：

```
sun.misc.Launcher$AppClassLoader@4aa298b7
sun.misc.Launcher$AppClassLoader@4aa298b7
null
sun.misc.Launcher$AppClassLoader@382f3bf0
sun.misc.Launcher$ExtClassLoader@25082661
null
```

按照双亲委派机制加载类，每当需要加载一个新的类时，当前的类加载器会先委托其父加载器，查询有没有加载该类。如果父类加载器已近加载该类，那么直接返回加载的class对象，如果没有那么继续向上寻找父类加载器，如果在祖宗类加载器Bootstrap都没有加载该类，那么需要当前的类加载器自己加载，如果当前的类加载器也不能加载则会跑出`ClassNotFoundException`异常 (PS:类加载器没有向下寻找，没有getChild只有getParent)。

用这种思想去解析上边代码：Thread.currentThread().getContextClassLoader()指出当前的类加载器是AppClassLoader，需要加载MyClassLoader.class先在父类加载器（ExtClassLoader）中寻找，没有再向祖宗类加载中寻找（Bootstrap ClassLoader），还没有找到那么AppClassLoader自己去加载。


JVM中的类加载器类型：
===
* (Bootstrap ClassLoader)启动类加载器:
负责加载java_home/jar/lib/rt.jar目录下的核心类或- Xbootclasspath指定目录下的类。由于引导类加载器全部是native代码来实现的并且涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用，所以不允许直接通过引用进行操作，从上面的ClassLoader体系结构图中可以看出在java代码中获取启动类加载器为null。

![rt.jar.png](http://upload-images.jianshu.io/upload_images/1319879-15a6783f4a84fe82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

像System.java这样的由系统提供的类都在rt.jar中，由Bootstrap ClassLoader加载，由于Bootstrap类加载器不是Java写的，所以打印出来的类名为null。


* (Extension)扩展类加载器：负责加载java_home/lib/ext目录下的扩展类或 -Djava.ext.dirs 指定目录下的类。 开发者可以直接使用标准扩展类加载器。

我们将第一段代码生产的MyClassLoader.class文件打包成jar（[java打包成jar|执行jar包中的main方法](http://www.jianshu.com/p/c23d26873ec3)），放在java_home/jar/lib/ext目录下。

![ext.png](http://upload-images.jianshu.io/upload_images/1319879-40d666ccd6046298.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再次执行该java程序
```
im@58user:/usr/lib/jvm/jdk1.8.0_101/jre/lib/ext$ java -cp MyClassLoader.jar com.loadclass.demo.ClassLoaderTest start
sun.misc.Launcher$AppClassLoader@55f96302
sun.misc.Launcher$ExtClassLoader@70dea4e
null
sun.misc.Launcher$AppClassLoader@55f96302
sun.misc.Launcher$ExtClassLoader@70dea4e
null
```
通过终端输出结果我们可以看到执行ClassLoaderTest程序的类加载器是AppClassLoader，但加载ClassLoaderTest类的类加载器是ExtClassLoader。因为java_home/jar/lib/ext/*.jar在执行程序之前就被ExtClassLoader类加载器加载过了。***这样避免了类的重复加载～！～！***

* (System)类加载器：负责加载-classpath/-Djava.class.path所指的目录下的类。开发者可以直接使用标准扩展类加载器。一般来说，Java应用的类都是由他来完成加载的。
除了以上三种类型的类加载器，还有一个中比较特殊的：线程上下文类加载器（PS:暂时没做这方面的记录）。


自定义类加载器
---
* 定义

```
package com.tzx.reflection;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

public class MyClassLoader extends ClassLoader{
    static {
        System.out.println("MyClassLoader");
    }
    public static final String driver = "/home/im/Desktop/";
    public static final String fileTyep = ".class";

    public Class findClass(String name) {
        byte[] data = loadClassData(name);
        return defineClass(data, 0, data.length);
    }

    public byte[] loadClassData(String name) {
        FileInputStream fis = null;
        byte[] data = null;
        try {
            File file = new File(driver + name + fileTyep);
            System.out.println(file.getAbsolutePath());
            fis = new FileInputStream(file);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int ch = 0;
            while ((ch = fis.read()) != -1) {
                baos.write(ch);
            }
            data = baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
            System.out.println("loadClassData-IOException");
        }
        return data;
    }
}

```
* 使用

```
public class ClassLoaderTest {
    public static void main(String[] args) {
        MyClassLoader cl1 = new MyClassLoader();
        //磁盘中/home/im/Desktop/Hello.class文件存在
        try {
            Class c1 = cl1.loadClass("Hello");
            Object object = c1.newInstance();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
            System.out.println("main-ClassNotFoundException");
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }
}

```


想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>