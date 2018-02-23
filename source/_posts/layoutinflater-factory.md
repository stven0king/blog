---
title: 遇见LayoutInflater&Factory
date: 2017-11-15 17:05:00
tags: [LayoutInflater,Factory]
categories: Android
description: "在我们写listview的adapter的getView方法中我们都会通过LayoutInflater.from(mContext)获取LayoutInflater实例然后调用inflate方法创建View。这个有xml布局文件转化为View对象的过程到底是怎么样的，我们今天通过源码来了解一下。"
---
<center>![奥体公园](http://upload-images.jianshu.io/upload_images/1319879-dc1d123cac6ec4ad.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)</center>

LayoutInflater的获取
---
在我们写listview的adapter的getView方法中我们都会通过 `LayoutInflater.from(mContext)` 获取LayoutInflater实例。
现在我们通过源码来分析一下LayoutInflater实例的获取：

```java
//LayoutInflater的获取
public abstract class LayoutInflater {
    /**
     * Obtains the LayoutInflater from the given context.
     */
    public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
}
```



`context.getSystemService` 是Android很重要的一个API，它是Activity的一个方法，根据传入的NAME来取得对应的Object，然后转换成相应的服务对象。以下介绍系统相应的服务。

|Name   |返回的对象|说明|
| :-------- | :-------- | :-------- |
|WINDOW_SERVICE|WindowManager|管理打开的窗口程序|
|LAYOUT_INFLATER_SERVICE|LayoutInflater|取得xml里定义的view|
|ACTIVITY_SERVICE|ActivityManager|管理应用程序的系统状态|
|POWER_SERVICE|PowerManger|电源的服务|
|ALARM_SERVICE|AlarmManager|闹钟的服务|
|NOTIFICATION_SERVICE|NotificationManager|状态栏的服务|
|KEYGUARD_SERVICE|KeyguardManager|键盘锁的服务|
|LOCATION_SERVICE|LocationManager|位置的服务，如GPS|
|SEARCH_SERVICE|SearchManager|搜索的服务|
|VEBRATOR_SERVICE|Vebrator|手机震动的服务|
|CONNECTIVITY_SERVICE|Connectivity|网络连接的服务|
|WIFI_SERVICE|WifiManager|Wi-Fi服务|
|TELEPHONY_SERVICE|TeleponyManager|电话服务|

获取LayoutInflater服务
---

```java
class ContextImpl extends Context {
    /***部分代码省略****/
    static {
        /***部分代码省略****/
        registerService(LAYOUT_INFLATER_SERVICE, new ServiceFetcher() {
            public Object createService(ContextImpl ctx) {
                return PolicyManager.makeNewLayoutInflater(ctx.getOuterContext());
            }});
        /***部分代码省略****/
    }
    /***部分代码省略****/
}
```

从源码可以看出LayoutInflater实例是由 `PolicyManager.makeNewLayoutInflater`获取的，PolicyManager有没有感觉很熟悉。上一章 [Activity中的Window的setContentView](//dandanlove.com/2017/11/10/activity-setcontentview/)  中我们获取Activity中的Window的实例的时候就是通过PolicyManager获取的，我们进一步往下跟进。

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
    /***部分代码省略****/

    public static LayoutInflater makeNewLayoutInflater(Context context) {
        //反射获取实例
        return sPolicy.makeNewLayoutInflater(context);
    }
}

public class Policy implements IPolicy {
    /***部分代码省略****/
    public LayoutInflater makeNewLayoutInflater(Context context) {
        //LayoutInflater的最终实例
        return new PhoneLayoutInflater(context);
    }
    /***部分代码省略****/
}
```

PhoneLayoutInflater的实现
---

```java
public class PhoneLayoutInflater extends LayoutInflater {
    private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
    };
    
    public PhoneLayoutInflater(Context context) {
        super(context);
    }
    
    protected PhoneLayoutInflater(LayoutInflater original, Context newContext) {
        super(original, newContext);
    }
    
    /** Override onCreateView to instantiate names that correspond to the
        widgets known to the Widget factory. If we don't find a match,
        call through to our super class.
    */
    @Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        for (String prefix : sClassPrefixList) {
            try {
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
                // In this case we want to let the base class take a crack
                // at it.
            }
        }

        return super.onCreateView(name, attrs);
    }
    
    public LayoutInflater cloneInContext(Context newContext) {
        return new PhoneLayoutInflater(this, newContext);
    }
}
```

LayoutInflater最常使用的方法
---

在Android中LayoutInflater中最常使用的情况基本都是调用inflate方法用来构造View对象。

```java
public abstract class LayoutInflater {
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root) {
        return inflate(parser, root, root != null);
    }
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
    /**
     * @param parser xml数据结构
     * @param root 一个可依附的rootview
     * @param attachToRoot 是否将parser解析生产的View添加在root上
     */
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
            //当前上下文环境
            final Context inflaterContext = mContext;
            //所有的属性集合获取类
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            //根节点
            View result = root;

            try {
                // Look for the root node.寻找根节点
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
                //找不到根节点抛出异常
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }
                //merge标签解析
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    //递归调用，添加root的孩子节点
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml。根据当前的attrs和xml创建一个xml根view
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied。构造LayoutParams
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.递归调用，添加temp的孩子节点
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        //将xml解析出来的viewgroup添加在root的根下
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (Exception e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                                + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }

            Trace.traceEnd(Trace.TRACE_TAG_VIEW);

            return result;
        }
    }
}
```

这四个重载的inflate方法最终都是通过`inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) `进行实现的。

> LayoutInflater的使用中重点关注inflate方法的参数含义：
>- inflate(xmlId, null); 只创建temp的View，然后直接返回temp。
>- inflate(xmlId, parent); 创建temp的View，然后执行root.addView(temp, params);最后返回root。
>- inflate(xmlId, parent, false); 创建temp的View，然后执行temp.setLayoutParams(params);然后再返回temp。
>- inflate(xmlId, parent, true); 创建temp的View，然后执行root.addView(temp, params);最后返回root。
>- inflate(xmlId, null, false); 只创建temp的View，然后直接返回temp。
>- inflate(xmlId, null, true); 只创建temp的View，然后直接返回temp。

LayoutInflater解析视图xml
---

- xml视图树解析

递归执行rInflate生产View并添加给父容器

```java
public abstract class LayoutInflater {
    /**
     * 将parser解析器中包含的view结合属性标签attrs生产view添加在parent容器中
     * @param parser xml解析器
     * @param parent 父容器
     * @param attrs  属性标签集合
     * @param finishInflate 生产view之后是否执行父容器的onFinishInflate方法。
     */
    final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }

    void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;

        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {  //requestFocus标签解析
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) { //tag标签解析
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) { //include标签解析
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) { //merge标签解析
                throw new InflateException("<merge /> must be the root element");
            } else {
                //View标签解析
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                //View所在容器（ViewGroup）的属性解析
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                //循环遍历xml的子节点
                rInflateChildren(parser, view, attrs, true);
                //将解析出的view和其对于的属性参数添加在父容器中
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
}
```

- 单个View布局的解析

调用createViewFromTag，设置View的Theme属性。再调用CreateView方法创建view


```java
public abstract class LayoutInflater {
    protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
        return createView(name, "android.view.", attrs);
    }

    protected View onCreateView(View parent, String name, AttributeSet attrs)
        throws ClassNotFoundException {
        return onCreateView(name, attrs);
    }

    /**
     * 将parser解析器中包含的view结合属性标签attrs生产view添加在parent容器中
     * @param parent 父容器
     * @param name view名称
     * @param context 上下文环境
     * @param attrs  属性标签集合
     */
    private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
        return createViewFromTag(parent, name, context, attrs, false);
    }

    View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        // Apply a theme wrapper, if allowed and one is specified.//应用theme
        if (!ignoreThemeAttr) {
            final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
            final int themeResId = ta.getResourceId(0, 0);
            if (themeResId != 0) {
                context = new ContextThemeWrapper(context, themeResId);
            }
            ta.recycle();
        }

        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

        try {
            View view;
            /*************************start Factory*/
            //使用LayoutInflater的Factory，对View进行修改
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }
            /*************************end Factory*/

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        //创建Android原生的View（android.view包下面的view）
                        view = onCreateView(parent, name, attrs);
                    } else {
                        //创建自定义View或者依赖包中的View（xml中声明的是全路径）
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } catch (InflateException e) {
            throw e;

        } catch (ClassNotFoundException e) {
            final InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;

        } catch (Exception e) {
            final InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;
        }
    }

    public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        Class<? extends View> clazz = null;

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);
            //判断View的构造是否进行缓存
            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                
                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                //static final Class<?>[] mConstructorSignature = new Class[] {Context.class, AttributeSet.class};
                constructor = clazz.getConstructor(mConstructorSignature);
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) : name).asSubclass(View.class);
                        
                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
            }
            
            Object[] args = mConstructorArgs;
            args[1] = attrs;
            constructor.setAccessible(true);
            //读取View的构造函数，传入context、attrs作为参数。View(Context context, AttributeSet attrs)；
            final View view = constructor.newInstance(args);
            //处理ViewStub标签
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            return view;

        } catch (NoSuchMethodException e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class "
                    + (prefix != null ? (prefix + name) : name));
            ie.initCause(e);
            throw ie;

        } catch (ClassCastException e) {
            // If loaded class is not a View subclass
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Class is not a View "
                    + (prefix != null ? (prefix + name) : name));
            ie.initCause(e);
            throw ie;
        } catch (ClassNotFoundException e) {
            // If loadClass fails, we should propagate the exception.
            throw e;
        } catch (Exception e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class "
                    + (clazz == null ? "<unknown>" : clazz.getName()));
            ie.initCause(e);
            throw ie;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
}
```

LayoutInflater创建View的总结
---
>- 在inflate方法中，通过Resource.getLayout(resource)生产XmlResourceParser对象；
>- 利用该对象实例生产XmlPullAttributes以便于xml标签中的属性。然后将这个两个对象传递到rInflate方法中，解析layout对应的xml文件；
>- 接着将（父容器、xml中View的名称、属性标签）传递给createViewFromTag方法创建对应的View；
>- 在createViewFromTag方法中执行LayoutInflater.Factory或者LayoutInflater的createView方法。
>- 在createView方法中我们已知View的类名和View的属性标签集合，通过Java反射执行View的构造方法创建View对象。这也就是为什么我们在自定义View的时候必须复写View的构造函数View(Context context, AttributeSet attrs)；

LayoutInflater.Factory简介
---
> LayoutInflater.Factory这个类在我们开发的过程中很少越到。但是我们在查看LayoutInflater解析View源码的过程中可以看到如果LayoutInflater中有mFactory这个实例那么可以通过mFactory创建View,同时也能修改入参AttributeSet属性值。

```java
public abstract class LayoutInflater {
    /***部分代码省略****/
    public interface Factory {
        public View onCreateView(String name, Context context, AttributeSet attrs);
    }

