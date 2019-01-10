---
title: 关于RxJava的基础心法解析
date: 2019-01-07 20:11:00
tags: [Observable, Observer]
categories: RxJava
description: "读完本篇文章希望所有读者能明白RxJava的观察者与java的观察者模式有什么不同，以及Rxjava的观察者模式的代码运行过程。"
---


# 前言

我接触Rxjava是在2015年底，已经过去4年的时间了。

2016年学习过一阵子`RxJava`的操作符也做过一些笔记，我们项目的网络请求框架也替换成了`Okhttp+Retrofit`，所以使用`RxJava`做线程间切换就非常好用。

一开始接触`RxJava`感觉除了线程切换之外很能发现其实际的作用，因为我感觉自己响应式编程的思想，很难实际运用到开发需求当中去。但我身边有一位前辈使用Rxjava非常溜，他一般做需求的时候写的都是流式的代码。

2017年`Kotlin`语言Google举行的I/O开发者大会上宣布，将`Kotlin`语言作为安卓开发的一级编程语言，所以自己又看了看了`Kotlin`语言。

`RxJava`在我们项目中还是静静的躺着，因为自己懒的思考，懒的在代码结构上做更新，懒的对`RxJava`做研究。有时候感觉自己就算会了`RxJava`也不会将其使用在项目当中，因为自己什么业务场景之下使用`Rxjava`更加方便。

2018就这么有一下没一下的使用`RxJava`，最近在做需求开发的时候用的`RxJava`比较多了，一些业务场景也逐渐思考使用响应式编程。思考这样写的好处，以及怎么将之前的代码结构转化为流式结构。

感觉有时候思维观念的转变是一个漫长的过程，但有时候又会很快。凡事都可以熟能生巧，我们使用`RxJava`多了之后再笨也会思考。之前想不到`RxJava`的使用场景是因为自己见的、写的代码还不够多。

今天回过头来从代码的角度看看一次`RxJava` 的基础操作，事件订阅到触发的过程。

这里推荐一篇`RxJava`的入门的文章 [给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083) 。

读完本篇文章希望所有读者能明白`RxJava`的观察者与java的观察者模式有什么不同，以及`Rxjava`的观察者模式的代码运行过程。至于怎么具体的使用 `Rxjava` 那么就需要更多学习和实践了。



# Java的观察者模式

观察者：`Observer`

被观察者：`Observable`

> 被观察者提供添加（注册）观察者的方法；
>
> 被观察者更新的同时可以主动通知注册它观察者更新；

观察者模式面向的需求是：收音机听广播，电台是被观察者，收音机是观察者。收音机调频到广播的波段（注册），广播发送信息（被观察者更新数据，通知所有的观察者）收音机接受信息从而播放声音（观察者数据更新）。



# RxJava的观察者模式

可观察者（被观察者）：`Observalbe`

观察者：`Observer`

订阅操作：`subscribe()`

订阅：`Subscription`

订阅者：`Subscriber` ，实现 `Observer` 和 `Subscription`

> `Observable` 和 `Subscriber` 通过 `subscribe()` 方法实现订阅关系，从而 `Observable` 可以在需要的时候发出事件来通知 `Subscriber`。

