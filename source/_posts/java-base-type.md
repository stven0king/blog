---
title: JavaSE的自动装箱和自动拆箱
date: 2017-5-30 17:34:00
tags: [拆箱&装箱]
categories: JavaSE 
description: "JavaSE的自动装箱和自动拆箱有需要回顾的么，温故而知新哦~！~！"
---
回顾一下Java基础数据类型：
-----
![Java数据类型](http://upload-images.jianshu.io/upload_images/1319879-1b843fb53d98cace.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

|基础类型|字节|包装类型|
| -------- | -------- | -------- |
|int|4字节|Integer|
|byte|1字节|Byte|
|short|2字节|Short|
|long|8字节|Long|
|float|4字节|Float|
|double|8字节|Double|
|char|2字节|Character|
|boolean|未定|Boolean|

> Java属于面向对象语言那么为什么会出现非对象类型数据（基础类型），因为基础数据类型是的虚拟机的运行速度更快而且占用内存更少。详情内容可以参见：[Java为什么需要保留基本数据类型](http://www.importnew.com/11915.html)

为什么要有装箱&拆箱
----
在JavaSE5之前我们创建爱你Integer对象：
```java
Integer i = new Integer(10);
```
从JavaSE5提供了自动装箱的特性时，我们可以更简单的创建基础类型的对象：
```java
Integer a = 10;
int b = a;
```

> 从上面的代码我们可以简单的看出装箱、拆箱的操作：
Integer a = 10;//我们将10【装箱】生成Integer对象。
int b = a;//我们将Integer【拆箱】转成int基础类型

装箱和拆箱是如何实现的
-----
我们这里先写一个简单的类，然后反编译看看它的字节码文件
```java
public class Main {
    public static void main(String[] args) {
        Integer a = 10;
        int b = a;
    }
}
```
反编译出来的字节码文件：

![Main字节码.jpg](http://upload-images.jianshu.io/upload_images/1319879-dc57d46990d3f88e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

结论：
----
装箱操作：
===
```java
Integer a = 10;
//实际执行的是Integer a = Integer.valueOf(10);
```
拆箱操作：
===
```java
int b = a;
//实际执行的是int b = a.intValue();
```
其他&扩展
---
我们先来看一道面试题：

```java
public class Main {
    public static void main(String[] args) {
        Integer a = 10;
        Integer b = 10;
        Integer c = 128;
        Integer d = 128;
        System.out.println(a == b);
        System.out.println(c == d);
    }
}
```
内心怀揣自己的真是答案，我们看看下边的源代码：
先看看Integer装箱和拆箱的函数源码：
```java
/**
 * Returns the value of this {@code Integer} as an
 * {@code int}.
 */
public int intValue() {
	return value;
}

/**
 * Returns an {@code Integer} instance representing the specified
 * {@code int} value.  If a new {@code Integer} instance is not
 * required, this method should generally be used in preference to
 * the constructor {@link #Integer(int)}, as this method is likely
 * to yield significantly better space and time performance by
 * caching frequently requested values.
 *
 * This method will always cache values in the range -128 to 127,
 * inclusive, and may cache other values outside of this range.
 *
 * @param  i an {@code int} value.
 * @return an {@code Integer} instance representing {@code i}.
 * @since  1.5
 */
public static Integer valueOf(int i) {
	if (i >= IntegerCache.low && i <= IntegerCache.high)
		return IntegerCache.cache[i + (-IntegerCache.low)];
	return new Integer(i);
}
```

> - 拆箱操作：直接返回Integer内的数值
- 装箱操作：在i大于IntegerCache.low或者i小于IntegerCache.high时返回缓存的Integer对象，否则创建新的Integer对象。


```java
/**
 * Cache to support the object identity semantics of autoboxing for values between
 * -128 and 127 (inclusive) as required by JLS.
 *
 * The cache is initialized on first usage.  The size of the cache
 * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.
 * During VM initialization, java.lang.Integer.IntegerCache.high property
 * may be set and saved in the private system properties in the
 * sun.misc.VM class.
 */
private static class IntegerCache {
	/*********/
}
```
通过源码码我们可以发现，Integer在数据为[-128,127]之间时。使用了IntegerCache 返回缓存中对象的引用，否则new一个新的对象。

看到上面这个答案，有些同学就会想到：除过Integer之前还有其他的基础数据类型，那么其他的类型是否也是专业那个的呢？答案：是也不是。原理想想大家也都明白：
> - Boolean内部有true&false两个静态变量，最后装箱得到的值都是这两个静态变量的引用。
- Long&Integer&Short&Byte在数值为[-128,127]之间都有Cache。
- Double&Float则都没有。

所以上面问题的正确答案分别是:true、false。

见识了"=="比较，现在看equals比较结果：
同样我们也先看一道题目：
```java
public class Main {
    public static void main(String[] args) {
        Integer a = 1;
        Integer b = 2;
        Long c = 3L;
        System.out.println(c == (a + b));
        System.out.println(c.equals(a + b));
    }
}
```
然后我们再看看源码：
```java
/**
 * Compares this object to the specified object.  The result is
 * {@code true} if and only if the argument is not
 * {@code null} and is an {@code Integer} object that
 * contains the same {@code int} value as this object.
 *
 * @param   obj   the object to compare with.
 * @return  {@code true} if the objects are the same;
 *          {@code false} otherwise.
 */
public boolean equals(Object obj) {
	if (obj instanceof Integer) {
		return value == ((Integer)obj).intValue();
	}
	return false;
}
```
在Java中我们知道操作"=="的两个数都是数据包装类型对象的引用的话，那么则是用来比较两个引用所指向的对象是不是同一个；而如果其中有一个操作数是表达式（即包含算术运算）则比较的是数值（即会触发自动拆箱的过程）。*为什么呢，因为"=="两边引用数据类型必须一致，要不然无语错误。*

所以我们得到上边题目的答案是：true、false。
因为第一次比较实际是先对数据进行拆箱然后比较，所以得到的结果是true；第二次比较实际是先拆箱（两个Integer对象拆箱）后装箱（将拆箱且计算后的数据再装箱），然后同Long对象比较，显然不是同一类型所以得到false。

以上问题及答案都是作者亲自敲出来的，想实际操作同学也可以反编译class文件看看【真相】’。

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>