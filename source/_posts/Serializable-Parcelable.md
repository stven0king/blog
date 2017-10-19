---
title: Serializable和Parcelable的再次回忆
date: 2016-10-18 19:18:43
tags: [序列化]
categories: Android
description: "序列化和反序列化几乎是工程师们每天都要面对的事情，但是要精确掌握这两个概念并不容易：一方面，它们往往作为框架的一部分出现而湮没在框架之中；另一方面，它们会以其他更容易理解的概念出现，例如加密、持久化。然而，序列化和反序列化的选型却是系统设计或重构一个重要的环节，在分布式、大数据量系统设计里面更为显著。恰当的序列化协议不仅可以提高系统的通用性、强健性、安全性、优化系统性能，而且会让系统更加易于调试、便于扩展。总而言之搞懂序列化是很重要滴。"
---
自己开发Android也有些时间了，Serializable和Parcelable遇到过不止一次了。但是每次别人问起具体的内容自己偏偏记得不是很清晰。因为某些原因再次梳理一下，以文章的形式给自己存储下来。温故而知新~！~！

序列化和反序列化几乎是工程师们每天都要面对的事情，但是要精确掌握这两个概念并不容易：一方面，它们往往作为框架的一部分出现而湮没在框架之中；另一方面，它们会以其他更容易理解的概念出现，例如加密、持久化。然而，序列化和反序列化的选型却是系统设计或重构一个重要的环节，在分布式、大数据量系统设计里面更为显著。恰当的序列化协议不仅可以提高系统的通用性、强健性、安全性、优化系统性能，而且会让系统更加易于调试、便于扩展。总而言之搞懂序列化是很重要滴。

序列化
-----
> 将对象的状态信息转换为可以存储或传输的形式的过程。在序列化期间，对象将其当前状态写入到临时或持久性存储区。

反序列化
------
> 就是读取序列化后保存在存储区的序列化信息或反序列化对象的状态，重新创建该对象。

序列化和反序列化通常是并存的，他们的关系就像进行加密和解密操作一样。前者需要相同的序列化方式，后者需要知道秘钥。

Android中将对象序列化的方式有两种Serializable和Parcelable这两个接口都可以完成。Serializable是Java自带的序列化方法，而Android原生的序列化为Parcelable。这并不意味着在Android中可以抛弃Serialable，只能说在Android中Parcelable方法实现序列化更有优势。下边我们可以具体来看看这两个接口实现。（PS：对象参加序列化的时候其类的结构是不能发生变化的，以下Demo都是在此基础上演示）

Serializable
------
我们先看一个实现Serializable接口的类：
```java
public class User implements Serializable {
    private static final long serialVersionUID = -4454266436543306544L;
    public int userId;
    public String userName;
    public User(int userId, String userName) {
        this.userId = userId;
        this.userName = userName;
    }
}
```
Serializable是java提供的一个序列化接口，它是一个空接口，专门为对象提供标准的序列化和反序列化操作，使用Serializable来实现序列化相当简单，只需要在类的声明中指定一个类似下面的标示即可自动实现默认的序列化过程（serialVersionUID可自动生成也可自己写，如0L,1L,2L...）。
```java
private static final long serialVersionUID = -4454266436543306544L;
```
通过Serializable方式来实现对象的序列化，实现起来非常简单，几乎素有工作都被系统自动完成。如果进行对象的序列化和反序列化也非常简单，只需要采用ObjectOutputStream和ObjectInputStream即可轻松实现。下面举个简单的列子：
```java
//序列化过程
User user = new User(1, "Tom");
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("cache.txt"));
out.writeObject(user);
out.close();
//反序列化过程
ObjectInputStream in = new ObjectInputStream(new FileInputStream("cache.txt"));
User newuser = (User) in.readObject();
in.close();
```
上述代码采用Serializable方式序列化对象的典型过程，只需要把实现Serializable接口的User对象写到文件中就可以快速恢复了，恢复后的对象newUser和User的内容完全一致，但是两个并不是同一个对象。

serialVersionUID
-------
上面提到，想让一个对象实现序列化，只需要这个类实现Serializable接口并声明一个serialVersionUID即可，实际上，甚至这个serialVersionUID也不是必需的，我们不声明这个serialVersionUID同样可以实现序列化，但是这将会对反序列化过程产生影响，具体会产生什么影响呢？

