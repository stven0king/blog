---
title: ViewRootImpl的独白，我不是一个View(布局篇)
date: 2017-12-11 18:11:00
tags: [ViewRootImpl]
categories: Android
description: "前一段时间写过两篇关于View的文章 Activity中的Window的setContentView 和 遇见LayoutInflater&Factory 。分析了Activity设置页面布局到页面View元素进行布局到底经历了一个怎么样的过程？"
---

前言
===
前一段时间写过两篇关于View的文章 [Activity中的Window的setContentView](http://dandanlove.com/2017/11/10/activity-setcontentview/) 和 [遇见LayoutInflater&Factory](http://dandanlove.com/2017/11/15/layoutinflater-factory/) 。分析了Activity设置页面布局到页面View元素进行布局到底经历了一个怎么样的过程？
>- Activity的attach中生成PhoneWindow对象;
>- setContentView中初始化DecorView（ViewGroup）;
>- 在LayoutInflater进行对布局文件的解析之后更加解析的数据
>- 根据解析出的数据执行View的构造函数进行View的构造，同时生成ViewTree。

为什么接下来继续写这篇文章呢？是因为我在掘金上看到一篇子线程更新View的文章之后，发现自己对View还不是很了，以这个问题为方向看了View相关的源码。发现网络上有些文章对于ViewRootImpl的分析还是有些问题或者疑惑的，所以自己整理过的知识点分享给大家，希望能对大家有帮助。（`源码cm12.1`）

View的介绍
===
最开始学习View的时候最先分析的是它的布局（LinearLayout、FrameLayout、TableLayout、RelativeLayout、AbsoluteLayout）,然后是它的三大方法（measure、layout、draw）。

绘制&加载View-----onMeasure()
---
>- MeasureSpec.EXACTLY是精确尺寸， 当我们将控件的layout_width或layout_height指定为具体数值时如andorid:layout_width="50dip"，或者为FILL_PARENT是，都是控件大小已经确定的情况，都是精确尺寸。
>- MeasureSpec.AT_MOST是最大尺寸，当控件的layout_width或layout_height指定为WRAP_CONTENT时  ，控件大小一般随着控件的子空间或内容进行变化，此时控件尺寸只要不超过父控件允许的最大尺寸即可。因此，此时的mode是AT_MOST，size给出了父控件允许的最大尺寸。  
>- MeasureSpec.UNSPECIFIED是未指定尺寸，这种情况不多，一般都是父控件是AdapterView,通过measure方法传入的模式。

ViewGroup.java

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec)
protected void measureChild(View child, int parentWidthMeasureSpec,int parentHeightMeasureSpec)
protected void measureChildWithMargins(View child,int parentWidthMeasureSpec, int widthUsed,int parentHeightMeasureSpec, int heightUsed)
```


绘制&加载View-----onLayout()
---
>- onLayout方法：是ViewGroup中子View的布局方法。放置子View很简单，只需在重写onLayout方法，然后获取子View的实例，调用子View的layout方法实现布局。在实际开发中，一般要配合onMeasure测量方法一起使用。View的放置都是根据一个矩形空间放置的。
>- layout方法：是View的放置方法，在View类实现。调用该方法需要传入放置View的矩形空间左上角left、top值和右下角right、bottom值。

绘制&加载View-----onDraw()
---

```java
public void draw(Canvas canvas)
protected void onDraw(Canvas canvas)
protected void dispatchDraw(Canvas canvas)(View,ViewGroup)
protected boolean drawChild(Canvas canvas, View child, long drawingTime) (ViewGroup)
```

<center>![ViewTree.jpg](http://upload-images.jianshu.io/upload_images/1319879-92ccbcbdca85efdc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

View的解析与生成
===

View的解析和生成之前在下边的这两篇文章中已经讲述

> View如何在页面进行展示的，View树是如何生成的。
[Activity中的Window的setContentView](http://dandanlove.com/2017/11/10/activity-setcontentview/)

> View对象的生成，属性值的初始化。
[遇见LayoutInflater&Factory](http://dandanlove.com/2017/11/15/layoutinflater-factory/)

在这两篇文章中用到了一些Android中相关的类：

>- Activity：一个Activity是一个应用程序组件,提供一个屏幕,用户可以用来交互为了完成某项任务,例如拨号、拍照、发送email……。
>- View：作为所有图形的基类。
>- ViewGroup:对View继承扩展为视图容器类。
>- Window:它概括了Android窗口的基本属性和基本功能。(抽象类)
>- PhoneWindow:Window的子类。
>- DecorView：界面的根View，PhoneWindow的内部类。
>- ViewRootImpl:ViewRoot是GUI管理系统与GUI呈现系统之间的桥梁。
>- WindowManangerService：简称WMS,它的作用是管理所有应用程序中的窗口，并用于管理用户与这些窗口发生的的各种交互。

ViewRootImpl简介
===

`The top of a view hierarchy, implementing the needed protocol between View and the WindowManager.  This is for the most part an internal implementation detail of {@link WindowManagerGlobal}.`


```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
    /*******部分代码省略**********/
}
```

ViewRootImpl是View中的最高层级，属于所有View的根（`但ViewRootImpl不是View，只是实现了ViewParent接口`），实现了View和WindowManager之间的通信协议，实现的具体细节在WindowManagerGlobal这个类当中。

<center>![ViewManager.png](http://upload-images.jianshu.io/upload_images/1319879-295a6896e2a27237.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>


```java
public interface ViewManager{
    //添加View
    public void addView(View view, ViewGroup.LayoutParams params);
    //更新View
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    //移除View
    public void removeView(View view);
}
```

上面提到了`WindowManager`对`View`的添加、更新、删除操作，那么联系这两者之间的`Window`呢？

>  [Activity中的Window的setContentView](http://dandanlove.com/2017/11/10/activity-setcontentview/) 阅读这篇文章我们知道Activity中有Window对象，一个Window对象对应着一个View（`DecorView`），`ViewRootImpl`就是对这个View进行操作的。

我们知道界面所有的元素都是有View构成的，界面上的每一个像素点也都是由View绘制的。Window只是一个抽象的概念，把界面抽象为一个窗口对象，也可以抽象为一个View。

ViewRootImpl与其他类之间的关系
===
在了解ViewRootImpl与其他类的关系之前，我们先看一副图和一段代码：

<center>![ViewRootImpl.jpg](http://upload-images.jianshu.io/upload_images/1319879-b845406a6c472c2d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>



```java
public final class WindowManagerGlobal {
    /*******部分代码省略**********/
    //所有Window对象中的View
    private final ArrayList<View> mViews = new ArrayList<View>();
    //所有Window对象中的View所对应的ViewRootImpl
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    //所有Window对象中的View所对应的布局参数
    private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.LayoutParams>();
    /*******部分代码省略**********/
}
```


> `WindowManagerGlobal`在其内部存储着`ViewRootImpl`和`View`实例的映射关系(顺序存储)。

> `WindowManager`继承与`ViewManger`，从`ViewManager`这个类名来看就是用来对`View`类进行管理的，从`ViewManager`接口中的添加、更新、删除View的方法也可以看出来`WindowManager`对`View`的管理。

```java
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
    /*******部分代码省略**********/
}
```

- `WindowManagerImpl`为`WindowManager`的实现类。`WindowManagerImpl`内部方法实现都是由代理类`WindowManagerGlobal`完成，而`WindowManagerGlobal`是一个单例，也就是一个进程中只有一个`WindowManagerGlobal`对象服务于所有页面的View。


ViewRootImpl的初始化
===

在Activity的onResume之后，当前Activity的Window对象中的View会被添加在WindowManager中。


```java
public final class ActivityThread {
    /*******部分代码省略**********/
    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
        /*******部分代码省略**********/

