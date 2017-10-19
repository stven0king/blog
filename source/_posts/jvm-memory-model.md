---
title: JVM内存模型
date: 2017-8-26 15:40:00
tags: [JVM]
categories: Java 
description: "前一段时间写过一篇关于【JVM虚拟机之类加载的过程】的文章，其中讲述了Java虚拟机对类的处理。最近听了一次部门内部有关JVM的分享，自己也顺便回顾了之前阅读《深入理解JVM虚拟机》一书中所讲述的Java虚拟机对内存的管理，再次将自己理解的JVM内存模型分享给大家。"
---
前一段时间写过一篇关于 [JVM虚拟机之类加载的过程](http://dandanlove.com/2016/09/20/java-jvm-class-loading/) 的文章，其中讲述了Java虚拟机对类的处理。最近听了一次部门内部有关JVM的分享，自己也顺便回顾了之前阅读《深入理解JVM虚拟机》一书中所讲述的Java虚拟机对内存的管理，再次将自己理解的JVM内存模型分享给大家。
 
Java运行时数据区域
---
Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区。这些区域都有各自的用途，一级创建和销毁的时间，有的区域随着虚拟机进程的启动而从在，有些区域则依赖用户线程的启动和结束而简历和销毁。

<center>![JVM内存模型](http://upload-images.jianshu.io/upload_images/1319879-46f35743dbab107d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

>- 程序计数器
- Java虚拟机栈
- 本地方法栈
- Java堆
- 方法区
- 运行时常量池
- 直接内存

<center>![java-memory-model](http://upload-images.jianshu.io/upload_images/1319879-79c0858a44380293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

程序计数器
---
当前线程所执行的字节码文件的行号指示器。
>- 每个线程都有一个程序计数器
- 不会发生OOM

Java虚拟机栈
---
虚拟机栈描述的是Java方法执行的内存模型：每个方法在执行时都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。 

>- 线程独有，生命周期与线程保持一致
- StackOverflowError   （单线程，栈帧太大，还是虚拟机栈容量太小）
- OutOfMemoryError  （创建线程太多）

本地方法栈
---
同Java虚拟机栈类型，只不过是native方法执行的内存模型。

Java堆
---
所有的对象实例以及数组都要在堆上分配（除了Class对象）

>- 所有线程共享的一块内存区域
- OutOfMemoryError

方法区
---
用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。 
>- 各个线程共享的内存区域
- 如果一个类被加载了，就会在方法区生成一个代表该类的Class对象，有了该对象的存在，才有了反射的实现。 
- 会出OOM

运行时常量池
---
运行时常量池中主要存放两大类常量：字面量和符号引用。字面量比较接近于Java语言层面的常量概念，如文本字符串、声明为final的常量值等。 

而符号引用则属于编译原理方面的概念，包括了下面三类常量：
>- 类和接口的全限定名（包名+类名）
- 字段的名称和描述符 
- 方法的名称和描述符

时常量池在JDK1.6及之前版本的JVM中是方法区的一部分，而在JDK1.7及之后版本的JVM已经将运行时常量池从方法区中移了出来，在Java 堆（Heap）中开辟了一块区域存放运行时常量池。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>