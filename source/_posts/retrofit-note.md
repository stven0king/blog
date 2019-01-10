---
title: Android网络之Retrofit2.0使用和解析
date: 2016-07-11 10:08:43
tags: [Retrofit2.0,Android网络请求框架]
categories: Android
description: "Retrofit+Rxjava+okhttp是时下比较受欢迎的网络请求框架，其源代码并不是很多，其底层网络通信时交由 OkHttp来完成的，但是Retrofit运用了大量的设计模式，代码逻辑很清晰。本文通过Retrofit2.0的使用讲述其实现原理"
---

# Android网络之Retrofit2.0使用和解析 #

## Retrofit2在项目中的使用 ##
### Android studio项目添加依赖 ###

``` java
compile 'com.squareup.retrofit2:retrofit:2.0.1'
```

### 项目中使用样例 ###
#### 定义HTTP API使用接口 ####

```java
public interface GitHubService {
	@GET("users/{user}/repos")
	 Call<List<Repo>> listRepos(@Path("user") String user);
}
```

>- 通过在接口上添加注解的方式来表示如何处理网络请求。
- Retrofit支持5中类型的注解：GET,POST,PUT,DELETE和HEAD.
- 可以使用不带参数的url`@GET("users/list")`，也可以使用带参数的url`@GET("users/list?sort=desc")`

#### 构造Retrofit实例 ####

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();
GitHubService service = retrofit.create(GitHubService.class);
```

#### 创建同步或异步HTTP请求到远程网络服务器 ####

```java
Call<List<Repo>> repos = service.listRepos("octocat");
```

#### 定制数据类型转换器 ####

>- Gson: com.squareup.retrofit2:converter-gson
- Jackson: com.squareup.retrofit2:converter-jackson
- Moshi: com.squareup.retrofit2:converter-moshi
- Protobuf: com.squareup.retrofit2:converter-protobuf
- Wire: com.squareup.retrofit2:converter-wire
- Simple XML: com.squareup.retrofit2:converter-simplexml
- Scalars (primitives, boxed, and String): com.squareup.retrofit2:converter-scalars

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(GsonConverterFactory.create())
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

### Retrofit使用扩展 ###
#### 自定义Gson类型转换器 ####

```java
/**
{
	"resultcode":0,
	"resultmsg":"请求成功",
	"result":{}
}
*/
public class Wrapper {
    public int resultcode ;
    public String resultmsg ;
    public Object result ;
    public static class JsonAdapter implements JsonDeserializer<Wrapper02> {
        @Override
        public Wrapper deserialize(JsonElement json, Type typeOfT, JsonDeserializationContext context) throws JsonParseException {
            try {
                String jsonRoot = json.getAsJsonObject().toString() ;
                Wrapper wrapper = new Wrapper() ;
                JSONObject jsobRespData = new JSONObject(jsonRoot) ;
                wrapper.resultcode = jsobRespData.getInt("resultcode") ;
                wrapper.resultmsg = jsobRespData.getString("resultmsg") ;
                wrapper.result = jsobRespData.get("result") ;
                return wrapper;
            } catch (JSONException e) {
                throw new JsonParseException(e) ;
            }
        }
    }

}
```

添加到Retrofit当中

```java
Gson gson = new GsonBuilder()
            .registerTypeAdapter(Wrapper.class, new Wrapper.JsonAdapter())
            .create() ;
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(GsonConverterFactory.create(gson))
    .build();

GitHubService service = retrofit.create(GitHubService.class);      
```

#### 请求时添加head信息 ####
在定义请求接口时添加：

```java
@Headers("Cache-Control: max-age=640000")
@GET("widget/list")
Call<List<Widget>> widgetList();

@Headers({
    "Accept: application/vnd.github.v3.full+json",
    "User-Agent: Retrofit-Sample-App"
})
@GET("users/{username}")
Call<User> getUser(@Path("username") String username);

