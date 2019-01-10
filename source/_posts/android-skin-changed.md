---
title: Android换肤原理和Android-Skin-Loader框架解析
date: 2017-11-27 11:54:00
tags: [Factory,inflate]
categories: Android
description: "Android换肤技术已经是很久之前就已经被成熟使用的技术了，然而我最近才在学习和接触热修复的时候才看到。在看了一些换肤的方法之后，并且对市面上比较认可的Android-Skin-Loader换肤框架的源码进行了分析总结。再次记录一下祭奠自己逝去的时间。"
---

<center>![换皮肤啦](http://upload-images.jianshu.io/upload_images/1319879-f50a325cf078154c.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)</center>

前言
===
Android换肤技术已经是很久之前就已经被成熟使用的技术了，然而我最近才在学习和接触热修复的时候才看到。在看了一些换肤的方法之后，并且对市面上比较认可的Android-Skin-Loader换肤框架的源码进行了分析总结。再次记录一下祭奠自己逝去的时间。

换肤介绍
===
换肤本质上是对资源的一中替换包括、字体、颜色、背景、图片、大小等等。当然这些我们都有成熟的api可以通过控制代码逻辑做到。比如View的修改背景颜色`setBackgroundColor`,TextView的`setTextSize`修改字体等等。但是作为程序员我们怎么能忍受对每个页面的每个元素一个行行代码做换肤处理呢？我们需要用最少的代码实现最容易维护和使用效果完美（动态切换，及时生效）的换肤框架。

换肤方式一：切换使用主题Theme
---
> 使用相同的资源id，但在不同的Theme下边自定义不同的资源。我们通过主动切换到不同的Theme从而切换界面元素创建时使用的资源。这种方案的代码量不多发，而且有个很明显的缺点不支持已经创建界面的换肤，必须重新加载界面元素。[GitHub Demo](https://github.com/stven0king/MultipleTheme.git)


换肤方式二：加载资源包
---
> 加载资源包是各种应用程序都在使用的换肤方法，例如我们最常用的输入法皮肤、浏览器皮肤等等。我们可以将皮肤的资源文件放入安装包内部，也可以进行下载缓存到磁盘上。Android的应用程序可以使用这种方式进行换肤。GitHub上面有一个start非常高的换肤框架[Android-Skin-Loader](https://github.com/stven0king/Android-Skin-Loader.git) 就是通过加载资源包对app进行换肤。对这个框架的分析这个也是这篇文章主要的讲述内容。

对比一下发现切换Theme可以进行小幅度的换肤设置（比如某个自定义组件的主题），而如果我们想要对整个app做主题切换那么通过加载资源包的这种方式目前应该说是比较好的了。

Android换肤知识点
===

换肤相应的API
---
我们先来看一下Android提供的一些基本的api，通过使用这些api可以在App内部进行资源对象的替换。

```java
public class Resources {
    public String getString(int id) throws NotFoundException {
        CharSequence res = mAssets.getResourceText(id);
        if (res != null) {
            return res;
        }
        throw new NotFoundException("String resource ID #0x"
                                    + Integer.toHexString(id));
    }
    public Drawable getDrawable(int id) throws NotFoundException {
        /********部分代码省略*******/
    }
    public int getColor(int id) throws NotFoundException {{
        /********部分代码省略*******/
    }
    /********部分代码省略*******/
}
```

这个是我们常用的Resources类的api，我们通常可以使用在资源文件中定义的`@+id`String类型，然后在编译出的R.java中对应的资源文件生产的id（int类型），从而通过这个id（int类型）调用Resources提供的这些api获取到对应的资源对象。这个在同一个app下没有任何问题，但是在皮肤包中我们怎么获取这个id值呢。

```java
public class Resources {
    /********部分代码省略*******/
    /**
     * 通过给的资源名称返回一个资源的标识id。
     * @param name 描述资源的名称
     * @param defType 资源的类型
     * @param defPackage 包名
     * 
     * @return 返回资源id，0标识未找到该资源
     */
    public int getIdentifier(String name, String defType, String defPackage) {
        if (name == null) {
            throw new NullPointerException("name is null");
        }
        try {
            return Integer.parseInt(name);
        } catch (Exception e) {
            // Ignore
        }
        return mAssets.getResourceIdentifier(name, defType, defPackage);
    }
}
```

Resources提供了可以通过`@+id`、Type、PackageName这三个参数就可以在AssetManager中寻找相应的PackageName中有没有Type类型并且id值都能与参数对应上的id，进行返回。然后我们可以通过这个id再调用Resource的获取资源的api就可以得到相应的资源。

> 这里我们需要注意的一点是`getIdentifier(String name, String defType, String defPackage)`方法和`getString(int id)`方法所调用Resources对象的mAssets对象必须是同一个，并且包含有PackageName这个资源包。

AssetManager构造
---
怎么构造一个包含特定packageName资源的AssetManager对象实例呢？

```java
public final class AssetManager implements AutoCloseable {
    /********部分代码省略*******/
    /**
     * Create a new AssetManager containing only the basic system assets.
     * Applications will not generally use this method, instead retrieving the
     * appropriate asset manager with {@link Resources#getAssets}.    Not for
     * use by applications.
     * {@hide}
     */
    public AssetManager() {
        synchronized (this) {
            if (DEBUG_REFS) {
                mNumRefs = 0;
                incRefsLocked(this.hashCode());
            }
            init(false);
            if (localLOGV) Log.v(TAG, "New asset manager: " + this);
            ensureSystemAssets();
        }
    }
```

从AssetManager的构造函数来看有`{@hide}`的朱姐，所以在其他类里面是直接创建AssetManager实例。但是不要忘记Java中还有反射机制可以创建类对象。

```java
AssetManager assetManager = AssetManager.class.newInstance();
```

让创建的assetManager包含特定的PackageName的资源信息，怎么办？我们在AssetManager中找到相应的api可以调用。

```java
public final class AssetManager implements AutoCloseable {
    /********部分代码省略*******/
    /**
     * Add an additional set of assets to the asset manager.  This can be
     * either a directory or ZIP file.  Not for use by applications.  Returns
     * the cookie of the added asset, or 0 on failure.
     * {@hide}
     */
    public final int addAssetPath(String path) {
        synchronized (this) {
            int res = addAssetPathNative(path);
            if (mStringBlocks != null) {
                makeStringBlocks(mStringBlocks);
            }
            return res;
        }
    }
}
```

同样改方法也不支持外部调用，我们只能通过反射的方法来调用。

```java
/**
 * apk路径
 */
String apkPath = Environment.getExternalStorageDirectory()+"/skin.apk";
AssetManager assetManager = null;
try {
    AssetManager assetManager = AssetManager.class.newInstance();
    AssetManager.class.getDeclaredMethod("addAssetPath", String.class).invoke(assetManager, apkPath);
} catch (Throwable th) {
    th.printStackTrace();
}
```

至此我们可以构造属于自己换肤的Resources了。

换肤Resources构造
---

```java
public Resources getSkinResources(Context context){
    /**
     * 插件apk路径
     */
    String apkPath = Environment.getExternalStorageDirectory()+"/skin.apk";
    AssetManager assetManager = null;
    try {
        AssetManager assetManager = AssetManager.class.newInstance();
        AssetManager.class.getDeclaredMethod("addAssetPath", String.class).invoke(assetManager, apkPath);
    } catch (Throwable th) {
        th.printStackTrace();
    }
    return new Resources(assetManager, context.getResources().getDisplayMetrics(), context.getResources().getConfiguration());
}
```

使用资源包中的资源换肤
---
我们将上述所有的代码组合在一起就可以实现，使用资源包中的资源对app进行换肤。

```java
public Resources getSkinResources(Context context){
    /**
     * 插件apk路径
     */
    String apkPath = Environment.getExternalStorageDirectory()+"/skin.apk";
    AssetManager assetManager = null;
    try {
        AssetManager assetManager = AssetManager.class.newInstance();
        AssetManager.class.getDeclaredMethod("addAssetPath", String.class).invoke(assetManager, apkPath);
    } catch (Throwable th) {
        th.printStackTrace();
    }
    return new Resources(assetManager, context.getResources().getDisplayMetrics(), context.getResources().getConfiguration());
}
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    ImageView imageView = (ImageView) findViewById(R.id.imageView);
    TextView textView = (TextView) findViewById(R.id.text);
    /**
     * 插件资源对象
     */
    Resources resources = getSkinResources(this);
    /**
     * 获取图片资源
     */
    Drawable drawable = resources.getDrawable(resources.getIdentifier("night_icon", "drawable","com.tzx.skin"));
    /**
     * 获取Color资源
     */
    int color = resources.getColor(resources.getIdentifier("night_color","color","com.tzx.skin"));

    imageView.setImageDrawable(drawable);
    textView.setText(text);

}
```

通过上述介绍，我们可以简单的对当前页面进行换肤了。但是想要做出一个一个成熟换肤框架那么仅仅这些还是不够的，提高一下我们的思维高度，如果我们在View创建的时候就直接使用皮肤资源包中的资源文件，那么这无疑就使换肤更加的简单已维护。

LayoutInflater.Factory
---
看过我前一篇[遇见LayoutInflater&Factory](http://dandanlove.com/2017/11/15/layoutinflater-factory/)文章的这部分可以省略掉.

很幸运Android给我们在View生产的时候做修改提供了法门。

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

我们可以给当前的页面的Window对象在创建的时候设置Factory，那么在Window中的View进行创建的时候就会先通过自己设置的Factory进行创建。Factory使用方式和相关注意事项请移位到[遇见LayoutInflater&Factory](http://dandanlove.com/2017/11/15/layoutinflater-factory/)，关于Factory的相关知识点尽在其中。

Android-Skin-Loader解析
===

初始化
---
- 初始化换肤框架，导入需要换肤的资源包（当前为一个apk文件，其中只有资源文件）。

```java
public class SkinApplication extends Application {
	public void onCreate() {
		super.onCreate();
		initSkinLoader();
	}
	/**
	 * Must call init first
	 */
	private void initSkinLoader() {
		SkinManager.getInstance().init(this);
		SkinManager.getInstance().load();
	}
}
```

构造换肤对象
---
- 导入需要换肤的资源包，并构造换肤的Resources实例。

```java
/** 
 * Load resources from apk in asyc task 
 * @param skinPackagePath path of skin apk 
 * @param callback callback to notify user 
 */
public void load(String skinPackagePath, final ILoaderListener callback) { 
    new AsyncTask<String, Void, Resources>() {
 
        protected void onPreExecute() {
            if (callback != null) {
                callback.onStart();
            }
        };
 
        @Override
        protected Resources doInBackground(String... params) {
            try {
                if (params.length == 1) {
                    String skinPkgPath = params[0];
                     
                    File file = new File(skinPkgPath); 
                    if(file == null || !file.exists()){
                        return null;
                    }
                     
                    PackageManager mPm = context.getPackageManager();
                    //检索程序外的一个安装包文件
                    PackageInfo mInfo = mPm.getPackageArchiveInfo(skinPkgPath, PackageManager.GET_ACTIVITIES);
                    //获取安装包报名
                    skinPackageName = mInfo.packageName;
                    //构建换肤的AssetManager实例
                    AssetManager assetManager = AssetManager.class.newInstance();
                    Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);
                    addAssetPath.invoke(assetManager, skinPkgPath);
                    //构建换肤的Resources实例
                    Resources superRes = context.getResources();
                    Resources skinResource = new Resources(assetManager,superRes.getDisplayMetrics(),superRes.getConfiguration());
                    //存储当前皮肤路径
                    SkinConfig.saveSkinPath(context, skinPkgPath);
                     
                    skinPath = skinPkgPath;
                    isDefaultSkin = false;
                    return skinResource;
                }
                return null;
            } catch (Exception e) {
                e.printStackTrace();
                return null;
            }
        };
 
        protected void onPostExecute(Resources result) {
            mResources = result;
 
            if (mResources != null) {
                if (callback != null) callback.onSuccess();
                //更新多有可换肤的界面
                notifySkinUpdate();
            }else{
                isDefaultSkin = true;
                if (callback != null) callback.onFailed();
            }
        };
 
    }.execute(skinPackagePath);
}
```
定义基类
---
- 换肤页面的基类的通用代码实现基本换肤功能。

```java
public class BaseFragmentActivity extends FragmentActivity implements ISkinUpdate, IDynamicNewView{
    
    /***部分代码省略****/
    
    //自定义LayoutInflater.Factory
    private SkinInflaterFactory mSkinInflaterFactory;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    
        try {
            //设置LayoutInflater的mFactorySet为true，表示还未设置mFactory,否则会抛出异常。
            Field field = LayoutInflater.class.getDeclaredField("mFactorySet");
            field.setAccessible(true);
            field.setBoolean(getLayoutInflater(), false);
            //设置LayoutInflater的MFactory
            mSkinInflaterFactory = new SkinInflaterFactory();
            getLayoutInflater().setFactory(mSkinInflaterFactory);

        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } 
        
    }

    @Override
    protected void onResume() {
        super.onResume();
        //注册皮肤管理对象
        SkinManager.getInstance().attach(this);
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        //反注册皮肤管理对象
        SkinManager.getInstance().detach(this);
    }
    /***部分代码省略****/
}
```

SkinInflaterFactory
---
- SkinInflaterFactory进行View的创建并对View进行换肤。

### 构造View ###

```java
public class SkinInflaterFactory implements Factory {
    /***部分代码省略****/
    public View onCreateView(String name, Context context, AttributeSet attrs) {
        //读取View的skin:enable属性，false为不需要换肤
        // if this is NOT enable to be skined , simplly skip it 
        boolean isSkinEnable = attrs.getAttributeBooleanValue(SkinConfig.NAMESPACE, SkinConfig.ATTR_SKIN_ENABLE, false);
        if (!isSkinEnable){
                return null;
        }
        //创建View
        View view = createView(context, name, attrs);
        if (view == null){
            return null;
        }
        //如果View创建成功，对View进行换肤
        parseSkinAttr(context, attrs, view);
        return view;
    }
    //创建View，类比可以查看LayoutInflater的createViewFromTag方法
    private View createView(Context context, String name, AttributeSet attrs) {
        View view = null;
        try {
            if (-1 == name.indexOf('.')){
                if ("View".equals(name)) {
                    view = LayoutInflater.from(context).createView(name, "android.view.", attrs);
                } 
                if (view == null) {
                    view = LayoutInflater.from(context).createView(name, "android.widget.", attrs);
                } 
                if (view == null) {
                    view = LayoutInflater.from(context).createView(name, "android.webkit.", attrs);
                } 
            }else {
                view = LayoutInflater.from(context).createView(name, null, attrs);
            }

            L.i("about to create " + name);

        } catch (Exception e) { 
            L.e("error while create 【" + name + "】 : " + e.getMessage());
            view = null;
        }
        return view;
    }
}
```

### 对生产的View进行换肤 ###

```java
public class SkinInflaterFactory implements Factory {
    //存储当前Activity中的需要换肤的View
    private List<SkinItem> mSkinItems = new ArrayList<SkinItem>();
    /***部分代码省略****/
    private void parseSkinAttr(Context context, AttributeSet attrs, View view) {
        //当前View的所有属性标签
        List<SkinAttr> viewAttrs = new ArrayList<SkinAttr>();
        
        for (int i = 0; i < attrs.getAttributeCount(); i++){
            String attrName = attrs.getAttributeName(i);
            String attrValue = attrs.getAttributeValue(i);
            
            if(!AttrFactory.isSupportedAttr(attrName)){
                continue;
            }
            //过滤view属性标签中属性的value的值为引用类型
            if(attrValue.startsWith("@")){
                try {
                    int id = Integer.parseInt(attrValue.substring(1));
                    String entryName = context.getResources().getResourceEntryName(id);
                    String typeName = context.getResources().getResourceTypeName(id);
                    //构造SkinAttr实例,attrname,id,entryName,typeName
                    //属性的名称（background）、属性的id值（int类型），属性的id值（@+id，string类型），属性的值类型（color）
                    SkinAttr mSkinAttr = AttrFactory.get(attrName, id, entryName, typeName);
                    if (mSkinAttr != null) {
                        viewAttrs.add(mSkinAttr);
                    }
                } catch (NumberFormatException e) {
                    e.printStackTrace();
                } catch (NotFoundException e) {
                    e.printStackTrace();
                }
            }
        }
        //如果当前View需要换肤，那么添加在mSkinItems中
        if(!ListUtils.isEmpty(viewAttrs)){
            SkinItem skinItem = new SkinItem();
            skinItem.view = view;
            skinItem.attrs = viewAttrs;

            mSkinItems.add(skinItem);
            //是否是使用外部皮肤进行换肤
            if(SkinManager.getInstance().isExternalSkin()){
                skinItem.apply();
            }
        }
    }
}
```

资源获取
---

通过当前的资源id，找到对应的资源name。再从皮肤包中找到该资源name所对应的资源id。

```java
public class SkinManager implements ISkinLoader{
    /***部分代码省略****/
    public int getColor(int resId){
        int originColor = context.getResources().getColor(resId);
        //是否没有下载皮肤或者当前使用默认皮肤
        if(mResources == null || isDefaultSkin){
            return originColor;
        }
        //根据resId值获取对应的xml的的@+id的String类型的值
        String resName = context.getResources().getResourceEntryName(resId);
        //更具resName在皮肤包的mResources中获取对应的resId
        int trueResId = mResources.getIdentifier(resName, "color", skinPackageName);
        int trueColor = 0;
        try{
            //根据resId获取对应的资源value
            trueColor = mResources.getColor(trueResId);
        }catch(NotFoundException e){
            e.printStackTrace();
            trueColor = originColor;
        }
        
        return trueColor;
    }
    public Drawable getDrawable(int resId){...}
}
```

其他
---

除此之外再增加以下对于皮肤的管理api（下载、监听回调、应用、取消、异常处理、扩展模块等等）。

总结
===
换肤就是这么简单~！~！

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)