    public interface Factory2 extends Factory {
        public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
    }
    /***部分代码省略****/
}
```

- LayoutInflater有两个工厂类，Factory和Factory2，区别只是在于Factory2可以传入父容器作为参数。

```java
public abstract class LayoutInflater {
    /***部分代码省略****/
    public void setFactory(Factory factory) {
        if (mFactorySet) {
            throw new IllegalStateException("A factory has already been set on this LayoutInflater");
        }
        if (factory == null) {
            throw new NullPointerException("Given factory can not be null");
        }
        mFactorySet = true;
        if (mFactory == null) {
            mFactory = factory;
        } else {
            mFactory = new FactoryMerger(factory, null, mFactory, mFactory2);
        }
    }

    public void setFactory2(Factory2 factory) {
        if (mFactorySet) {
            throw new IllegalStateException("A factory has already been set on this LayoutInflater");
        }
        if (factory == null) {
            throw new NullPointerException("Given factory can not be null");
        }
        mFactorySet = true;
        if (mFactory == null) {
            mFactory = mFactory2 = factory;
        } else {
            mFactory = mFactory2 = new FactoryMerger(factory, factory, mFactory, mFactory2);
        }
    }
    /***部分代码省略****/
}
```

这两个方法的功能基本是一致的，setFactory2是在Android3.0之后以后引入的，所以我们要根据SDK的版本去选择调用上述方法。

在supportv4下边也有LayoutInflaterCompat可以做相同的操作。

```java
public class LayoutInflaterCompat {
    /***部分代码省略****/
    static final LayoutInflaterCompatImpl IMPL;
    static {
        final int version = Build.VERSION.SDK_INT;
        if (version >= 21) {
            IMPL = new LayoutInflaterCompatImplV21();
        } else if (version >= 11) {
            IMPL = new LayoutInflaterCompatImplV11();
        } else {
            IMPL = new LayoutInflaterCompatImplBase();
        }
    }
    private LayoutInflaterCompat() {
    }
    public static void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
        IMPL.setFactory(inflater, factory);
    }
}
```

LayoutInflater.Factory的使用
---
找到当前Activity中的id=R.id.text的TextView将其替换为Button，并修改BackgroundColor。

```java
public class MainActivity extends Activity {
    final String TAG = "MainActivity";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LayoutInflater.from(this).setFactory(new LayoutInflater.Factory() {

            @Override
            public View onCreateView(String name, Context context, AttributeSet attrs) {
                if ("TextView".equals(name)) {
                    Log.e(TAG, "name = " + name);
                    int n = attrs.getAttributeCount();
                    //打印所有属性标签
                    for (int i = 0; i < n; i++) {
                        Log.e(TAG, attrs.getAttributeName(i) + " , " + attrs.getAttributeValue(i));
                    }
                    for (int i = 0; i < n; i++) {
                        if (attrs.getAttributeName(i).equals("id")) {
                            String attributeValue = attrs.getAttributeValue(i);
                            String id = attributeValue.substring(1, attributeValue.length());
                            if (R.id.text == Integer.valueOf(id)) {
                                Button button = new Button(context, attrs);
                                button.setBackgroundColor(Color.RED);
                                return button;
                            }
                        }
                    }
                }
                return null;
            }
        });
        setContentView(R.layout.activity_main);
    }
}
```

Console输出：

```java
MainActivity: name = TextView
MainActivity: id , @2131492944
MainActivity: layout_width , -2
MainActivity: layout_height , -2
MainActivity: text , Hello World!
```

是不是发现LayoutInflater的Factory功能很好很强大。

> 这里提一个问题，如果把上面代码中的MainActivity的父类修改为AppCompatActivity会怎么样呢？我们试着运行一下。

```java
java.lang.RuntimeException: Unable to start activity
ComponentInfo{com.example.tzx.dexload/com.example.tzx.dexload.MainActivity}: 
java.lang.IllegalStateException: A factory has already been set on this LayoutInflater
```

程序运行报错:`A factory has already been set on this LayoutInflater`。这个是在执行LayoutInflater的setFactory方法时抛出的异常。因为mFactorySet=true。。。。这个时候我们发现LayoutInflater的Factory已经被设置过了。具体是在哪里设置的呢？我们看看下边`LayoutInflater.Factory在Android源码中的使用`部分内容。

LayoutInflater.Factory在Android源码中的使用
---
在我们开发过程是很少使用到LayoutInflater.Factory，但是Android在supportv7中就使用，我们来学习一下。

在AppComPatActivity中的onCreate就进行了LayoutInflater.Factory的设置。

```java
public class AppCompatActivity extends FragmentActivity implements AppCompatCallback,
    TaskStackBuilder.SupportParentable, ActionBarDrawerToggle.DelegateProvider {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        getDelegate().installViewFactory();
        getDelegate().onCreate(savedInstanceState);
        super.onCreate(savedInstanceState);
    }

    public AppCompatDelegate getDelegate() {
        if (mDelegate == null) {
            mDelegate = AppCompatDelegate.create(this, this);
        }
        return mDelegate;
    }
}
```

- 根据不通的sdk版本做适配

```java
public abstract class AppCompatDelegate {
    public static AppCompatDelegate create(Activity activity, AppCompatCallback callback) {
        return create(activity, activity.getWindow(), callback);
    }
    /***部分代码省略****/

