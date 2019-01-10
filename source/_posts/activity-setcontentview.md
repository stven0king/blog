---
title: Activity中的Window的setContentView
date: 2017-11-10 21:29:00
tags: [Window,Decor]
categories: Android
description: "这篇文章距离现在已经两年的时间了。当初自己刚毕业工作不久，才开始接触Android，有一天中午和同事一起吃饭的时候，一个大牛问我你思考过Activity的setContentView是怎么执行的么。当初就因为这个问题我接入到了Android源码。两年时间过去了现在回过头来看，感觉自己写得有很多的不足，本次再补充一下。"
---
这篇文章距离现在已经两年的时间了。当初自己刚毕业工作不久，才开始接触Android，有一天中午和同事一起吃饭的时候，一个大牛问我你思考过Activity的setContentView是怎么执行的么。当初就因为这个问题我接入到了Android源码。两年时间过去了现在回过头来看，感觉自己写得有很多的不足，本次再补充一下。


前言
===
这几天正在进行初级自定义组件的学习，一不小心想到了view到底是怎么加载到屏幕上面的。每一个Activity中都有一个方法setContentView,我们可以加载自己想要的界面布局，展示在手机屏幕上。但到底内部是怎么实现的呢？（PS:源码基于Android5.1，cm12.1）

Activity的onContentView
===
首先查看Activity的onContentView的方法：

```
//Activity.java
public void setContentView(int layoutResID) {
     getWindow().setContentView(layoutResID);
     initActionBar();
}
public void setContentView(View view) {
    getWindow().setContentView(view);
    initActionBar();
}
public void setContentView(View view, ViewGroup.LayoutParams params) {
    getWindow().setContentView(view, params);
    initActionBar();
}
```

Activity一共重载了三个setContentView方法，其中第一个setContentView(int layoutResID)方法是我们常用的。

```
public void setContentView(int layoutResID) {
     //getWindow()获取activity内部对象mWindow并调用它的setContentView方法
      getWindow().setContentView(layoutResID);
      initActionBar(); //这是初始化actionBar，我们不关注它
}
public Window getWindow() {
    return mWindow;
}
```

Activity的setContentView方法实际还是调用mWindow的setContentView方法，接下看我们试看看mWindow的相关代码。

mWindow对象
===
查看Activity源码，找到在attach方法中对mWindow做了赋值。


```
final void attach(Context context, ActivityThread aThread,
           Instrumentation instr, IBinder token, int ident,
           Application application, Intent intent, ActivityInfo info,
           CharSequence title, Activity parent, String id,
           NonConfigurationInstances lastNonConfigurationInstances,
           Configuration config) {
    attachBaseContext(context);
    
    mFragments.attachActivity(this, mContainer, null);
    
    mWindow = PolicyManager.makeNewWindow(this);
    mWindow.setCallback(this);
    /…部分代码省略…/
}
```

那么Activity的attach方法是Activity生命周期的第一个方法，它是ActivityThread中performLaunchActivity方法调用的，这是通过AMS(ActivityManagerService)的startActivity调用ActivityTrack的startActivityMayWait来调用的。

attach字面意思就是“使依附；贴上；系上”，也就是点击activity进行启动的时候之执行的。

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    /*******部分代码省略********/
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        /*******部分代码省略********/

        if (activity != null) {
            Context appContext = createBaseContextForActivity(r, activity);
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor);

            /*******部分代码省略********/
}
```

如述源代码当中就是在启动Activity的时候执行其attach：

>- ApplicationThread#scheduleLaunchActivity
>- ActivityThread#handleLaunchActivity
>- ActivityThread#performLaunchActivity
>- Activity#attach

PolicyManager获取Window对象
===

```java
public final class PolicyManager {
    private static final String POLICY_IMPL_CLASS_NAME =
        "com.android.internal.policy.impl.Policy";

    private static final IPolicy sPolicy;

    static {
        // Pull in the actual implementation of the policy at run-time
        try {
            Class policyClass = Class.forName(POLICY_IMPL_CLASS_NAME);
            sPolicy = (IPolicy)policyClass.newInstance();
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be loaded", ex);
        } catch (InstantiationException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        }
    }

    // Cannot instantiate this class
    private PolicyManager() {}

    // The static methods to spawn new policy-specific objects
    public static Window makeNewWindow(Context context) {
        return sPolicy.makeNewWindow(context);
    }
    /*******部分代码省略********/
}
```

PolicyManager.makeNewWindow方法实际是通过反射机制调用了"com.android.internal.policy.impl.Policy"的makeNewWindow方法。

```java
public class Policy implements IPolicy {
    private static final String TAG = "PhonePolicy";