@GET("user")
Call<User> getUser(@Header("Authorization") String authorization)
```

![Retrofit依赖](https://img-blog.csdn.net/20160708104109016)
如果所示在Retrofit2.0中只支持okhttp，所以另一种方法是在okhttp的拦截器中addheader。

## Retrofit2源码解析 ##
> Retrofit请求框架实现了高度的解耦，通过解析注解的得到的代理类生成http请求，然后将请求交给OkHttp。通过在Retrofit创建时生成的Converter再将OkHttp返回的数据进行类型转换得到自己需要的数据。现在Rxjava响应式编程已经广泛应用，在使用Retrofit时也会结合RxJava使编码更加简单、高效。

一张图简单描述一下Retrofit的工作原理：
![Retrofit工作原理](https://img-blog.csdn.net/20160708153944054)

### 定义网络请求接口 ###

```java
public interface GitHubService {
	 @GET("users/{user}/repos")
	 Call<List<Repo>> listRepos(@Path("user") String user);
}
```

### 创建Retrofit实例 ###

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    //支持RxJava
    .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
    .addConverterFactory(ByteArrayConverterFactory.create())
    .addConverterFactory(JSONObjectResponseConverterFactory.create())
    .addConverterFactory(StringResponseConverterFactory.create())
    //支持对象转json串
    .addConverterFactory(GsonConverterFactory.create())
    .build();
```

>- 设置BaseUrl
- 添加CallAdapterFactory
- 添加converterFactory
- 此时也可以设置自定义的okHttpclient

接下来我们看

```java
GitHubService service = retrofit.create(GitHubService.class);
```

### Retrofit.create方法的详细介绍 ###

```java
public <T> T create(final Class<T> service) {
	//判断定义的接口服务是否可用
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          //判断Android，IOS，java8
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            //如果是对象里的方法直接调用
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            /**
            * 对java8做兼容，android和ios值恒为false
            */
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //主要看这三行代码
            /**
            * 1、生成获取缓存中的method对应的ServiceMethod或者生产method对应的ServiceMethod
            * 2、生成OkHttpCall的实例
            * 3、根据生成的ServiceMethod实例中的callAdapter对象，调用callAdapter.adapt方法创建
            * 对应的Observable
            */
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

### Platfrom的获取 ###

![Platfrom](https://img-blog.csdn.net/20160902155113639)

```java
class Platform {
	//这个方法Android中为Plafrom默认的
	//Java8返回的是method.isDefault()，熟悉Java8的都知道这是Java8的新特性。。
	boolean isDefaultMethod(Method method) {
	    return false;
	}
	//这个方法Android中有自己默认的实现MainThreadExecutor
	Executor defaultCallbackExecutor() {
	    return null;
	}
	static class Android extends Platform {
		@Override 
		public Executor defaultCallbackExecutor() {
			return new MainThreadExecutor();
		}

		@Override 
		CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
			return new ExecutorCallAdapterFactory(callbackExecutor);
		}
		//Rx默认请求方式都是同步请求，所以我们在发出请求和请求结果回来的时候切换线程
		static class MainThreadExecutor implements Executor {
			private final Handler handler = new Handler(Looper.getMainLooper());
			@Override 
			public void execute(Runnable r) {
				handler.post(r);
			}
		}
	}
}
```


### ServiceMethod对象的生成 ###
先看一张我debug时候的ServiceMethod的图
![ServiceMethod](https://img-blog.csdn.net/20160902161408563)

#### ServiceMethod的获取 ####

```java
/**Retrofit.java
* 首先从serviceMethodCache缓存中获取，如果为null则创建
*/
ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

#### ServiceMethod的创建 ####