    //对于不同的版本做适配
    private static AppCompatDelegate create(Context context, Window window,
        AppCompatCallback callback) {
        final int sdk = Build.VERSION.SDK_INT;
        if (sdk >= 23) {
            return new AppCompatDelegateImplV23(context, window, callback);
        } else if (sdk >= 14) {
            return new AppCompatDelegateImplV14(context, window, callback);
        } else if (sdk >= 11) {
            return new AppCompatDelegateImplV11(context, window, callback);
        } else {
            return new AppCompatDelegateImplV7(context, window, callback);
        }
    }
}
```

LayoutInflaterFactory的实现类，以及触发LayoutInflater.setFactory的调用。

```java
class AppCompatDelegateImplV7 extends AppCompatDelegateImplBase
    implements MenuBuilder.Callback, LayoutInflaterFactory {
    /***部分代码省略****/
    @Override
    public void installViewFactory() {
        LayoutInflater layoutInflater = LayoutInflater.from(mContext);
        if (layoutInflater.getFactory() == null) {
            //进行setFactory的设置
            LayoutInflaterCompat.setFactory(layoutInflater, this);
        } else {
            Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                    + " so we can not install AppCompat's");
        }
    }
    /***部分代码省略****/
    @Override
    public View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs) {
        final boolean isPre21 = Build.VERSION.SDK_INT < 21;

        if (mAppCompatViewInflater == null) {
            //具体的实现类，下文会有讲到
            mAppCompatViewInflater = new AppCompatViewInflater();
        }

        // We only want the View to inherit it's context if we're running pre-v21
        final boolean inheritContext = isPre21 && mSubDecorInstalled
                && shouldInheritContext((ViewParent) parent);

        return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext,
                isPre21, /* Only read android:theme pre-L (L+ handles this anyway) */
                true /* Read read app:theme as a fallback at all times for legacy reasons */
        );
    }
}
```

- 根据不同的版本找到LayoutInflater的包装类

```java
public final class LayoutInflaterCompat {
    