    private static final String[] preload_classes = {
        "com.android.internal.policy.impl.PhoneLayoutInflater",
        "com.android.internal.policy.impl.PhoneWindow",
        "com.android.internal.policy.impl.PhoneWindow$1",
        "com.android.internal.policy.impl.PhoneWindow$DialogMenuCallback",
        "com.android.internal.policy.impl.PhoneWindow$DecorView",
        "com.android.internal.policy.impl.PhoneWindow$PanelFeatureState",
        "com.android.internal.policy.impl.PhoneWindow$PanelFeatureState$SavedState",
    };

    static {
        // For performance reasons, preload some policy specific classes when
        // the policy gets loaded.
        for (String s : preload_classes) {
            try {
                Class.forName(s);
            } catch (ClassNotFoundException ex) {
                Log.e(TAG, "Could not preload class for phone policy: " + s);
            }
        }
    }

    public Window makeNewWindow(Context context) {
        return new PhoneWindow(context);
    }

    /*******部分代码省略********/
}
```

Policy的makeNewWindow方法实际是返回一个PhoneWindow对象。

PhoneWindow.setContentView
===


```java
public class PhoneWindow extends Window implements MenuBuilder.Callback {
    /*******部分代码省略********/
    @Override
    public void setContentView(View view) {
        setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }

    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            //构造DecorView对象并赋值给mDecor，并进行mContentParent的初始化
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            view.setLayoutParams(params);
            final Scene newScene = new Scene(mContentParent, view);
            transitionTo(newScene);
        } else {
            //将附带params属性的view对象添加在mContentParent中
            mContentParent.addView(view, params);
        }
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            //将Resource对于的id等于layoutResID的xml布局文件，add到mContentParent中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
    /*******部分代码省略********/
}
```

setContentView主要做了两件事：
>- 初始化整个界面（即：DecorView）
>- 将setContentView的参数对于的View，add到mContentParent中。

addView
===
setContentView方法有两种在界面添加View的方法。
- 调用mContentParent的add方法，将目标View添加进去。
- 调用LayoutInfater.inflate方法将资源xml解析并转化为View，添加到mContentParent中。 

installDecor
===
在看installDecor方法的源代码的时候，我先让大家看一个Android手机界面的布局文件的分析图。
<center>![Android手机界面的布局](http://upload-images.jianshu.io/upload_images/1319879-7b9e921ef2419900?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

<center>![PhoneWindow](http://upload-images.jianshu.io/upload_images/1319879-800e67e27b8d6817?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

PhoneWindow.java部分代码

```java
protected DecorView generateDecor() {
    return new DecorView(getContext(), -1);
}
private void installDecor() {
    if (mDecor == null) {
        //构造mDecor对象DecorView
        mDecor = generateDecor();
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    }
    if (mContentParent == null) {
        //构造mContentParent
        mContentParent = generateLayout(mDecor);

        // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
        mDecor.makeOptionalFitsSystemWindows();

        final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);

        if (decorContentParent != null) {
            //mDecorContentParent赋值R.id.decor_content_parent
            mDecorContentParent = decorContentParent;
            mDecorContentParent.setWindowCallback(getCallback());
            if (mDecorContentParent.getTitle() == null) {
                mDecorContentParent.setWindowTitle(mTitle);
            }   
   /*******部分代码省略********/
}
```

> mDecorContentParent为mDecor中的R.id.decor_content_parent

installDecor先构造mDecor，然后通过mDecor执行generateLayout()方法初始化mContentParent。

```java
protected ViewGroup generateLayout(DecorView decor) {
     /*******部分代码省略********/
    View in = mLayoutInflater.inflate(layoutResource, null);
    decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    mContentRoot = (ViewGroup) in;

    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }
     /*******部分代码省略********/
    return contentParent; 
}
```

> mContentRoot为decor的content，经测试(mContentRoot == mDecorContentParent)为true。
generateLayout(DecorView decor)方法构造出来的mContentParent为ID_ANDROID_CONTENT，即mDecor中的R.id.content。

从代码中可以看出显示获取当前窗口的根ViewGroup（mDecor），然后往这个ViewGroup中添加view。

最终我们要展示在Activity中的View已经构造好了，那么在Activity的`onResume` 方法之后，在 `ActivityThread#handleResumeActivity` 方法中会将该View通过WindowManager添加在Activity所挂在的Window上进行展现。

mDecor是什么可以参考博客：[DecorView浅析](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2013/0322/1054.html)

好了学习过程到此结束~！
下边介绍在我学习过程中膜拜的博客，感觉这些大牛就是点亮我前行的灯塔，哈哈哈。
[Android View的加载过程](http://blog.csdn.net/xyz_lmn/article/details/20122303)
[Android应用setContentView与LayoutInflater加载解析机制源码分析](http://www.2cto.com/kf/201505/402754.html)
[android的窗口机制分析------UI管理系统](http://blog.csdn.net/windskier/article/details/6957854)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)