```java
final class ServiceMethod<T> {
	//部分代码省略
	ServiceMethod(Builder<T> builder) {
	    this.callFactory = builder.retrofit.callFactory();
	    this.callAdapter = builder.callAdapter;
	    this.baseUrl = builder.retrofit.baseUrl();
	    this.responseConverter = builder.responseConverter;
	    this.httpMethod = builder.httpMethod;
	    this.relativeUrl = builder.relativeUrl;
	    this.headers = builder.headers;
	    this.contentType = builder.contentType;
	    this.hasBody = builder.hasBody;
	    this.isFormEncoded = builder.isFormEncoded;
	    this.isMultipart = builder.isMultipart;
	    this.parameterHandlers = builder.parameterHandlers;
	}
	static final class Builder<T> {
		public Builder(Retrofit retrofit, Method method) {
		this.retrofit = retrofit;
		this.method = method;
		this.methodAnnotations = method.getAnnotations();
		this.parameterTypes = method.getGenericParameterTypes();
		this.parameterAnnotationsArray = method.getParameterAnnotations();
	}

		public ServiceMethod build() {
			//创建CallAdapter<?>
			callAdapter = createCallAdapter();
			responseType = callAdapter.responseType();
			if (responseType == Response.class || responseType == okhttp3.Response.class) {
			 throw methodError("'"
				 + Utils.getRawType(responseType).getName()
				 + "' is not a valid response body type. Did you mean ResponseBody?");
			}
			//创建Converter<ResponseBody, T>
			responseConverter = createResponseConverter();
			for (Annotation annotation : methodAnnotations) {
			 parseMethodAnnotation(annotation);
			}
			/********/
			return new ServiceMethod<>(this);
		}
		
		private CallAdapter<?> createCallAdapter() {
			Type returnType = method.getGenericReturnType();
			/*******/
			Annotation[] annotations = method.getAnnotations();
			try {
				//调用retrofit中的方法进行创建
				return retrofit.callAdapter(returnType, annotations);
			} catch (RuntimeException e) { // Wide exception range because factories are user code.
				throw methodError(e, "Unable to create call adapter for %s", returnType);
			}
		}
		private Converter<ResponseBody, T> createResponseConverter() {
			Annotation[] annotations = method.getAnnotations();
			try {
				//调用retrofit中的方法进行创建
				return retrofit.responseBodyConverter(responseType, annotations);
			} catch (RuntimeException e) { // Wide exception range because factories are user code.
				throw methodError(e, "Unable to create converter for %s", responseType);
			}
		}
	}
}
```

##### CallAdapter的创建 #####

```java
public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
	return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
  Annotation[] annotations) {
	checkNotNull(returnType, "returnType == null");
	checkNotNull(annotations, "annotations == null");
	int start = adapterFactories.indexOf(skipPast) + 1;
	for (int i = start, count = adapterFactories.size(); i < count; i++) {
		CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
		if (adapter != null) {
			return adapter;
		}
	}
	/********************/
}
```

在创建Retrofit的时候我们添加过addCallAdapterFactory(RxJavaCallAdapterFactory.create()),这是我们会调用RxJavaCallAdapterFactory.get方法获取CallAdapter，通过源代码我们可以找到其返回的是new SimpleCallAdapter(observableType, scheduler)。

##### Converter的创建 #####

```java
public <T> Converter<T, RequestBody> requestBodyConverter(Type type,
   Annotation[] parameterAnnotations, Annotation[] methodAnnotations) {
   return nextRequestBodyConverter(null, type, parameterAnnotations, methodAnnotations);
}

public <T> Converter<T, RequestBody> nextRequestBodyConverter(Converter.Factory skipPast,
	Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations) {
	checkNotNull(type, "type == null");
    checkNotNull(parameterAnnotations, "parameterAnnotations == null");
    checkNotNull(methodAnnotations, "methodAnnotations == null");
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
		Converter.Factory factory = converterFactories.get(i);
		Converter<?, RequestBody> converter =
			factory.requestBodyConverter(type, parameterAnnotations, methodAnnotations, this);
		if (converter != null) {
			//noinspection unchecked
			return (Converter<T, RequestBody>) converter;
		}
    }
    /******/
}
```

相同的在创建Retrofit的时候我们也添加过许多的ConverterFactory，在寻找相匹配的Converter时我们是通过遍历在寻找到第一个合适的Converter返回。什么叫做合适的Converter，即该ConverterFactory能产生出匹配服务接口注解和返回类型。

```java
public static final class Builder {
    private Platform platform;
    private okhttp3.Call.Factory callFactory;
    private HttpUrl baseUrl;
    private List<Converter.Factory> converterFactories = new ArrayList<>();
    private List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
    private Executor callbackExecutor;
    private boolean validateEagerly;

    Builder(Platform platform) {
      this.platform = platform;
      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      //自动回添加加默认的转化器
      converterFactories.add(new BuiltInConverters());
    }
    /****************/
    public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      //添加一个默认的适配器（Android、IOS、Java8）
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }

```

### OkHttpCall的创建 ###

```java
OkHttpCall(ServiceMethod<T> serviceMethod, Object[] args) {
   this.serviceMethod = serviceMethod;
   this.args = args;
}
```

### 网络请求 ###
#### 请求的准备 ####

`serviceMethod.callAdapter.adapt(okHttpCall)` 对应SimpleCallAdapter.adapt