<center>![RxJavaObservable.png](https://upload-images.jianshu.io/upload_images/1319879-399d99f474b65019.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>



# 分析源代码

我们先看一个简单的打印数据的例子：



```java
Observable.just(1)
    .subscribe(new Subscriber<Object>() {
        @Override
        public void onCompleted() {
            System.out.println("onCompleted");
        }
        @Override
        public void onError(Throwable e) {
            System.out.println("onError");
        }
        @Override
        public void onNext(Object o) {
            System.out.println("onNext");
            System.out.println(Integer.valueOf(o.toString()));
        }
    });
```



`RxJava` 的版本已经更新到2了，我们现在还用的是版本1。版本1中1.0和1.3这两个版本用的比较多。但这两个`RxJava` 版本之前改动不是很大，我们来分析分析最初始的版本，主要看看其中的设计思想啥的~！



## 谁触发了被观察者

我们进行了 `subscribe` 之后就会触发 `Observable` 的执行动作，然后将执行结果传输给订阅它的 `Subscriber` 。



```java
//无参的subscribe
public final Subscription subscribe() {
    return subscribe(new Subscriber<T>() {
        @Override
        public final void onCompleted() {}
        @Override
        public final void onError(Throwable e) {throw new OnErrorNotImplementedException(e);}
        @Override
        public final void onNext(T args) {}
    });
}
/***部分代码省略***/
//onnext、onerror、oncomplete参数
public final Subscription subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError, final Action0 onComplete) {
    if (onNext == null) {
        throw new IllegalArgumentException("onNext can not be null");
    }
    if (onError == null) {
        throw new IllegalArgumentException("onError can not be null");
    }
    if (onComplete == null) {
        throw new IllegalArgumentException("onComplete can not be null");
    }
    return subscribe(new Subscriber<T>() {
        @Override
        public final void onCompleted() {onComplete.call();}
        @Override
        public final void onError(Throwable e) {onError.call(e);}
        @Override
        public final void onNext(T args) {onNext.call(args);}
    });
}
//observer参数
public final Subscription subscribe(final Observer<? super T> observer) {
    return subscribe(new Subscriber<T>() {
        @Override
        public void onCompleted() {observer.onCompleted();}
        @Override
        public void onError(Throwable e) {observer.onError(e);}
        @Override
        public void onNext(T t) {observer.onNext(t);}
    });
}
//Subscriber参数
public final Subscription subscribe(Subscriber<? super T> subscriber) {
    if (subscriber == null) {
        throw new IllegalArgumentException("observer can not be null");
    }
    if (onSubscribe == null) {
        throw new IllegalStateException("onSubscribe function can not be null.");
    }
    //调用订阅者的start方法
    subscriber.onStart();
    if (!(subscriber instanceof SafeSubscriber)) {
        subscriber = new SafeSubscriber<T>(subscriber);
    }
    try {
        //Observable的OnSubscribe调用call方法
        hook.onSubscribeStart(this, onSubscribe).call(subscriber);
        return hook.onSubscribeReturn(subscriber);
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        try {
            //调用订阅者的onError方法
            subscriber.onError(hook.onSubscribeError(e));
        } catch (OnErrorNotImplementedException e2) {
            throw e2;
        } catch (Throwable e2) {
            RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
            hook.onSubscribeError(r);
            throw r;
        }
        return Subscriptions.unsubscribed();
    }
}
```


```java
private static final RxJavaObservableExecutionHook hook = RxJavaPlugins.getInstance().getObservableExecutionHook();

class RxJavaObservableExecutionHookDefault extends RxJavaObservableExecutionHook {
    private static RxJavaObservableExecutionHookDefault INSTANCE = new RxJavaObservableExecutionHookDefault();
    public static RxJavaObservableExecutionHook getInstance() {
        return INSTANCE;
    }
}
public abstract class RxJavaObservableExecutionHook {
    public <T> OnSubscribe<T> onSubscribeStart(Observable<? extends T> observableInstance, final OnSubscribe<T> onSubscribe) {
        // pass-thru by default
        return onSubscribe;
    }
}
```



从 `RxJavaObservableExecutionHookonSubscribeStart`  可以看出  `hook.onSubscribeStart(this, onSubscribe).call(subscriber);`  实际是 `onSubscribe.call(subscirber)` 。



## 开始你的表演：Observable.OnSubscribe.call

刚才我们了解到通过 `subscirbe` 可以通知被观察者进行 `call` 操作。



```java

public class Observable<T> {
    protected Observable(OnSubscribe<T> f) {
        this.onSubscribe = f;
    }
	/***部分代码省略***/
    public final static <T> Observable<T> just(final T value) {
        return ScalarSynchronousObservable.create(value);
    }
}

public final class ScalarSynchronousObservable<T> extends Observable<T> {
    public static final <T> ScalarSynchronousObservable<T> create(T t) {
        return new ScalarSynchronousObservable<T>(t);
    }
    private final T t;
    protected ScalarSynchronousObservable(final T t) {
        super(new OnSubscribe<T>() {
            @Override
            public void call(Subscriber<? super T> s) {
                s.onNext(t);
                s.onCompleted();
            }
        });
        this.t = t;
    }
    public T get() {
        return t;
    }
}

```



我们最后看到 `Observable.just(1)` 生成的 `Observable` 实际是 `ScalarSynchronousObservable` 实例。  `Observable.OnSubscribe`  实际上是：



```java
new OnSubscribe<T>() {
    @Override
    public void call(Subscriber<? super T> s) {
        s.onNext(t);
        s.onCompleted();
    }
}
```



所以 `onSubscribe.call(subscirber)`  最终调用的是了 `subscirber` 的 `onNext和onCompleted` 方法。



# 总结

对于Android开发人员开发来说 `RxJava` 是一个很好用的库，但是需要我们转化平时的对代码结构设计的思想，能很好的去使用到大部分的业务场景之中。只有对 `RxJava` 有了足够的了解我们才能灵活、熟练的使用。

本篇文章只是一个 `RxJava` 简单的基础开篇，`观察者：Observer`  、 `订阅操作：subscribe()`  、`订阅：Subscription` 、`订阅者：Subscriber`  以及 `Observer` 和 `Subscription` 的订阅关系，之后我会慢慢的学习和分享关于 `RxJava` 更多的知识。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>