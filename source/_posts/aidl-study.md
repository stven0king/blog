---
title: Android：IPC之AIDL的学习和总结
date: 2016-10-27 19:14:00
tags: [IPC,AIDL]
categories: Android
description: "为了使得一个程序能够在同一时间里处理许多用户的要求。即使用户可能发出一个要求，也肯能导致一个操作系统中多个进程的运行（PS:听音乐，看地图）。而且多个进程间需要相互交换、传递信息，IPC方法提供了这种可能。IPC方法包括管道（PIPE）、消息排队、旗语、共用内存以及套接字（Socket）。"
---
为了使得一个程序能够在同一时间里处理许多用户的要求。即使用户可能发出一个要求，也肯能导致一个操作系统中多个进程的运行（PS:听音乐，看地图）。而且多个进程间需要相互交换、传递信息，IPC方法提供了这种可能。IPC方法包括管道（PIPE）、消息排队、旗语、共用内存以及套接字（Socket）。

Android中的IPC方式有Bundle、文件共享、Messager、AIDL、ContentProvider和Socket。
这次我们学习的是Android中的AIDL。
概述
===========
AIDL（Android接口描述语言）是一个IDL语言，它可以生成一段代码，可以是一个在Android设备上运行的两个进程使用内部通信进程进行交互。在Android上，一个进程通常无法访问另一个进程的内存。所以说，如果你想在一个进程中（例如在一个Activity中）访问另一个进程中（例如service）某个对象的方法，你就可以使用AIDL来生成这样的代码来伪装传递各种参数。

语法
========
AIDL它和Java基本上类似，只是有一些细微的差别（PS：可能Google为了方便Android程序猿使用）。AIDL使用简单的语法来声明接口，描述其方法以及方法的参数和返回值。这些参数和返回值可以是任何类型，甚至是其他AIDL生成的接口。重要的是必须导入所有非内置类型，哪怕是这些类型是在与接口相同的包中。

下边说说AIDL的一些特点：
> - 通常引引用方式传递的其他AIDL生成的接口，必须要import 语句声明。
- Java编程语言的主要类型 (int, boolean等) —不需要 import 语句。
- 在AIDL文件中，并不是所有的数据类型都是可以使用的，那么到底AIDL文件中支持哪些数据类型呢？
如下所示：
1、基本数据类型（int,long,char,boolean,float,double,byte,short八种基本类型）;
2、String和CharSequence;
3、List:只支持ArrayList,里面每个元素都必须能够被AIDL支持；
4、Map:只支持HashMap，里面的每个元素都必须被AIDL支持，包括key和value;
5、Parcelable:所有实现了Parcelable接口的对象；
6、AIDL：所有的AIDL接口本身也可以在AIDL文件中使用；
以上6中数据类型就是AIDL所支持的所有类型，其中自定义的Parcelable对象和AIDL对象必须要显式import进来，不管它们是否和当前的AIDL文件位于同一个包内。

