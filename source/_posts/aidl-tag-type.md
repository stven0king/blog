---
title: AIDL源码解析in、out和inout
date: 2016-10-27 19:16:43
tags: [IPC,AIDL]
categories: Android
description: "为什么会想写这篇文章，只因为一个error idl.exe E  4928  5836 type_namespace.cpp:130]     'Book' can be an out type, so you must declare it as in, out or inout. 看过上一篇文章Android：IPC之AIDL的学习和总结的同学都知道这是因为在AIDL文件中使用非常规类型作为参数传递的时候没有标记指向tag，那么到底为什么会是这样子的呢，作为一个好奇宝宝我想好好看看。"
---
为什么会想写这篇文章，只因为一个error。"aidl.exe E  4928  5836 type_namespace.cpp:130]     'Book' can be an out type, so you must declare it as in, out or inout. "看过上一篇文章[Android：IPC之AIDL的学习和总结](http://dandanlove.com/2016/10/27/aidl-study/)的同学都知道这是因为在AIDL文件中使用非常规类型作为参数传递的时候没有标记指向tag，那么到底为什么会是这样子的呢，作为一个好奇宝宝我想好好看看。

介绍
-----
[官网介绍AIDL](https://developer.android.com/guide/components/aidl.html)的时候上有这么一段话
>- All non-primitive parameters require a directional tag indicating which way the data goes. Either in, out, or inout (see the example below).
- Primitives are in by default, and cannot be otherwise.
- Caution: You should limit the direction to what is truly needed, because marshalling parameters is expensive.

大概意思是非默认类型的参数都需要添加指向标签in，out或inout。根据自己的需求去添加，因为实现是有代价的。

已知结论
-----
看过我写的[Android：IPC之AIDL的学习和总结](http://dandanlove.com/2016/10/27/aidl-study/)的同学都知道：
> - in表示输入型参数（Server可以获取到Client传递过去的数据，但是不能对Client端的数据进行修改）
- out表示输出型参数（Server获取不到Client传递过去的数据，但是能对Client端的数据进行修改）
- inout表示输入输出型参数（Server可以获取到Client传递过去的数据，但是能对Client端的数据进行修改）。

提出问题
-------
下边我们就研究一个in，out或inout为什么能代表不同的传输方式，为什么实现的代价不一样。

过程验证
------
创建Book.aidl文件
```java
package com.tzx.aidldemo.aidl;
parcelable Book;
```

创建Book.java文件
```java
package com.tzx.aidldemo.aidl;
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

创建aidl接口文件IBookManager.aidl文件
```java
package com.tzx.aidlinout.aidl;
import com.tzx.aidlinout.aidl.Book;
interface IBookManager {
    Book addInBook(in Book book);
    Book addOutBook(out Book book);
    Book addInoutBook(inout Book book);
}
```

创建远程服务
```java
//将bookId都改为-1，在bookName后面都添加参数的tag标记
public class BookManagerService extends Service {
    private CopyOnWriteArrayList list = new CopyOnWriteArrayList();
    private IBinder mBinder = new IBookManager.Stub(){

        @Override
        public Book addInBook(Book book) throws RemoteException {
            book.bookId = -1;
            book.bookName = book.bookName + "-in";
            list.add(book);
            return book;
        }

        @Override
        public Book addOutBook(Book book) throws RemoteException {
            book.bookId = -1;
            book.bookName = book.bookName + "-out";
            list.add(book);
            return book;
        }

        @Override
        public Book addInoutBook(Book book) throws RemoteException {
            book.bookId = -1;
            book.bookName = book.bookName + "-inout";
            list.add(book);
            return book;
        }

        @Override
        public List<Book> getBookList() throws RemoteException {
            return list;
        }
    };
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```
在创建上面的文件的过程中，遇到不太清楚的或者编译出现Error的，可以参考上一篇文章[Android：IPC之AIDL的学习和总结](http://dandanlove.com/2016/10/27/aidl-study/)。

具体方法调用的Activity就不写全部代码了，我们看看三种方法的调用
```java
@Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.book_in:
                try {
                    int bookId = Integer.parseInt(bookIdET.getText().toString());
                    String bookName = bookNameET.getText().toString();
                    if (bookId <= 0 || TextUtils.isEmpty(bookName)) return;
                    StringBuilder builder = new StringBuilder();
                    //LogUtils.d("-----------book_in-----------------");
                    Book book0 = new Book(bookId, bookName);
                    String source = "source:" + book0.toString();
                    //LogUtils.d(source);
                    builder.append(source);
                    builder.append('\n');
                    String result = "result:" + bookManager.addInBook(book0).toString();
                    //LogUtils.d(result);
                    builder.append(result);
                    builder.append('\n');
                    source = "source" + book0.toString();
                    //LogUtils.d(source);
                    builder.append(source);
                    //LogUtils.d("**************book_in****************");
                    bookinfoTV.setText(builder.toString());
                } catch (Exception e) {
                    e.printStackTrace();
                }
                break;
            case R.id.book_out:
                try {
                    int bookId = Integer.parseInt(bookIdET.getText().toString());
                    String bookName = bookNameET.getText().toString();
                    if (bookId <= 0 || TextUtils.isEmpty(bookName)) return;
                    StringBuilder builder = new StringBuilder();
                    //LogUtils.d("-----------book_out-----------------");
                    Book book0 = new Book(bookId, bookName);
                    String source = "source:" + book0.toString();
                    //LogUtils.d(source);
                    builder.append(source);
                    builder.append('\n');
                    String result = "result:" + bookManager.addOutBook(book0).toString();
                    //LogUtils.d(result);
                    builder.append(result);
                    builder.append('\n');
                    source = "source" + book0.toString();
                    //LogUtils.d(source);
                    builder.append(source);
                    //LogUtils.d("**************book_out****************");
                    bookinfoTV.setText(builder.toString());
                } catch (Exception e) {
                    e.printStackTrace();
                }
                break;
            case R.id.book_inout:
                try {
                    int bookId = Integer.parseInt(bookIdET.getText().toString());
                    String bookName = bookNameET.getText().toString();
                    if (bookId <= 0 || TextUtils.isEmpty(bookName)) return;
                    StringBuilder builder = new StringBuilder();
                    //LogUtils.d("-----------book_inout-----------------");
                    Book book0 = new Book(bookId, bookName);
                    String source = "source:" + book0.toString();
                    //LogUtils.d(source);
                    builder.append(source);
                    builder.append('\n');
                    String result = "result:" + bookManager.addInoutBook(book0).toString();
                    //LogUtils.d(result);
                    builder.append(result);
                    builder.append('\n');
                    source = "source" + book0.toString();
                    //LogUtils.d(source);
                    builder.append(source);
                    //LogUtils.d("**************book_inout****************");
                    bookinfoTV.setText(builder.toString());
                } catch (Exception e) {
                    e.printStackTrace();
                }
                break;
        }
    }
```

创建好上面三个文件后，我们编译整个项目工程（PS：生成aidl接口实现类）。

运行结果
![AIDLinout运行结果](https://github.com/stven0king/aidl-study/raw/master/AIDLinout/aidlinout.gif)

下边是与结果相对应的Log输出
```java
14962-14962/com.tzx.aidlinout D/xxx: -----------book_in-----------------
14962-14962/com.tzx.aidlinout D/xxx: source:[bookId=1212,bookName=C++]
14962-14962/com.tzx.aidlinout D/xxx: result:[bookId=-1,bookName=C++-in]
14962-14962/com.tzx.aidlinout D/xxx: source[bookId=1212,bookName=C++]
14962-14962/com.tzx.aidlinout D/xxx: **************book_in****************
14962-14962/com.tzx.aidlinout D/xxx: -----------book_out-----------------
14962-14962/com.tzx.aidlinout D/xxx: source:[bookId=1212,bookName=C++]
14962-14962/com.tzx.aidlinout D/xxx: result:[bookId=-1,bookName=null-out]
14962-14962/com.tzx.aidlinout D/xxx: source[bookId=-1,bookName=null-out]
14962-14962/com.tzx.aidlinout D/xxx: **************book_out****************
14962-14962/com.tzx.aidlinout D/xxx: -----------book_inout-----------------
14962-14962/com.tzx.aidlinout D/xxx: source:[bookId=1212,bookName=C++]
14962-14962/com.tzx.aidlinout D/xxx: result:[bookId=-1,bookName=C++-inout]
14962-14962/com.tzx.aidlinout D/xxx: source[bookId=-1,bookName=C++-inout]
14962-14962/com.tzx.aidlinout D/xxx: **************book_inout****************
```

实际结果与我们已知结论一致~！~！

但问题我们还没有解决，我们继续看代码，其实所有的实现都是在改接口实现类中IBookManager.java

源码解析
------
```java
package com.tzx.aidlinout.aidl;
public interface IBookManager extends android.os.IInterface {
    public com.tzx.aidlinout.aidl.Book addInBook(
        com.tzx.aidlinout.aidl.Book book) throws android.os.RemoteException;

    public com.tzx.aidlinout.aidl.Book addOutBook(
        com.tzx.aidlinout.aidl.Book book) throws android.os.RemoteException;

    public com.tzx.aidlinout.aidl.Book addInoutBook(
        com.tzx.aidlinout.aidl.Book book) throws android.os.RemoteException;

    public java.util.List<com.tzx.aidlinout.aidl.Book> getBookList()
        throws android.os.RemoteException;

    /** Local-side IPC implementation stub class. */
    public static abstract class Stub extends android.os.Binder implements com.tzx.aidlinout.aidl.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.tzx.aidlinout.aidl.IBookManager";
        static final int TRANSACTION_addInBook = (android.os.IBinder.FIRST_CALL_TRANSACTION +
            0);
        static final int TRANSACTION_addOutBook = (android.os.IBinder.FIRST_CALL_TRANSACTION +
            1);
        static final int TRANSACTION_addInoutBook = (android.os.IBinder.FIRST_CALL_TRANSACTION +
            2);
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION +
            3);

        /** Construct the stub at attach it to the interface. */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.tzx.aidlinout.aidl.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.tzx.aidlinout.aidl.IBookManager asInterface(
            android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }

            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);

            if (((iin != null) &&
                    (iin instanceof com.tzx.aidlinout.aidl.IBookManager))) {
                return ((com.tzx.aidlinout.aidl.IBookManager) iin);
            }

            return new com.tzx.aidlinout.aidl.IBookManager.Stub.Proxy(obj);
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

            case TRANSACTION_addInBook: {
                data.enforceInterface(DESCRIPTOR);
				//声明输入的参数_arg0的引用
                com.tzx.aidlinout.aidl.Book _arg0;
				//并根据输入的数据为其创建对象
                if ((0 != data.readInt())) {
                    _arg0 = com.tzx.aidlinout.aidl.Book.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }
				//获取调用this.addInBook方法返回的_result
                com.tzx.aidlinout.aidl.Book _result = this.addInBook(_arg0);
                reply.writeNoException();
				//并向reply中写入返回值_result
                if ((_result != null)) {
                    reply.writeInt(1);
                    _result.writeToParcel(reply,
                        android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                } else {
                    reply.writeInt(0);
                }

                return true;
            }

            case TRANSACTION_addOutBook: {
                data.enforceInterface(DESCRIPTOR);
				//声明输入的参数_arg0的引用
                com.tzx.aidlinout.aidl.Book _arg0;
				//并为其创建新的对象
                _arg0 = new com.tzx.aidlinout.aidl.Book();
				//获取调用this.addOutBook方法返回的_result
                com.tzx.aidlinout.aidl.Book _result = this.addOutBook(_arg0);
                reply.writeNoException();
				//并向reply中写入返回值_result
                if ((_result != null)) {
                    reply.writeInt(1);
                    _result.writeToParcel(reply,
                        android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                } else {
                    reply.writeInt(0);
                }
				//再将参数_arg0写入reply中，至于为什么写入，我们看看客户端Proxy中的读取
                if ((_arg0 != null)) {
                    reply.writeInt(1);
                    _arg0.writeToParcel(reply,
                        android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                } else {
                    reply.writeInt(0);
                }

                return true;
            }

            case TRANSACTION_addInoutBook: {
                data.enforceInterface(DESCRIPTOR);
				//声明输入的参数_arg0的引用
                com.tzx.aidlinout.aidl.Book _arg0;
				//并根据输入的数据为其创建对象
                if ((0 != data.readInt())) {
                    _arg0 = com.tzx.aidlinout.aidl.Book.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }
				//获取调用this.addInoutBook方法返回的_result
                com.tzx.aidlinout.aidl.Book _result = this.addInoutBook(_arg0);
                reply.writeNoException();
				//并向reply中写入返回值_result
                if ((_result != null)) {
                    reply.writeInt(1);
                    _result.writeToParcel(reply,
                        android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                } else {
                    reply.writeInt(0);
                }
				//再将参数_arg0写入reply中，至于为什么写入，我们看看客户端Proxy中的读取
                if ((_arg0 != null)) {
                    reply.writeInt(1);
                    _arg0.writeToParcel(reply,
                        android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                } else {
                    reply.writeInt(0);
                }

                return true;
            }

            case TRANSACTION_getBookList: {
                data.enforceInterface(DESCRIPTOR);

                java.util.List<com.tzx.aidlinout.aidl.Book> _result = this.getBookList();
                reply.writeNoException();
                reply.writeTypedList(_result);

                return true;
            }
            }

            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.tzx.aidlinout.aidl.IBookManager {
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
            public com.tzx.aidlinout.aidl.Book addInBook(
                com.tzx.aidlinout.aidl.Book book)
                throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                com.tzx.aidlinout.aidl.Book _result;

                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
					//将客户端调用时传入的参数写入_data中
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
					//将_data、_reply序列化对象和Stub.TRANSACTION_addInBook指令传递到Server端
                    mRemote.transact(Stub.TRANSACTION_addInBook, _data, _reply,
                        0);
                    _reply.readException();
					//读取Server端返回的序列化_reply中的对象
                    if ((0 != _reply.readInt())) {
                        _result = com.tzx.aidlinout.aidl.Book.CREATOR.createFromParcel(_reply);
                    } else {
                        _result = null;
                    }
					//然后直接将_result返回
					//我们发现整个方法调用期间传入的对象book只是将数据写入到Server，它的值进行并没有任何修改。
					//总结：in类型的参数，它向服务端传入数据，但是却不接受Server返回的值。
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }

                return _result;
            }

            @Override
            public com.tzx.aidlinout.aidl.Book addOutBook(
                com.tzx.aidlinout.aidl.Book book)
                throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                com.tzx.aidlinout.aidl.Book _result;

                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
					//将_data、_reply序列化对象和Stub.TRANSACTION_addInBook指令传递到Server端
					//_data和_reply序列化对象并没有进行写入
                    mRemote.transact(Stub.TRANSACTION_addOutBook, _data,
                        _reply, 0);
                    _reply.readException();
					//读取Server端返回的序列化_reply中的对象，写入到_result
                    if ((0 != _reply.readInt())) {
                        _result = com.tzx.aidlinout.aidl.Book.CREATOR.createFromParcel(_reply);
                    } else {
                        _result = null;
                    }
					//读取Server端返回的序列化_reply中的对象，写入到传入的book对象中
                    if ((0 != _reply.readInt())) {
                        book.readFromParcel(_reply);
                    }
					//然后直接将_result返回
					//我们发现整个方法调用期间传入的对象book并没有将数据写入到Server，它的值确实是Server返回的。
					//总结：out类型的参数，它并不向服务端传入数据，但是却接受Server返回的值。
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }

                return _result;
            }

            @Override
            public com.tzx.aidlinout.aidl.Book addInoutBook(
                com.tzx.aidlinout.aidl.Book book)
                throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                com.tzx.aidlinout.aidl.Book _result;

                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
					//将客户端调用时传入的参数写入_data中
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
					//将_data、_reply序列化对象和Stub.TRANSACTION_addInoutBook指令传递到Server端
                    mRemote.transact(Stub.TRANSACTION_addInoutBook, _data,
                        _reply, 0);
                    _reply.readException();
					//读取Server端返回的序列化_reply中的对象，写入到_result
                    if ((0 != _reply.readInt())) {
                        _result = com.tzx.aidlinout.aidl.Book.CREATOR.createFromParcel(_reply);
                    } else {
                        _result = null;
                    }
					//读取Server端返回的序列化_reply中的对象，写入到传入的book对象中
                    if ((0 != _reply.readInt())) {
                        book.readFromParcel(_reply);
                    }
					//然后直接将_result返回
					//我们发现整个方法调用期间传入的对象book将其数据写入到Server，并且它的值被Server返回的数据修改。
					//总结：inout类型的参数，它既向服务端传入数据，也却接受Server返回的值。
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }

                return _result;
            }

            @Override
            public java.util.List<com.tzx.aidlinout.aidl.Book> getBookList()
                throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.tzx.aidlinout.aidl.Book> _result;

                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data,
                        _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.tzx.aidlinout.aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }

                return _result;
            }
        }
    }
}
```

看了这么多代码是不是感觉脑袋大了，没事接下来一张图帮你理的清清楚楚的：
![aidl-tag-type](https://raw.githubusercontent.com/stven0king/aidl-study/master/AIDLinout/aidl-tag-type.jpg)

经过两篇文章对aidl的讲解，我想你已经把它理解的透透的了，如果还有什么问题可以给我留言哦~！

[GitHubDemo地址](https://github.com/stven0king/aidl-study)

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>