    interface LayoutInflaterCompatImpl {
        public void setFactory(LayoutInflater layoutInflater, LayoutInflaterFactory factory);
        public LayoutInflaterFactory getFactory(LayoutInflater layoutInflater);
    }

    static class LayoutInflaterCompatImplBase implements LayoutInflaterCompatImpl {
        @Override
        public void setFactory(LayoutInflater layoutInflater, LayoutInflaterFactory factory) {
            LayoutInflaterCompatBase.setFactory(layoutInflater, factory);
        }

        @Override
        public LayoutInflaterFactory getFactory(LayoutInflater layoutInflater) {
            return LayoutInflaterCompatBase.getFactory(layoutInflater);
        }
    }

    static final LayoutInflaterCompatImpl IMPL;
    static {
        final int version = Build.VERSION.SDK_INT;
        if (version >= 21) {
            IMPL = new LayoutInflaterCompatImplV21();
        } else if (version >= 11) {
            IMPL = new LayoutInflaterCompatImplV11();
        } else {
            IMPL = new LayoutInflaterCompatImplBase();
        }
    }

    /***部分代码省略****/

    private LayoutInflaterCompat() {
    }

    public static void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
        IMPL.setFactory(inflater, factory);
    }

    public static LayoutInflaterFactory getFactory(LayoutInflater inflater) {
        return IMPL.getFactory(inflater);
    }
}
```

- FactoryWrapper类通过调用LayoutInflaterFactory的onCreateView方法，实现了LayoutInflater.Factory接口。最终调用了LayoutInflater的setFactory方法，使得在LayoutInflater.createViewFromTag中创建View的时候通过Factory进行创建。

```java
public interface LayoutInflaterFactory {
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
}

