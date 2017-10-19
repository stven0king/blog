---
title: 深入解析AIDL的实现：Messenger
date: 2017-6-29 15:54:00
tags: [Android消息机制,Messenger,AIDL]
categories: Android 
description: "Messenger可以翻译为信使，顾名思义，通过它可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，它是AIDL在Android中的一种经典实践。。"
---
Messenger可以翻译为信使，顾名思义，通过它可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，它是AIDL在Android中的一种经典实践。

【本篇文章中讲述的都是跨进程通信，在相同进程中使用Messenger文章不做讲述~！~！】

文章主要讲述Messenger利用AIDL进行进程间通信，其中不免会涉及到AIDL的知识点。对ADIL不熟悉的同学可以阅读我之前写过的一篇[Android：IPC之AIDL的学习和总结](http://www.jianshu.com/p/419cc7b95358)。

本来想先通过Demo来引出Messenger，然后再根据Demo一步一步分析源码。但是最后还是觉得本次应该先讲述Messenger的基础知识，结合aidl的知识分析源码，然后再通过讲述Demo深入一些分析Android当时设计Messenger的心情。
先来看看Messenger的主要成员变量和成员方法：
```java
public final class Messenger implements Parcelable {
    private final IMessenger mTarget;
    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
    public void send(Message message) throws RemoteException {
        mTarget.send(message);
    }
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
	
	/**其他代码省略**/
	//重点mTarget为IMessenger.Stub.asInterface
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
}
```
Messenger一共有两个构造函数，一个通过IBinder构造，一个是通过Handler构造。
两种实现方式意义相同么（这是一个非常重要的问题）？
```java
//Handler#getIMessenger
final IMessenger getIMessenger() {
	synchronized (mQueue) {
		if (mMessenger != null) {
			return mMessenger;
		}
		mMessenger = new MessengerImpl();
		return mMessenger;
	}
}
//Handler#MessengerImpl
//重点MessengerImp是继承IMessenger.Stub
private final class MessengerImpl extends IMessenger.Stub {
	public void send(Message msg) {
		msg.sendingUid = Binder.getCallingUid();
		Handler.this.sendMessage(msg);
	}
}
```
IMessenger是一个aidl接口,MessengerImpl为它的实现类
```java
package android.os;

import android.os.Message;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
```
之前介绍的[Android：IPC之AIDL的学习和总结](http://www.jianshu.com/p/419cc7b95358)文章中对aidl接口在编译后生成的Java类文件做了详细的分析和讲解。
这次我们再看一遍生成的IMessenger.java文件，对此熟悉的同学可以略过了。
```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: frameworks/base/core/java/android/os/IMessenger.aidl
 */
package android.os;

/** @hide */
public interface IMessenger extends android.os.IInterface {
    public void send(android.os.Message msg) throws android.os.RemoteException;

    /** Local-side IPC implementation stub class. */
    public static abstract class Stub extends android.os.Binder implements android.os.IMessenger {
        private static final java.lang.String DESCRIPTOR = "android.os.IMessenger";
        static final int TRANSACTION_send = (android.os.IBinder.FIRST_CALL_TRANSACTION +
            0);

        /** Construct the stub at attach it to the interface. */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an android.os.IMessenger interface,
         * generating a proxy if needed.
         */
        public static android.os.IMessenger asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }

            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);

            if (((iin != null) && (iin instanceof android.os.IMessenger))) {
                return ((android.os.IMessenger) iin);
            }

            return new android.os.IMessenger.Stub.Proxy(obj);
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

            case TRANSACTION_send: {
                data.enforceInterface(DESCRIPTOR);

                android.os.Message _arg0;

                if ((0 != data.readInt())) {
                    _arg0 = android.os.Message.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }

                this.send(_arg0);

                return true;
            }
            }

            return super.onTransact(code, data, reply, flags);
        }
		//远端代理类
        private static class Proxy implements android.os.IMessenger {
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
            public void send(android.os.Message msg)
                throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();

                try {
                    _data.writeInterfaceToken(DESCRIPTOR);

                    if ((msg != null)) {
                        _data.writeInt(1);
                        msg.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }

                    mRemote.transact(Stub.TRANSACTION_send, _data, null,
                        android.os.IBinder.FLAG_ONEWAY);
                } finally {
                    _data.recycle();
                }
            }
        }
    }
}

```
看完这个IMessenger.java这个类，我们再回一下刚才Messenger的两个构造方法。

使用Handler构造：
```java
//Messenger#Messenger
public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
//Handler#getIMessenger
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}
//Handler#MessengerImpl
//重点MessengerImp是继承IMessenger.Stub
private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}
```

使用Handler构造出来的Messenger的mTarget成员变量类型为IMessenger.Stub即Binder的实现类。

使用IBinder构造：
```java
/**其他代码省略**/
//重点mTarget为IMessenger.Stub.asInterface
public Messenger(IBinder target) {
	mTarget = IMessenger.Stub.asInterface(target);
}
```
IMessenger.java类中我们可以看到IMessenger.Stub.asInterface方法获得是IMessenger.Proxy类型对象，为Binder的代理类。这样构造出来的Messenger即为Binder的代理类。

通过源码的详细比较我们得出了结论：
> - 使用Handler构造出来的Messenger为Binder的实现类；
- 使用使用IBinder构造构造出来的Messenger为Binder的代理类；

到这里我们队Messenger代码的研究就差不多结束了，接下来我们现在实现一个简单的利用Messenger在两个不同的进程进行简单通信的例子，希望通过这个Demo的讲解能将我们前面的知识消化吸收。
现在看看服务端的代码
```java
public class MessengerService extends Service {

    private static final String TAG = "MessengerService";
    private static class MessengerHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            String msgString = msg.getData().getString("msg");
            if (!TextUtils.isEmpty(msgString)) {
                Log.d(TAG, "handleMessage: msg = [" + msgString + "]");
            }
        }
    }
    //创建service端处理消息的Messenger
    private final Messenger mMessenger = new Messenger(new MessengerHandler());
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```
客户端代码：
```java
public class MessengerActivity extends AppCompatActivity {
    private static final String TAG = "MessengerActivity";
    private Messenger mService;
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //构造客户端的Messenger往服务端发消息
            mService = new Messenger(service);
            Message msg = Message.obtain();
            Bundle data = new Bundle();
            String msgString = "hello ,this is client.";
            data.putString("msg", msgString);
            msg.setData(data);
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);
        bindService(new Intent(this, MessengerService.class),mConnection,
                Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}
```
将MessengerService在AndroidManifest.xml注册在另个一remote进程中：
```java
<service android:name=".MessengerService"
                 android:enabled="true"
                 android:exported="true"
                 android:process=":remote" />
```
在com.example.ybb.aidlexample:remote进程中的Logcat输出结果：
```java
06-28 14:55:29.793 22206-22206/com.example.ybb.aidlexample:remote D/MessengerService: handleMessage() : msg = [hello ,this is client.]
```

是不是感觉很简单呢？那么如果在这个代码上修改一下：让客户端和服务端能互相发送和接受消息。

首相我们回想一下：
> - 发送消息必须要得到远端的Binder对象来构造Messenger；
- 处理消息必须新建一个Handler来构造Messenger；

就以上这两点我们来重新写一下service端和Client端的代码：
Service端代码：
```java
public class MessengerService extends Service {

    private static final String TAG = "MessengerService";
    private static class MessengerHandler extends Handler{
        boolean received = false;
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            String msgString = msg.getData().getString("msg");
            if (!TextUtils.isEmpty(msgString)) {
                Log.d(TAG, "handleMessage: msg = [" + msgString + "]");
            }
            if (!received) {
                //利用Client端接受处理消息的Messenger来发送Message
                Messenger client = msg.replyTo;
                Message data = Message.obtain();
                Bundle bundle = new Bundle();
                bundle.putString("reply", "Your messages had received~!");
                data.setData(bundle);
                try {
                    client.send(data);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                received = true;
            }
        }
    }

    private final Messenger mMessenger = new Messenger(new MessengerHandler());
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```
Client端代码：
```java
public class MessengerActivity extends AppCompatActivity {
    private static final String TAG = "MessengerActivity";
    private static class ClientHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            String replyString = msg.getData().getString("reply");
            if (!TextUtils.isEmpty(replyString)) {
                Log.d(TAG, "handleMessage() : reply = [" + replyString + "]");
            }
        }
    }
    private Messenger clientHandler = new Messenger(new ClientHandler());
    private Messenger mService;
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mService = new Messenger(service);
            Message msg = Message.obtain();
            Bundle data = new Bundle();
            String msgString = "hello ,this is client.";
            data.putString("msg", msgString);
            msg.setData(data);
            //将处理消息的Messenger绑定到消息上带到服务端
            msg.replyTo = clientHandler;
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);
        bindService(new Intent(this, MessengerService.class),mConnection,
                Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}
```
运行结果：
<center>![messenger-logcat.jpg](http://upload-images.jianshu.io/upload_images/1319879-d5eb97beef5b0b79.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>