```java
static final class SimpleCallAdapter implements CallAdapter<Observable<?>> {
    private final Type responseType;
    private final Scheduler scheduler;
    SimpleCallAdapter(Type responseType, Scheduler scheduler) {
		this.responseType = responseType;
		this.scheduler = scheduler;
    }
    @Override public Type responseType() {
		return responseType;
    }
    @Override public <R> Observable<R> adapt(Call<R> call) {
	    //创建请求的观察者，返回我们需要的Ovservable
		Observable<R> observable = Observable.create(new CallOnSubscribe<>(call)) //
          .lift(OperatorMapResponseToBodyOrError.<R>instance());
		if (scheduler != null) {
			return observable.subscribeOn(scheduler);
		}
		return observable;
    }
}
```

```java
static class Android extends Platform {
    @Override
    public Executor defaultCallbackExecutor() {
        return new MainThreadExecutor();
    }
    @Override
    CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
        return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    static class MainThreadExecutor implements Executor {
        private final Handler handler = new Handler(Looper.getMainLooper());
        @Override
        public void execute(Runnable r) {
            handler.post(r);
        }
    }
}
/*
* 因为默认的执行线程为主线程，所以我们要切换到subscribeOn(Schedulers.io())线程从而达到异步的目的。
* 然后通过observeOn(AndroidSchedulers.mainThread())将线程切回UI线程。
* 当Okhttp请求完数据并进行相应的convert之后，就可以在UI处理相应的逻辑。
*/
service.listRepos("octocat")
.observeOn(AndroidSchedulers.mainThread())
.subscribeOn(Schedulers.io())
.subscribe(list->{ 
	if(list!=null){      
	//TODO 取得数据后逻辑处理     
	}     
});
```
#### 请求的发起 ####
回到CallAdapt方法，创建Observable，而new CallOnSubscribe<>(call)生成了一个OnSubscribe（）的实例，而OnSubscribe继承自Action1，其只包含一个call()方法，而这个call是在CallOnSubscribe中实现： 

```java
static final class CallOnSubscribe<T> implements Observable.OnSubscribe<Response<T>> {
    private final Call<T> originalCall;
    CallOnSubscribe(Call<T> originalCall) {
	this.originalCall = originalCall;
    }
    @Override public void call(final Subscriber<? super Response<T>> subscriber) {
		// Since Call is a one-shot type, clone it for each new subscriber.
		Call<T> call = originalCall.clone();

		// Wrap the call in a helper which handles both unsubscription and backpressure.
		RequestArbiter<T> requestArbiter = new RequestArbiter<>(call, subscriber);
		subscriber.add(Subscriptions.create(requestArbiter));
		subscriber.setProducer(requestArbiter);
    }
}
```

首先clone了一份Call，然后生成了RequestArbiter，他继承自AtomicBoolean，实现了Subscription, Producer接口，Producer只有一个request方法；一般实现该接口的类，都会包含一个Subscriber对象和一个待处理的数据：

```java
static final class RequestArbiter<T> extends AtomicBoolean implements Action0,
    Producer {
    private final Call<T> call;
    private final Subscriber<?super Response<T>> subscriber;
    RequestArbiter(Call<T> call, Subscriber<?super Response<T>> subscriber) {
        this.call = call;
        this.subscriber = subscriber;
    }
    @Override
    public void request(long n) {
        if (n < 0) {
            throw new IllegalArgumentException("n < 0: " + n);
        }
        if (n == 0) {
            return; // Nothing to do when requesting 0.
        }
        if (!compareAndSet(false, true)) {
            return; // Request was already triggered.
        }
        try {
	        //进行网络请求
            Response<T> response = call.execute();
            if (!subscriber.isUnsubscribed()) {
                subscriber.onNext(response);
            }
        } catch (Throwable t) {
            Exceptions.throwIfFatal(t);
            if (!subscriber.isUnsubscribed()) {
                subscriber.onError(t);
            }
            return;
        }
        if (!subscriber.isUnsubscribed()) {
            subscriber.onCompleted();
        }
    }

    @Override
    public void call() {
        call.cancel();
    }
}
```

#### 请求的执行 ####

回顾创建Retrofit.create()代码中`serviceMethod.callAdapter.adapt(okHttpCall)`，所以上面的`call.execute()` 就是OkHttpCall.execute