class LayoutInflaterCompatBase {
    
    static class FactoryWrapper implements LayoutInflater.Factory {

        final LayoutInflaterFactory mDelegateFactory;

        FactoryWrapper(LayoutInflaterFactory delegateFactory) {
            mDelegateFactory = delegateFactory;
        }

        @Override
        public View onCreateView(String name, Context context, AttributeSet attrs) {
            //调用LayoutInflaterFactory实现类的onCreateView(null, name, context, attrs)方法
            return mDelegateFactory.onCreateView(null, name, context, attrs);
        }

        public String toString() {
            return getClass().getName() + "{" + mDelegateFactory + "}";
        }
    }

    static void setFactory(LayoutInflater inflater, LayoutInflaterFactory factory) {
        //最终调用了LayoutInflater的setFactory方法，对Factory进行设置
        inflater.setFactory(factory != null ? new FactoryWrapper(factory) : null);
    }

    static LayoutInflaterFactory getFactory(LayoutInflater inflater) {
        LayoutInflater.Factory factory = inflater.getFactory();
        if (factory instanceof FactoryWrapper) {
            return ((FactoryWrapper) factory).mDelegateFactory;
        }
        return null;
    }

}
```

- 在LayoutInflaterFactory的实现类之一AppCompatDelegateImplV7中，找到了setFactory的实际使用意义实际意思。
>- 在LayoutInflater.createViewFromTag方法中调用` Factory.onCreateView(name, context, attrs) `方法
>- Factory的实现类FactoryWrapper中，调用`LayoutInflaterFactory的onCreateView(null, name, context, attrs)`方法

```java
class AppCompatViewInflater {
    /***部分代码省略****/
    public final View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs, boolean inheritContext,
        boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
        final Context originalContext = context;