-------------------
> 需要注意的地方：
AIDL中除了基本数据类型，其他类型的参数必须标上方向：in、out或者inout；
(PS:假若传递一个Book对象且没有加指向tag时，则会抛出"aidl.exe E  4928  5836 type_namespace.cpp:130]     'Book' can be an out type, so you must declare it as in, out or inout."异常)
- in表示输入型参数（Server可以获取到Client传递过去的数据，但是不能对Client端的数据进行修改）
- out表示输出型参数（Server获取不到Client传递过去的数据，但是能对Client端的数据进行修改）
- inout表示输入输出型参数（Server可以获取到Client传递过去的数据，但是能对Client端的数据进行修改）。
更多tag相关的内容：[AIDL源码解析in、out和inout](http://dandanlove.com/2016/10/27/aidl-tag-type/)

使用AIDL实现IPC
=======
实现步骤 （[官网AIDL样例](https://developer.android.com/guide/components/aidl.html)）
-------
```java
// IRemoteService.aidl
package com.example.android;

// Declare any non-default types here with import statements

/** Example service interface */
interface IRemoteService {
    /** Request the process ID of this service, to do evil things with it. */
    int getPid();

    /** Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```
这是官网创建的一个简单的AIDL文件。

这次我们自己声明一个包含非默认支持类型的AIDL文件。
AIDL要跨进程通信，其所携带的数据也需要跨进程传输。所以我们首先需要自定自己想要传输的数据类必须其必须实现Parcelable接口从而可以被序列化。
为什么需要序列化呢，为什么不适用Serializable，不知道的同学可以看下这篇文章：[ Serializable和Parcelable的再次回忆](http://dandanlove.com/2016/10/18/Serializable-Parcelable/)
所以我们先创建需要传输的数据所对应的aidl文件，然后再相同目录下创建对应的Java类文件。这里可能有些同学会疑惑，不是直接创建Java类么。AIDL文件有两种类型，一种是我们上边定义的接口，而另外一种就是非常规类型的数据对象文件。即：如果AIDL文件中用到了自定义的Parcelable对象，那么必须新建一个和它同名的AIDL文件，并在其中声明它为Parcelable类型。详细的使用我们看下边例子：

创建一个Book.aidl文件
------
在Android Studio的项目中先创建对应的aidl包，然后右击选择创建aidl文件，so easy。
```java
// Book.aidl
package com.tzx.aidldemo.aidl;

// Declare any non-default types here with import statements
//所有注释掉的内容都是Android Studio帮你写的，但是我们不需要。
//我们创建的是aidl数据对象，所以我们只需写出parcelable 后面跟对象名。
//parcelabe前的字母‘p’是小写的哦~
parcelable Book;
//interface Book {
//
//    /**
//     * Demonstrates some basic types that you can use as parameters
//     * and return values in AIDL.
//     */
//    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
//            double aDouble, String aString);
//}

```

在Book.aidl的包下创建Book.java类文件
-------
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

> 在Android Studio中如果先创建Java类文件，然后创建AIDL文件则会提示命名重复，但顺序反过来就可以。

创建aidl接口文件IBookManager.aidl
-------
```java
// IBookManager.aidl
package com.tzx.aidldemo.aidl;
//通常引用方式传递自定义对象，必须要import语句声明
import com.tzx.aidldemo.aidl.Book;
interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

这样所有aidl相关的文件就定义完了，我们可以写客户端和服务端了么。然而实际结果表明我们还是无法在客户端或服务费使用aidl类。在这里说一下其实aidl方式只不过是为我们提供模板自动创建aidl对应的Java类文件，只有生成了对应的Java文件之后我们才可以在客户端或服务端使用。Android studio中make一下当前的project就会在项目的app/build/source/aidl/包名/debug这个目录下生成对应的aidl类文件（PS：只有aidl接口文件才会生成java类文件）。

make的时候可能提示找不到对应的Book.java文件，我们可以在build.gradle文件中的android{}标签里面添加：
```java
sourceSets{
	main{
		aidl.srcDirs = ['src/main/java']
	}
}
```
这种情况只适合aidl类文件和对应的java类文件在同一个包下。

好了，现在所有的aidl文件都有了，我们开始写我们的服务交互了~！~！

服务端：
-----
Service服务
```java
/**
 * Created by tanzhenxing
 * Date: 2016/10/17.
 * Description:远程服务
 */
public class BookManagerService extends Service {
    //支持并发读写
    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();
	//服务端定义Binder类（IBookManager.Stub）
    private Binder mBinder = new IBookManager.Stub() {

        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```
并在Manifest文件中声明，将它放在一个新的进程中，这样方便我们演示跨进程通信。
```
<service android:name=".BookManagerService"
	android:process=":server"/>
```

客户端：
-----
```java
/**
 * Created by tanzhenxing
 * Date: 2016/10/17.
 * Description:主界面
 */
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    private EditText bookNameTV;
    private Button bookAddTV;
    private Button bookCountTV;
    private TextView bookInfoTV;
    private Intent bookManagerIntent;
    private boolean mBound = false;
    private IBookManager bookManager;
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
	        //客户端获取代理对象
            bookManager = IBookManager.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onStart() {
        super.onStart();
        bookManagerIntent = new Intent(this, BookManagerService.class);
        bindService(bookManagerIntent, mConnection, Context.BIND_AUTO_CREATE);
        mBound = true;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initView();
        initListener();

    }


    private void initView() {
        bookNameTV = (EditText) findViewById(R.id.book_name);
        bookAddTV = (Button) findViewById(R.id.book_add);
        bookCountTV = (Button) findViewById(R.id.book_count);
        bookInfoTV = (TextView) findViewById(R.id.book_info);
    }

    private void initListener() {
        bookAddTV.setOnClickListener(this);
        bookCountTV.setOnClickListener(this);
    }


    @Override
    protected void onStop() {
        super.onStop();
        if (mBound) {
            mBound = false;
            unbindService(mConnection);
        }
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.book_add:
                addBook();
                break;
            case R.id.book_count:
                getBookList();
                break;
        }
    }

    private void addBook() {
        if (bookManager != null && !TextUtils.isEmpty(bookNameTV.getText().toString())) {
            Book book = new Book((int) System.currentTimeMillis(), bookNameTV.getText().toString());
            try {
                bookManager.addBook(book);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }

    public void getBookList() {
        try {
            if (bookManager != null) {
                List<Book> list = bookManager.getBookList();
                if (list != null && list.size() > 0) {
                    StringBuilder builder = new StringBuilder();
                    for (Book book : list) {
                        builder.append(book.toString());
                        builder.append('\n');
                    }
                    bookInfoTV.setText(builder.toString());
                } else {
                    bookInfoTV.setText("Empty~!");
                }
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}
```

运行结果
-----
![AIDLDEMO](https://raw.githubusercontent.com/stven0king/aidl-study/master/AIDLDemo/aidldemo.gif)

解析aidl生成的java类
----
```java
public interface IBookManager extends android.os.IInterface {
	//根据aidl文件中定义的方法，进行接口声明
    public java.util.List<com.tzx.aidldemo.aidl.Book> getBookList()
        throws android.os.RemoteException;
	//根据aidl文件中定义的方法，进行接口声明
    public void addBook(com.tzx.aidldemo.aidl.Book book)
        throws android.os.RemoteException;

    /** Local-side IPC implementation stub class. */
    public static abstract class Stub extends android.os.Binder implements com.tzx.aidldemo.aidl.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.tzx.aidldemo.aidl.IBookManager";
		//定义方法执行code，与客户端同步
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION +
            0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION +
            1);

        /** Construct the stub at attach it to the interface. */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.tzx.aidldemo.aidl.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.tzx.aidldemo.aidl.IBookManager asInterface(
            android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }

            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);

            if (((iin != null) &&
                    (iin instanceof com.tzx.aidldemo.aidl.IBookManager))) {
                return ((com.tzx.aidldemo.aidl.IBookManager) iin);
            }
			//生成代理对象
            return new com.tzx.aidldemo.aidl.IBookManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data,
            android.os.Parcel reply, int flags)
            throws android.os.RemoteException {
            switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(DESCRIPTOR);

                return true;
            }

            case TRANSACTION_getBookList: {
                data.enforceInterface(DESCRIPTOR);
				//调用服务端getBookList()
                java.util.List<com.tzx.aidldemo.aidl.Book> _result = this.getBookList();
                reply.writeNoException();
                reply.writeTypedList(_result);

                return true;
            }

            case TRANSACTION_addBook: {
                data.enforceInterface(DESCRIPTOR);

                com.tzx.aidldemo.aidl.Book _arg0;

                if ((0 != data.readInt())) {
                    _arg0 = com.tzx.aidldemo.aidl.Book.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }
				//调用服务端addBook()
                this.addBook(_arg0);
                reply.writeNoException();

                return true;
            }
            }

            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.tzx.aidldemo.aidl.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public java.util.List<com.tzx.aidldemo.aidl.Book> getBookList()
                throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.tzx.aidldemo.aidl.Book> _result;

                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
					//调用远程服务addBook()
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data,
                        _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.tzx.aidldemo.aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }

                return _result;
            }

            @Override
            public void addBook(com.tzx.aidldemo.aidl.Book book)
                throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();

                try {
                    _data.writeInterfaceToken(DESCRIPTOR);

                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
					//调用远程服务addBook
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }
    }
}
```

用关系图表示比较清楚些。。

![IBookManager.java](https://raw.githubusercontent.com/stven0king/aidl-study/master/AIDLDemo/IBookManager.jpg)

每个文件结构我们都解析完了，那么aidl到底是怎么实现通信的呢，要让我们自己写一套类似于aidl的那么应该怎么去设计呢？

我们仿aidl画一幅结构图：
![AIDL](https://raw.githubusercontent.com/stven0king/aidl-study/master/AIDLDemo/ipc-aidl.jpg)

根据上面这个图，我们就可以写出自己的aidl。
```java
//方法接口
public interface IBookManager {
    final int CAHAGE_MSG = 1;
    Book change(Book book);
}
//实现方法的代理类
public class Proxy implements IBookManager{
    private static IBinder mRemote;
    public static Proxy asInterface(IBinder service) {
        mRemote = service;
        return new Proxy();
    }

    public Book change(Book book) {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        try {
            data.writeInt(1);
            book.writeToParcel(data, 0);
            mRemote.transact(IBookManager.CAHAGE_MSG, data, reply, 0);
            reply.readException();
            if(0 != reply.readInt()) {
                return Book.CREATOR.createFromParcel(reply);
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }finally {
            data.recycle();
            reply.recycle();
        }
        return null;
    }
}
//Binder远端实现类
public class Stub extends Binder implements IBookManager {
    @Override
    public Book change(Book book) {
        book.bookName = "Server";
        return book;
    }

    @Override
    protected boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        switch (code) {
            case IBookManager.CAHAGE_MSG:
                Book book = null;
                if (0 != data.readInt()) {
                    book = Book.CREATOR.createFromParcel(data);
                }
                book = change(book);
                reply.writeNoException();
                reply.writeInt(1);
                book.writeToParcel(reply, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                reply.writeParcelable(book, 0);
                return true;
        }
        return super.onTransact(code, data, reply, flags);
    }
}
```
看完以上针对aidl的文字和图片结合方式讲解我想你应该能熟练的开发aidl了吧，如果还有问题可以给我留言哦~！

[GitHubDemo地址](https://github.com/stven0king/aidl-study)

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>