解决这个问题前我们先提一个问题，为什么需要serialVersionUID呢？
> 因为静态成员变量属于类不属于对象，不会参与序列化过程，使用transient关键字标记的成员变量也不参与序列化过程。 （PS：关键字transient，这里简单说明一下，Java的serialization提供了一种持久化对象实例的机制。当持久化对象时，可能有一个特殊的对象数据成员，我们不想用serialization机制来保存它。为了在一个特定对象的一个域上关闭serialization，可以在这个域前加上关键字transient。当一个对象被序列化的时候，transient型变量的值不包括在序列化的表示中，然而非transient型的变量是被包括进去的）

这个时候又有一个疑问serialVersionUID是静态成员变量不参与序列化过程，那么它的存在与否有什么影响呢？
> 具体过程是这样的：序列化操作的时候系统会把当前类的serialVersionUID写入到序列化文件中，当反序列化时系统会去检测文件中的serialVersionUID，判断它是否与当前类的serialVersionUID一致，如果一致就说明序列化类的版本与当前类版本是一样的，可以反序列化成功，否则失败。

![InvalidClassException](http://img.blog.csdn.net/20161018095926141)

接下来我们回答声明serialVersionUID对进行序列化有啥影响？
> 如果不手动指定serialVersionUID的值，反序列化时当前类有所改变，比如增加或者删除了某些成员变量，那么系统就会重新计算当前类的hash值并且把它赋值给serialVersionUID，这个时候当前类的serialVersionUID就和序列化的数据中的serialVersionUID不一致，于是反序列化失败。所以我们手动指定serialVersionUID的值能很大程度上避免了反序列化失败。

以上就是自己对Serializable的认识，下边来看看Parcelable相关的知识~！~！

Parcelable
-------
我们先看一个使用Parcelable进行序列化的例子：
```java
public class Book implements Parcelable {
    public int bookId;
    public String bookName;

    public Book() {
    }

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }
    //从序列化后的对象中创建原始对象
    protected Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        //从序列化后的对象中创建原始对象
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }
        //指定长度的原始对象数组
        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };
    //返回当前对象的内容描述。如果含有文件描述符，返回1，否则返回0，几乎所有情况都返回0
    @Override
    public int describeContents() {
        return 0;
    }
    //将当前对象写入序列化结构中，其flags标识有两种（1|0）。
    //为1时标识当前对象需要作为返回值返回，不能立即释放资源，几乎所有情况下都为0.
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }

    @Override
    public String toString() {
        return "[bookId=" + bookId + ",bookName='" + bookName + "']";
    }
}
```

虽然Serializable可以将数据持久化在磁盘，但其在内存序列化上开销比较大（PS：Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC），而内存资源属于android系统中的稀有资源（android系统分配给每个应用的内存开销都是有限的），为此android中提供了Parcelable接口来实现序列化操作，在使用内存的时候，Parcelable比Serializable性能高，所以推荐使用Parcelable。

Parcelable内部包装了可序列化的数据，可以在Biander中自由的传输，从代码中可以看出，在序列化过程中需要实现的功能有序列化，反序列化和内容描述。序列化功能是由writetoParcel方法来完成，最终是通过Parcel中的一系列write方法来完成的。反序列化功能是由CREATOR方法来完成，其内部标明了如何创建序列化对象和数组，并通过Parcel的一系列read方法来完成反序列化过程（PS：write和read的顺序必须一致~！）；内容描述功能是有describeContents方法来完成，几乎所有情况下这个方法都应该返回0，仅当当前对象中存在文件描述符时，此方法返回1.

系统已经给我们提供了许多实现了Parcelable接口类，他们都是可以直接序列化的，比如Intent，Bundle，Bitmap等，同事List和Map也支持序列化，提前是他们里面的每个元素都是可以序列化的。


<font color=#0099ff face="微软雅黑">总结</font>
> 既然Parcelable和Serializable都能实现序列化并且都可以用于Intent间传递数据，那么二者改如果选择呢？Serializable是Java中的序列化接口，其使用起来简单但是开销很大，序列化和反序列化过程需要大量的I/O操作。而Parcelable是Android中序列化方法，因为更适合于在Android平台上，它的缺点就是使用起来比较麻烦，但是它的效率很高，这是Android推荐的序列化方法，因为我们要首选Parcelable。Parcelable主要用于内存序列化上，通过Parcelable将对象序列化到存储设备中或者将对象序列化后通过网络传输也都是可以的，但是这个过程稍显复杂，因此在这两种情况下建议使用Serializable。

文章部分摘自任玉刚的《Android开发艺术探索》

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>