        // We can emulate Lollipop's android:theme attribute propagating down the view hierarchy
        // by using the parent's context
        if (inheritContext && parent != null) {
            context = parent.getContext();
        }
        if (readAndroidTheme || readAppTheme) {
            // We then apply the theme on the context, if specified
            context = themifyContext(context, attrs, readAndroidTheme, readAppTheme);
        }
        if (wrapContext) {
            context = TintContextWrapper.wrap(context);
        }

        View view = null;

        // We need to 'inject' our tint aware Views in place of the standard framework versions
        switch (name) {
            case "TextView":
                view = new AppCompatTextView(context, attrs);
                break;
            case "ImageView":
                view = new AppCompatImageView(context, attrs);
                break;
            case "Button":
                view = new AppCompatButton(context, attrs);
                break;
            case "EditText":
                view = new AppCompatEditText(context, attrs);
                break;
            case "Spinner":
                view = new AppCompatSpinner(context, attrs);
                break;
            case "ImageButton":
                view = new AppCompatImageButton(context, attrs);
                break;
            case "CheckBox":
                view = new AppCompatCheckBox(context, attrs);
                break;
            case "RadioButton":
                view = new AppCompatRadioButton(context, attrs);
                break;
            case "CheckedTextView":
                view = new AppCompatCheckedTextView(context, attrs);
                break;
            case "AutoCompleteTextView":
                view = new AppCompatAutoCompleteTextView(context, attrs);
                break;
            case "MultiAutoCompleteTextView":
                view = new AppCompatMultiAutoCompleteTextView(context, attrs);
                break;
            case "RatingBar":
                view = new AppCompatRatingBar(context, attrs);
                break;
            case "SeekBar":
                view = new AppCompatSeekBar(context, attrs);
                break;
        }