        // TODO Push resumeArgs into the activity for consideration
        ActivityClientRecord r = performResumeActivity(token, clearHide);

        if (r != null) {
            /*******部分代码省略**********/
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                //window的类型：一个应用窗口类型（所有的应用窗口类型都展现在最顶部）。
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    //将decor添加在WindowManager中
                    wm.addView(decor, l);
                }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }

            /*******部分代码省略**********/

        } else {
            // If an exception was thrown when trying to resume, then
            // just end this activity.
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
            }
        }
    }
}
```

创建Window所对应的ViewRootImpl，并将Window所对应的View、ViewRootImpl、LayoutParams顺序添加在WindowManager中。


```java
public final class WindowManagerGlobal {
    /*******部分代码省略**********/
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        //检查参数是否合法
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent and we're running on L or above (or in the
            // system context), assume we want hardware acceleration.
            final Context context = view.getContext();
            if (context != null
                    && context.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        //声明ViwRootImpl
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }
            //创建ViwRootImpl
            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);
            //将Window所对应的View、ViewRootImpl、LayoutParams顺序添加在WindowManager中
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {
            //把将Window所对应的View设置给创建的ViewRootImpl
            //通过ViewRootImpl来更新界面并完成Window的添加过程。
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }
}
```

ViewRootImpl绑定Window所对应的View
===
ViewRootImpl绑定Window所对应的View，并对该View进行测量、布局、绘制等。

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
    /*******部分代码省略**********/
    /**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                //ViewRootImpl成员变量view进行复制，以后操作的都是mView。
                mView = view;
                /*******部分代码省略**********/
                //Window在添加完之前先进行一次布局，确保以后能再接受系统其它事件之后重新布局。
                //对View完成异步刷新，执行View的绘制方法。
                requestLayout();
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    //将该Window添加到屏幕。
                    //mWindowSession实现了IWindowSession接口，它是Session的客户端Binder对象.
                    //addToDisplay是一次AIDL的跨进程通信，通知WindowManagerService添加IWindow
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets, mInputChannel);
                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
                /*******部分代码省略**********/
                //设置当前View的mParent
                view.assignParent(this);
                /*******部分代码省略**********/
            }
        }
    }
}
```