```java
public Response<T> execute() throws IOException{
	okhttp3.Call call;
	synchronized (this) {
		if ( executed )
			throw new IllegalStateException( "Already executed." );
		executed = true;
		if ( creationFailure != null ){
			if ( creationFailure instanceof IOException ){
				throw (IOException) creationFailure;
			} else {
				throw (RuntimeException) creationFailure;
			}
		}
		call = rawCall;
		if ( call == null ){
			try {
				//获取okhttp实例
				call = rawCall = createRawCall();
			} catch ( IOException | RuntimeException e ) {
				creationFailure = e;
				throw e;
			}
		}
	}
	if ( canceled ){
		call.cancel();
	}
	//执行okhttp请求
	return(parseResponse( call.execute() ) );
}

private okhttp3.Call createRawCall() throws IOException{
	Request	request = serviceMethod.toRequest( args );
	//serviceMethod构造中this.callFactory = builder.retrofit.callFactory();
	okhttp3.Call call = serviceMethod.callFactory.newCall( request );
	if ( call == null ){
		throw new NullPointerException( "Call.Factory returned null." );
	}
	return(call);
}
```

#### 请求的OkHttpClient实例获取 ####

```java
public okhttp3.Call.Factory callFactory() {
    return callFactory;
}
public Builder callFactory(okhttp3.Call.Factory factory) {
	this.callFactory = checkNotNull(factory, "factory == null");
	return this;
}
//使用自定义OkHttpClient
public Builder client(OkHttpClient client) {
	return callFactory(checkNotNull(client, "client == null"));
}

public Retrofit build(){
	if ( baseUrl == null ){
		throw new IllegalStateException( "Base URL required." );
	}
	okhttp3.Call.Factory callFactory = this.callFactory;
	//没有自定义OkHttpClient，则会新创建一个
	if ( callFactory == null ){
		callFactory = new OkHttpClient();
	}
	Executor callbackExecutor = this.callbackExecutor;
	if ( callbackExecutor == null ){
		callbackExecutor = platform.defaultCallbackExecutor();
	}
	List<CallAdapter.Factory> adapterFactories = new ArrayList<>( this.adapterFactories );
	adapterFactories.add( platform.defaultCallAdapterFactory( callbackExecutor ) );
	List<Converter.Factory> converterFactories = new ArrayList<>( this.converterFactories );
	return(new Retrofit( callFactory, baseUrl, converterFactories, adapterFactories,
			     callbackExecutor, validateEagerly ) );
	}
}
```

#### 请求结果的处理 ####

```java
Response<T> parseResponse( okhttp3.Response rawResponse ) throws IOException{
	ResponseBody rawBody = rawResponse.body();
	rawResponse = rawResponse.newBuilder()
		      .body( new NoContentResponseBody( rawBody.contentType(), rawBody.contentLength() ) )
		      .build();
	int code = rawResponse.code();
	if ( code < 200 || code >= 300 ){
		try {
			ResponseBody bufferedBody = Utils.buffer( rawBody );
			return(Response.error( bufferedBody, rawResponse ) );
		} finally {
			rawBody.close();
		}
	}
	if ( code == 204 || code == 205 ){
		return(Response.success( null, rawResponse ) );
	}
	ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody( rawBody );
	try {
		//使用Converter将返回结果转化为接口返回的数据格式类型
		T body = serviceMethod.toResponse( catchingBody );
		//包装成Response并返回
		return(Response.success( body, rawResponse ) );
	} catch ( RuntimeException e ) {
		catchingBody.throwIfCaught();
		throw e;
	}
}
```

还记得创建Observable时 ` Observable<R> observable = Observable.create(new CallOnSubscribe<>(call)).lift(OperatorMapResponseToBodyOrError.<R>instance()); ` ，OperatorMapResponseToBodyOrError将包装的Response中的body取出来并进行发射。

## 总结 ##
现在随着Rxjava响应式编程越来越多的程序猿使用，自己也开始接触和使用。Retrofit+Rxjava+okhttp是时下比较受欢迎的网络请求框架，其源代码并不是很多，其底层网络通信时交由 OkHttp来完成的，但是Retrofit运用了大量的设计模式，代码逻辑很清晰，笔者以前用的是AsyncHttpClient作为app的网络请求框架，其源码也没自己的研究过。但看完Retrofit代码之后觉得收获很大，建议如果感兴趣可以抽时间仔细的阅读。

想阅读作者的更多文章，可以查看我的公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>