        if (view == null && originalContext != context) {
            // If the original context does not equal our themed context, then we need to manually
            // inflate it using the name so that android:theme takes effect.
            view = createViewFromTag(context, name, attrs);
        }

        if (view != null) {
            // If we have created a view, check it's android:onClick
            checkOnClickListener(view, attrs);
        }

        return view;
        }
}
```

- AppCompatViewInflater作为LayoutInflaterFactory的的onCreateView方法的最终实现类，通过createView方法替换了一些我们想要自己替换的View。比如：

|原始View|实际创建的View|
| :-------- | :-------- |
| TextView | AppCompatTextView |
| ImageView| AppCompatImageView|
| Button| AppCompatButton|
|……|……|

在appcompat使用自定义的LayoutInflater.Factory
---
这里我们有两种书写方式：

- 继续使用 LayoutInflater.from(this).setFactory 方法。

```java
LayoutInflater.from(this).setFactory(new LayoutInflater.Factory() {
    @Override
    public View onCreateView(String name, Context context, AttributeSet attrs) {
        AppCompatDelegate delegate = getDelegate();
        //调用AppCompatDelegate的createView方法将第一个参数设置为null
        View view = delegate.createView(null, name, context, attrs);
        if ("TextView".equals(name)) {
            Log.e(TAG, "name = " + name);
            int n = attrs.getAttributeCount();
            for (int i = 0; i < n; i++) {
                Log.e(TAG, attrs.getAttributeName(i) + " , " + attrs.getAttributeValue(i));
            }
            for (int i = 0; i < n; i++) {
                if (attrs.getAttributeName(i).equals("id")) {
                    String attributeValue = attrs.getAttributeValue(i);
                    String id = attributeValue.substring(1, attributeValue.length());
                    if (R.id.text == Integer.valueOf(id)) {
                        Button button = new Button(context, attrs);
                        button.setBackgroundColor(Color.RED);
                        button.setAllCaps(false);
                        return button;
                    }

                }
            }
        }
        return view;
    }
});
```

- 使用LayoutInflaterCompat.setFactory方法

```java
LayoutInflaterCompat.setFactory(LayoutInflater.from(this), new LayoutInflaterFactory() {
   @Override
   public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        //appcompat 创建view代码
        AppCompatDelegate delegate = getDelegate();
        View view = delegate.createView(parent, name, context, attrs);
        if ("TextView".equals(name)) {
            Log.e(TAG, "name = " + name);
            int n = attrs.getAttributeCount();
            for (int i = 0; i < n; i++) {
                Log.e(TAG, attrs.getAttributeName(i) + " , " + attrs.getAttributeValue(i));
            }
            for (int i = 0; i < n; i++) {
                if (attrs.getAttributeName(i).equals("id")) {
                    String attributeValue = attrs.getAttributeValue(i);
                    String id = attributeValue.substring(1, attributeValue.length());
                    if (R.id.text == Integer.valueOf(id)) {
                        Button button = new Button(context, attrs);
                        button.setBackgroundColor(Color.RED);
                        button.setAllCaps(false);
                        return button;
                    }

                }
            }
        }
        return view;
   }
});
```

两种写法的原理是相同的，因为上面讲述的LayoutInflater.Factory的实现类FactoryWrapper实现onCreateView方法的时候调用的AppCompatDelegate.onCreateView的时候第一个参数传递的值就是null。

Android应用中的换肤（夜间模式）是不是也利用的是LayoutInflater.Factory原理实现的呢，我们一起期待下一篇关于Android换肤文章。

参考文章：
[Android 探究 LayoutInflater setFactory](http://blog.csdn.net/lmj623565791/article/details/51503977)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>