ViewRootImpl对mView进行操作
===

对View的操作包括文章最开始讲述的测量、布局、绘制，其过程主要是在ViewRootImpl的performTraversals方法中。

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
    /*******部分代码省略**********/
    //请求对界面进行布局
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
    //校验所在线程，mThread是在ViewRootImpl初始化的时候执行mThread = Thread.currentThread()进行赋值的，也就是初始化ViewRootImpl所在的线程。
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
    //安排任务
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
        }
    }

    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    //做任务
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "performTraversals");
            try {
                //执行任务
                performTraversals();
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }

    //执行任务（分别调用mView的measure、layout、draw）
    private void performTraversals() {
        // cache mView since it is used so much below...
        final View host = mView;
        /*******部分代码省略**********/
        //想要展示窗口的宽高
        int desiredWindowWidth;
        int desiredWindowHeight;

        //开始进行布局准备
        if (mFirst || windowShouldResize || insetsChanged ||
            viewVisibilityChanged || params != null) {
            /*******部分代码省略**********/
            if (!mStopped) {
                boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                        (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
                if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged) {
                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

                    
                     // Ask host how big it wants to be
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                    // Implementation of weights from WindowManager.LayoutParams
                    // We just grow the dimensions as needed and re-measure if
                    // needs be
                    int width = host.getMeasuredWidth();
                    int height = host.getMeasuredHeight();
                    boolean measureAgain = false;

                    /*******部分代码省略**********/

                    if (measureAgain) {
                        //View的测量
                        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    }

                    layoutRequested = true;
                }
            }
        } else {
            /*******部分代码省略**********/
        }

        final boolean didLayout = layoutRequested /*&& !mStopped*/ ;
        boolean triggerGlobalLayoutListener = didLayout
                || mAttachInfo.mRecomputeGlobalAttributes;
        if (didLayout) {
            //View的布局
            performLayout(lp, desiredWindowWidth, desiredWindowHeight);

            /*******部分代码省略**********/
        }
        /*******部分代码省略**********/
        
        if (!cancelDraw && !newSurface) {
            if (!skipDraw || mReportNextDraw) {
                /*******部分代码省略**********/
                //View的绘制
                performDraw();
            }
        } else {
            if (viewVisibility == View.VISIBLE) {
                // Try again
                scheduleTraversals();
            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }
        }

        mIsInTraversal = false;
    }
}
```

View的测量
---
ViewRootImpl调用performMeasure执行Window对应的View的测量。

>- ViewRootImpl的performMeasure;
>- DecorView(FrameLayout)的measure;
>- DecorView(FrameLayout)的onMeasure;
>- DecorView(FrameLayout)所有子View的measure；

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
    /*******部分代码省略**********/
    //View的测量
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            //mView在Activity中为DecorView（FrameLayout）
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
}
```

