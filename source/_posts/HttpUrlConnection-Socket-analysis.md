---
title: Android网络之HttpUrlConnection和Socket关系解析
date: 2016-07-18 00:41:23
tags: [HttpUrlConnection,Socket,Android网络]
categories: Android
description: 多年以前Android的网络请求只有Apache开源的HttpClient和JDK的HttpUrlConnection，近几年随着OkHttp的流行Android在高版本的SDK中加入了OkHttp。但在Android官方文档中推荐使用HttpUrlConnection并且其会一直被维护，所以在学习Android网络相关的知识时我们队HttpUrlConnection要有足够的了解。。。。
---
多年以前Android的网络请求只有Apache开源的HttpClient和JDK的HttpUrlConnection，近几年随着OkHttp的流行Android在高版本的SDK中加入了OkHttp。但在Android官方文档中推荐使用HttpUrlConnection并且其会一直被维护，所以在学习Android网络相关的知识时我们队HttpUrlConnection要有足够的了解。。。。

前几天因为时间的关系只画了图 [HttpUrlConnection和Socket的关系图](http://blog.csdn.net/stven_king/article/details/51913649) ，本来说好的第二天续写，结果一直拖到了周末晚上。幸好时间还来的及，趁这短时间影响深刻，将自己解析代码过程记录下来。（PS：解析的过程有什么地方不明白的可以看看 [HttpUrlConnection和Socket的关系图](http://blog.csdn.net/stven_king/article/details/51913649) 图中讲出的过程和这次代码分析的过程是一样的，只不过代码讲述更加详细。***所有源码都是来自Android4.0.4***。有代码就有真相~！~！）

类结构图
-----------------------------------------------------------
先给大家展示一张相关类的结构图：
![HttpUrlConnection和Socket关系类图](http://img.blog.csdn.net/20160717213831370)

HttpUrlConnection 使用
-----------------------------------------------------------
在分析代码的时候我希望首相脑海中要有一个URL的请求过程。
这是我在网上摘的一个HttpUrlConnection请求小Demo：
```java
public class EsmTest {
    /**
     * 通过HttpURLConnection模拟post表单提交
     * @throws Exception
     */
    @Test
    public void sendEms() throws Exception {
        String wen = "MS2201828";
        String btnSearch = "EMS快递查询";
        URL url = new URL("http://www.kd185.com/ems.php");
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setRequestMethod("POST");// 提交模式
        // conn.setConnectTimeout(10000);//连接超时 单位毫秒
        // conn.setReadTimeout(2000);//读取超时 单位毫秒
        conn.setDoOutput(true);// 是否输入参数
        StringBuffer params = new StringBuffer();
        // 表单参数与get形式一样
        params.append("wen").append("=").append(wen).append("&")
              .append("btnSearch").append("=").append(btnSearch);
        byte[] bypes = params.toString().getBytes();
        conn.getOutputStream().write(bypes);// 输入参数
        InputStream inStream=conn.getInputStream();
        System.out.println(new String(StreamTool.readInputStream(inStream), "gbk"));
 
    }

    public void sendSms() throws Exception{
        String message="货已发到";
        message=URLEncoder.encode(message, "UTF-8");
        System.out.println(message);
        String path ="http://localhost:8083/DS_Trade/mobile/sim!add.do?message="+message;
        URL url =new URL(path);
        HttpURLConnection conn = (HttpURLConnection)url.openConnection();
        conn.setConnectTimeout(5*1000);
        conn.setRequestMethod("GET");
        InputStream inStream = conn.getInputStream();    
        byte[] data = StreamTool.readInputStream(inStream);
        String result=new String(data, "UTF-8");
        System.out.println(result);
    }
}

```

URL产生请求
-----------------------------------------------------------
```java
/*****************URL.java************************/
/**
 * 创建一个新的URL实例
 */
public URL(String spec) throws MalformedURLException {
	this((URL) null, spec, null);
}
public URL(URL context, String spec, URLStreamHandler handler) throws MalformedURLException {
	if (spec == null) {
		throw new MalformedURLException();
	}
	if (handler != null) {
		streamHandler = handler;
	}
	spec = spec.trim();
	//获取url的协议类型，http,https
	protocol = UrlUtils.getSchemePrefix(spec);
	//请求开始部分的位置
	int schemeSpecificPartStart = protocol != null ? (protocol.length() + 1) : 0;
	if (protocol != null && context != null && !protocol.equals(context.protocol)) {
		context = null;
	}
	if (context != null) {
		set(context.protocol, context.getHost(), context.getPort(), context.getAuthority(),
				context.getUserInfo(), context.getPath(), context.getQuery(),
				context.getRef());
		if (streamHandler == null) {
			streamHandler = context.streamHandler;
		}
	} else if (protocol == null) {
		throw new MalformedURLException("Protocol not found: " + spec);
	}
	//这里为重点，获取StreamHandler
	if (streamHandler == null) {
		setupStreamHandler();
		if (streamHandler == null) {
			throw new MalformedURLException("Unknown protocol: " + protocol);
		}
	}
	try {
		//对url的处理
		streamHandler.parseURL(this, spec, schemeSpecificPartStart, spec.length());
	} catch (Exception e) {
		throw new MalformedURLException(e.toString());
	}
}

void setupStreamHandler() {
	//从缓存中获取
	streamHandler = streamHandlers.get(protocol);
	if (streamHandler != null) {
		return;
	}
	//通过工厂方法创建
	if (streamHandlerFactory != null) {
		streamHandler = streamHandlerFactory.createURLStreamHandler(protocol);
		if (streamHandler != null) {
			streamHandlers.put(protocol, streamHandler);
			return;
		}
	}
	//在同名包下检测一个可用的hadnler
	String packageList = System.getProperty("java.protocol.handler.pkgs");
	ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
	if (packageList != null && contextClassLoader != null) {
		for (String packageName : packageList.split("\\|")) {
			String className = packageName + "." + protocol + ".Handler";
			try {
				Class<?> c = contextClassLoader.loadClass(className);
				streamHandler = (URLStreamHandler) c.newInstance();
				if (streamHandler != null) {
					streamHandlers.put(protocol, streamHandler);
				}
				return;
			} catch (IllegalAccessException ignored) {
			} catch (InstantiationException ignored) {
			} catch (ClassNotFoundException ignored) {
			}
		}
	}
	//如果还是没有创建成功那么new一个handler
	if (protocol.equals("file")) {
		streamHandler = new FileHandler();
	} else if (protocol.equals("ftp")) {
		streamHandler = new FtpHandler();
	} else if (protocol.equals("http")) {
		streamHandler = new HttpHandler();
	} else if (protocol.equals("https")) {
		streamHandler = new HttpsHandler();
	} else if (protocol.equals("jar")) {
		streamHandler = new JarHandler();
	}
	if (streamHandler != null) {
		streamHandlers.put(protocol, streamHandler);
	}
}

```

```java
/**
 * streamHandler实现类为HttpURLConnectionImpl
 */
public final class HttpHandler extends URLStreamHandler {

    @Override protected URLConnection openConnection(URL u) throws IOException {
        return new HttpURLConnectionImpl(u, getDefaultPort());
    }

    @Override protected URLConnection openConnection(URL url, Proxy proxy) throws IOException {
        if (url == null || proxy == null) {
            throw new IllegalArgumentException("url == null || proxy == null");
        }
        return new HttpURLConnectionImpl(url, getDefaultPort(), proxy);
    }

    @Override protected int getDefaultPort() {
        return 80;
    }
}
```

创建连接请求准备
-----------------------------------------------------------
```java
/*****************HttpURLConnectionImpl.java start************************/
/**
 * 无论是get还是post都需要建立连接
 * post
 */
@Override 
public final OutputStream getOutputStream() throws IOException {
	connect();
	OutputStream result = httpEngine.getRequestBody();
	if (result == null) {
		throw new ProtocolException("method does not support a request body: " + method);
	} else if (httpEngine.hasResponse()) {
		throw new ProtocolException("cannot write request body after response has been read");
	}
	return result;
}
/**
 * 无论是get还是post都需要建立连接
 * get
 */
@Override 
public final InputStream getInputStream() throws IOException {
	if (!doInput) {
		throw new ProtocolException("This protocol does not support input");
	}
	//获取http响应
	HttpEngine response = getResponse();
	//返回400抛异常
	if (getResponseCode() >= HTTP_BAD_REQUEST) {
		throw new FileNotFoundException(url.toString());
	}
	InputStream result = response.getResponseBody();
	if (result == null) {
		throw new IOException("No response body exists; responseCode=" + getResponseCode());
	}
	return result;
}
private HttpEngine getResponse() throws IOException {
	//初始化http引擎
	initHttpEngine();
	//是否有响应头信息
	if (httpEngine.hasResponse()) {
		return httpEngine;
	}
	try {
		while (true) {
			//发送请求
			httpEngine.sendRequest();
			httpEngine.readResponse();
			//为下次请求做准备
			Retry retry = processResponseHeaders();
			if (retry == Retry.NONE) {
				httpEngine.automaticallyReleaseConnectionToPool();
				break;
			}
			//如果一个请求不能完成那么接下来为下次请求做准备
			String retryMethod = method;
			OutputStream requestBody = httpEngine.getRequestBody();

			/*
			 * Although RFC 2616 10.3.2 specifies that a HTTP_MOVED_PERM
			 * redirect should keep the same method, Chrome, Firefox and the
			 * RI all issue GETs when following any redirect.
			 */
			int responseCode = getResponseCode();
			if (responseCode == HTTP_MULT_CHOICE || responseCode == HTTP_MOVED_PERM
					|| responseCode == HTTP_MOVED_TEMP || responseCode == HTTP_SEE_OTHER) {
				retryMethod = HttpEngine.GET;
				requestBody = null;
			}

			if (requestBody != null && !(requestBody instanceof RetryableOutputStream)) {
				throw new HttpRetryException("Cannot retry streamed HTTP body",
						httpEngine.getResponseCode());
			}

			if (retry == Retry.DIFFERENT_CONNECTION) {
				httpEngine.automaticallyReleaseConnectionToPool();
			}

			httpEngine.release(true);

			httpEngine = newHttpEngine(retryMethod, rawRequestHeaders,
					httpEngine.getConnection(), (RetryableOutputStream) requestBody);
		}
		return httpEngine;
	} catch (IOException e) {
		httpEngineFailure = e;
		throw e;
	}
}

@Override 
public final void connect() throws IOException {
	initHttpEngine();
	try {
		httpEngine.sendRequest();
	} catch (IOException e) {
		httpEngineFailure = e;
		throw e;
	}
}
/**
 * 无论是get还是post都需要初始化Http引擎
 */
private void initHttpEngine() throws IOException {
	if (httpEngineFailure != null) {
		throw httpEngineFailure;
	} else if (httpEngine != null) {
		return;
	}
	connected = true;
	try {
		if (doOutput) {
			if (method == HttpEngine.GET) {
				//如果要写入那么这就是一个post请求
				method = HttpEngine.POST;
			} else if (method != HttpEngine.POST && method != HttpEngine.PUT) {
				//如果你要写入，那么不是post请求也不是put请求那就抛异常吧。
				throw new ProtocolException(method + " does not support writing");
			}
		}
		httpEngine = newHttpEngine(method, rawRequestHeaders, null, null);
	} catch (IOException e) {
		httpEngineFailure = e;
		throw e;
	}
}
```
创建Socket连接
-----------------------------------------------------------
```java
/********************HttpEngine.java**************/
/**
 * Figures out what the response source will be, and opens a socket to that
 * source if necessary. Prepares the request headers and gets ready to start
 * writing the request body if it exists.
 */
public final void sendRequest() throws IOException {
	if (responseSource != null) {
		return;
	}
	//填充请求头和cookies
	prepareRawRequestHeaders();
	//初始化响应资源，计算缓存过期时间，判断是否读取缓冲中数据，或者进行网络请求
	//responseSource = ?
	//CACHE:返回缓存信息
	//CONDITIONAL_CACHE：进行网络请求如果网络请求结果无效则使用缓存
	//NETWORK:返回网络请求
	initResponseSource();
	//请求行为记录
	if (responseCache instanceof HttpResponseCache) {
		((HttpResponseCache) responseCache).trackResponse(responseSource);
	}
	//请求资源需要访问网络，但请求头部禁止请求。在这种情况下使用BAD_GATEWAY_RESPONSE替代
	if (requestHeaders.isOnlyIfCached() && responseSource.requiresConnection()) {
		if (responseSource == ResponseSource.CONDITIONAL_CACHE) {
			IoUtils.closeQuietly(cachedResponseBody);
		}
		this.responseSource = ResponseSource.CACHE;
		this.cacheResponse = BAD_GATEWAY_RESPONSE;
		RawHeaders rawResponseHeaders = RawHeaders.fromMultimap(cacheResponse.getHeaders());
		setResponse(new ResponseHeaders(uri, rawResponseHeaders), cacheResponse.getBody());
	}

	if (responseSource.requiresConnection()) {
		//socket网络连接
		sendSocketRequest();
	} else if (connection != null) {
		HttpConnectionPool.INSTANCE.recycle(connection);
		connection = null;
	}
}
private void sendSocketRequest() throws IOException {
	if (connection == null) {
		connect();
	}
	if (socketOut != null || requestOut != null || socketIn != null) {
		throw new IllegalStateException();
	}
	socketOut = connection.getOutputStream();
	requestOut = socketOut;
	socketIn = connection.getInputStream();
	if (hasRequestBody()) {
		initRequestBodyOut();
	}
}
//打开Socket连接
protected void connect() throws IOException {
	if (connection == null) {
		connection = openSocketConnection();
	}
}
protected final HttpConnection openSocketConnection() throws IOException {
	HttpConnection result = HttpConnection.connect(
			uri, policy.getProxy(), requiresTunnel(), policy.getConnectTimeout());
	Proxy proxy = result.getAddress().getProxy();
	if (proxy != null) {
		policy.setProxy(proxy);
	}
	result.setSoTimeout(policy.getReadTimeout());
	return result;
}

```
```java
/********************HttpConnection.java**************/
public static HttpConnection connect(URI uri, Proxy proxy, boolean requiresTunnel,
		int connectTimeout) throws IOException {
	//代理直连
	if (proxy != null) {
		Address address = (proxy.type() == Proxy.Type.DIRECT)
				? new Address(uri)
				: new Address(uri, proxy, requiresTunnel);
		return HttpConnectionPool.INSTANCE.get(address, connectTimeout);
	}
	//寻找代理直连
	ProxySelector selector = ProxySelector.getDefault();
	List<Proxy> proxyList = selector.select(uri);
	if (proxyList != null) {
		for (Proxy selectedProxy : proxyList) {
			if (selectedProxy.type() == Proxy.Type.DIRECT) {
				// the same as NO_PROXY
				// TODO: if the selector recommends a direct connection, attempt that?
				continue;
			}
			try {
				Address address = new Address(uri, selectedProxy, requiresTunnel);
				return HttpConnectionPool.INSTANCE.get(address, connectTimeout);
			} catch (IOException e) {
				// failed to connect, tell it to the selector
				selector.connectFailed(uri, selectedProxy.address(), e);
			}
		}
	}
	//创建一个直连接
	return HttpConnectionPool.INSTANCE.get(new Address(uri), connectTimeout);
}
private HttpConnection(Address config, int connectTimeout) throws IOException {
	this.address = config;
	Socket socketCandidate = null;
	InetAddress[] addresses = InetAddress.getAllByName(config.socketHost);
	for (int i = 0; i < addresses.length; i++) {
		socketCandidate = (config.proxy != null && config.proxy.type() != Proxy.Type.HTTP)
				? new Socket(config.proxy)
				: new Socket();
		try {
			//DNS解析，socket连接（这块不做详细分析）
			socketCandidate.connect(
					new InetSocketAddress(addresses[i], config.socketPort), connectTimeout);
			break;
		} catch (IOException e) {
			if (i == addresses.length - 1) {
				throw e;
			}
		}
	}
	this.socket = socketCandidate;
}
```

```java
/********************HttpConnectionPool.java**************/
public HttpConnection get(HttpConnection.Address address, int connectTimeout)
		throws IOException {
	//首先尝试重用现有的HTTP连接。
	synchronized (connectionPool) {
		List<HttpConnection> connections = connectionPool.get(address);
		if (connections != null) {
			while (!connections.isEmpty()) {
				HttpConnection connection = connections.remove(connections.size() - 1);
				if (!connection.isStale()) { // TODO: this op does I/O!
					// Since Socket is recycled, re-tag before using
					final Socket socket = connection.getSocket();
					SocketTagger.get().tag(socket);
					return connection;
				}
			}
			connectionPool.remove(address);
		}
	}
	//无法找到可以复用的链接是，创建一个新的链接
	return address.connect(connectTimeout);
}

/********************HttpConnection.Address.java**************/
public HttpConnection connect(int connectTimeout) throws IOException {
	return new HttpConnection(this, connectTimeout);
}
```

输出内容获取
-----------------------------------------------------------
```java
/********************HttpEngine.java**************/
public final void readResponse() throws IOException {
	//如果有响应头
	if (hasResponse()) {
		return;
	}
	//readResponse之前是否sendRequest
	if (responseSource == null) {
		throw new IllegalStateException("readResponse() without sendRequest()");
	}
	//如果不进行网络请求直接返回
	if (!responseSource.requiresConnection()) {
		return;
	}
	//刷新请求头
	if (sentRequestMillis == -1) {
		int contentLength = requestBodyOut instanceof RetryableOutputStream
				? ((RetryableOutputStream) requestBodyOut).contentLength()
				: -1;
		writeRequestHeaders(contentLength);
	}
	//刷新请求体
	if (requestBodyOut != null) {
		requestBodyOut.close();
		if (requestBodyOut instanceof RetryableOutputStream) {
			((RetryableOutputStream) requestBodyOut).writeToSocket(requestOut);
		}
	}

	requestOut.flush();
	requestOut = socketOut;
	//解析响应头
	readResponseHeaders();
	responseHeaders.setLocalTimestamps(sentRequestMillis, System.currentTimeMillis());
	//判断响应体类型
	if (responseSource == ResponseSource.CONDITIONAL_CACHE) {
		if (cachedResponseHeaders.validate(responseHeaders)) {
			if (responseCache instanceof HttpResponseCache) {
				((HttpResponseCache) responseCache).trackConditionalCacheHit();
			}
			//释放资源
			release(true);
			//返回缓存信息
			setResponse(cachedResponseHeaders.combine(responseHeaders), cachedResponseBody);
			return;
		} else {
			IoUtils.closeQuietly(cachedResponseBody);
		}
	}

	if (hasResponseBody()) {
		maybeCache(); // reentrant. this calls into user code which may call back into this!
	}
	
	initContentStream(getTransferStream());
}
private InputStream getTransferStream() throws IOException {
	if (!hasResponseBody()) {
		return new FixedLengthInputStream(socketIn, cacheRequest, this, 0);
	}

	if (responseHeaders.isChunked()) {
		return new ChunkedInputStream(socketIn, cacheRequest, this);
	}

	if (responseHeaders.getContentLength() != -1) {
		return new FixedLengthInputStream(socketIn, cacheRequest, this,
				responseHeaders.getContentLength());
	}
	return new UnknownLengthHttpInputStream(socketIn, cacheRequest, this);
}
private void initContentStream(InputStream transferStream) throws IOException {
	//是否gzip压缩
	if (transparentGzip && responseHeaders.isContentEncodingGzip()) {
		responseHeaders.stripContentEncoding();
		responseBodyIn = new GZIPInputStream(transferStream);
	} else {
		responseBodyIn = transferStream;
	}
}
```

整个请求的响应流程大概就是这样子的，其中的涉及的路由信息获取，DNS解析与缓存，请求的缓存过期等都还没有仔细研读。不过这些也够自己消化一段时间了^_^，相信自己现在回过头来看OkHttp的实现应该不是那么困难了。

默默肃立的路灯，像等待检阅的哨兵，站姿笔挺，瞪着炯炯有神的眼睛，时刻守护着这城市的安宁。一排排、一行行路灯不断向远方延伸，汇聚成了一支支流光溢彩的河流，偶尔有汽车疾驰而去，也是一尾尾鱼儿在河里游动。夜已深~！~！~！

想阅读作者的更多文章，可以查看我的公共号：
<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>