View的布局
---
ViewRootImpl调用performLayout执行Window对应的View的布局。

>- ViewRootImpl的performLayout；
>- DecorView(FrameLayout)的layout方法；
>- DecorView(FrameLayout)的onLayout方法；
>- DecorView(FrameLayout)的layoutChildren方法；
>- DecorView(FrameLayout)的所有子View的Layout；

在这期间可能View会自己触发布布局请求，所以在此过程会在此调用ViewRootImpl的requestLayout重新进行测量、布局、绘制。

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
    /*******部分代码省略**********/

    //请求对ViewRootImpl进行布局
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

    //View的布局
    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        /*******部分代码省略**********/
        final View host = mView;
        /*******部分代码省略**********/
        try {
            //调用View的Layout方法进行布局
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
            mInLayout = false;
            //在ViewRootImpl进行布局的期间，Window内的View自己进行requestLayout
            int numViewsRequestingLayout = mLayoutRequesters.size();
            if (numViewsRequestingLayout > 0) {
                // requestLayout() was called during layout.
                // If no layout-request flags are set on the requesting views, there is no problem.
                // If some requests are still pending, then we need to clear those flags and do
                // a full request/measure/layout pass to handle this situation.
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);
                if (validLayoutRequesters != null) {
                    // Set this flag to indicate that any further requests are happening during
                    // the second pass, which may result in posting those requests to the next
                    // frame instead
                    mHandlingLayoutInLayoutRequest = true;

                    // Process fresh layout requests, then measure and layout
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        Log.w("View", "requestLayout() improperly called by " + view +
                                " during layout: running second layout pass");
                        //请求对该View布局，最终回调到ViewRootImpl的requestLayout进行重新测量、布局、绘制
                        view.requestLayout();
                    }
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
                    mInLayout = true;
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                    mHandlingLayoutInLayoutRequest = false;

                    // Check the valid requests again, this time without checking/clearing the
                    // layout flags, since requests happening during the second pass get noop'd
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        // Post second-pass requests to the next frame
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    /*******部分代码省略**********/
                                    //请求对该View布局，最终回调到ViewRootImpl的requestLayout进行重新测量、布局、绘制
                                    view.requestLayout();
                                }
                            }
                        });
                    }
                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
}
```

View的绘制
---
ViewRootImpl调用performDraw执行Window对应的View的布局。

>- ViewRootImpl的performDraw；
>- ViewRootImpl的draw；
>- ViewRootImpl的drawSoftware；
>- DecorView(FrameLayout)的draw方法；
>- DecorView(FrameLayout)的dispatchDraw方法；
>- DecorView(FrameLayout)的drawChild方法；
>- DecorView(FrameLayout)的所有子View的draw方法；

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, HardwareRenderer.HardwareDrawCallbacks {
    /*******部分代码省略**********/
    //View的绘制
    private void performDraw() {
        if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
            return;
        }

        final boolean fullRedrawNeeded = mFullRedrawNeeded;
        mFullRedrawNeeded = false;

        mIsDrawing = true;
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
        try {
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        /*******部分代码省略**********/
    }
    //进行绘制
    private void draw(boolean fullRedrawNeeded) {
        
        /*******部分代码省略**********/
        //View上添加的Observer进行绘制事件的分发
        mAttachInfo.mTreeObserver.dispatchOnDraw();

        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
                /*******部分代码省略**********/
                //调用Window对应的ViewRootImpl的invalidate方法
                mAttachInfo.mHardwareRenderer.draw(mView, mAttachInfo, this);
            } else {
                /*******部分代码省略**********/
                //绘制Window
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                    return;
                }
            }
        }

        if (animating) {
            mFullRedrawNeeded = true;
            scheduleTraversals();
        }
    }

    void invalidate() {
        mDirty.set(0, 0, mWidth, mHeight);
        if (!mWillDrawSoon) {
            scheduleTraversals();
        }
    }

    /**
     * @return true if drawing was successful, false if an error occurred
     */
    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {

        /*******部分代码省略**********/
        try {
            /*******部分代码省略**********/
            try {
                canvas.translate(-xoff, -yoff);
                if (mTranslator != null) {
                    mTranslator.translateCanvas(canvas);
                }
                canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
                attachInfo.mSetIgnoreDirtyState = false;
                //View绘制
                mView.draw(canvas);

                drawAccessibilityFocusedDrawableIfNeeded(canvas);
            } finally {
                if (!attachInfo.mSetIgnoreDirtyState) {
                    // Only clear the flag if it was not set during the mView.draw() call
                    attachInfo.mIgnoreDirtyState = false;
                }
            }
        } finally {
            /*******部分代码省略**********/
        }
        return true;
    }
}
```

异步线程更新视图
===
异步线程中到底是否能对视图进行更新呢？我们先看看`TextView.setText`方法的代码执行流程图。

<center>![setText.png](http://upload-images.jianshu.io/upload_images/1319879-2c7e4d1df9ee3ff7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>

从上图看在页面进行视图更新的时候会触发`checkThread`,校验当前线程是否是`ViewRootImpl`被创建时所在的线程。而`ViewRootImpl`的创建是在Activity的`onResume`生命周期之后。

>- 我们可以再`onResume`之前在异步线程进行视图更新,因为这个时候不会发生线程校验。
>- 我们可以再异步线程初始化`ViewRootImpl`同时在该线程进行视图更新。

PS:我们学习Android源码并通过了解其内部实现，我们可以通过技术手段去掉、增加或修改部分代码逻辑。以期望能做出更好的产品、做更细节的优化。但是想这种界面绘制只能发生在主线程规则我们还是必须要遵守的。`两个线程不能同时draw，否则屏幕会花；不能同时写一块内存，否则内存会花；不能同时写一份文件，否则文件会花。同一时刻只有一个线程可以做ui，那么当两个线程互斥几率较大时，或者保证互斥的代码复杂时，选择其中一个长期持有其他发消息就是典型的解决方案。所以普遍的要求ui只能单线程。`

Activity、Dialog、Toast的ViewRootImpl的不同
===
文章前面内容都是站在Activity的角度来进行代码解析的，因此我们不再对Dialog和Toast与Activity做具体分析，主要来看看它们与Activity有什么不同之处。详见：[Dialog、Toast的Window和ViewRootImpl](http://dandanlove.com/2017/12/11/viewrootimpl-dialog-toast/)。

总结
===
通过对ViewRootImpl的更细节的分析，我们再看自定义View的布局时的一些方法会更加清楚（知其然且知其所以然）。同时也解开了`Android中异步线程更新View`的谜底。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>