---
layout: posts
title: okHttp解析
date: 2018-06-04 10:31:58
tags: [源码解析,笔记,android]
categories: 源码解析
---
## 介绍
OkHttp 现在作为Android时下最流行的网络框架(Retrofit 也是以OkHttp为底层的).先将okhttp的学习整理如下,本文是以3.10版本进行分析.(因为3.9版本跟3.10版本差别不大,有些图片的引用还是3.9的)

感谢: [json_it的博客](https://blog.csdn.net/json_it/article/details/78404010)  [码迷](http://www.mamicode.com/info-detail-2161332.html)

<img src = "http://okskqdic8.bkt.clouddn.com/okhttp_1.jpg" width = 500/>

## okhttp网络请求流程
以官网demo为例进行分析

<!-- more -->

<font color=#aaaaaa >Get请求</font>

```java
OkHttpClient client = new OkHttpClient();
String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();
  Response response = client.newCall(request).execute();
  return response.body().string();
}
```

<font color=#aaaaaa >Post请求</font>

```java
public static final MediaType JSON
    = MediaType.parse("application/json; charset=utf-8");
OkHttpClient client = new OkHttpClient();
String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(JSON, json);
  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
  Response response = client.newCall(request).execute();
  return response.body().string();
}
```
### 调用流程图如下

<img src = "http://okskqdic8.bkt.clouddn.com/okhttp_2_1.png" width = 500/>
## 类,方法具体分析
### OkHttpClient
要完成一次请求,我们必须先有OkHttpClient实例,使用者的所有操作都是通过它来完成的.OkHttpClient用于协调各个子模块工作,各个子模块直接相互独立.OkHttpClient因为有众多子模块,所以使用建造者设计模式进行装配.获取OkHttpClient实例有两种方式

#### 方式1:

```java
	OkHttpClient client=new OkHttpClient(); 
```
不是说使用的是建造者模式吗?为啥没有Builder?
因为OkHttpClient的构造函数默认使用了默认的builder()

```java
	public OkHttpClient() {
    		this(new Builder());
 	}

	........

	public Builder() {
      		//调度器
      		dispatcher = new Dispatcher();
      		//请求协议 http1.1 http2
      		protocols = DEFAULT_PROTOCOLS;
      		// 链接规范 普通请求 加密请求(TLS)
      		connectionSpecs = DEFAULT_CONNECTION_SPECS;
      		//应用程序的HTTP调用的数量、大小和持续时间
      		eventListenerFactory =EventListener.factory(EventListener.NONE);
      		//选择在连接到URL引用的网络资源时要使用的代理服务器(如果有的话)
      		proxySelector = ProxySelector.getDefault();
      		//cookie
      		cookieJar = CookieJar.NO_COOKIES;
      		//java 套接字工厂
      		socketFactory = SocketFactory.getDefault();
      		//用于HTTPS请求时与域名校验
      		hostnameVerifier = OkHostnameVerifier.INSTANCE;
      		//https证书校验
      		certificatePinner = CertificatePinner.DEFAULT;
      		proxyAuthenticator = Authenticator.NONE;
      		authenticator = Authenticator.NONE;
      		//连接池
      		connectionPool = new ConnectionPool();
      		dns = Dns.SYSTEM;
      		followSslRedirects = true;
      		followRedirects = true;
      		retryOnConnectionFailure = true;
      		connectTimeout = 10_000;
      		readTimeout = 10_000;
      		writeTimeout = 10_000;
      		pingInterval = 0;
    	}

	........	

```
#### 方式2:

```java
	OkHttpClient.Builder bulider=new OKHttpClient.Builder(); 
	//根据builder进行自定义
	OkHttpClient client=buider.xxx().xxx().build();
```

注: 因为OkHttpClient内部有连接池,线程池等,所以一般来说一个应用就一个OkHttpClient实例.

### Request
Request类比较简单 只包含了 url, method, headers, body等一次请求的基本信息
### RealCall
RealCall 目前是Call的唯一实现类,表示一个请求/响应对(流),用于操作管理请求.如:发送同步/异步请求,取消请求等.每一个call只能执行一次.其中重要方法如下

同步请求:

<img src="http://okskqdic8.bkt.clouddn.com/okhttp_3.jpg" width = 500/>

异步请求:

<img src="http://okskqdic8.bkt.clouddn.com/okhttp_4.jpg" width = 500/>
<img src = "http://okskqdic8.bkt.clouddn.com/okhttp_5.jpg" width = 500/>

getResponseWithInterceptorChain:

<img src = "http://okskqdic8.bkt.clouddn.com/okhttp_6_1.jpg" width = 500 />

  从以上可以看到,RealCall 通过 execute / enqueue发送同步/异步请求,
getResponseWithInterceptorChain获取Response.其中异步请求会用Dispatcher来进行分配,虽然同步请求中也使用了Dispatcher但只是记录,稍后会在Dispatcher中详细分析.在getResponseWithInterceptorChain中,将开发者自定义的interceptor可必须的interceptor为每一个call进行添加,然后通过RealInterceptorChain进行逐个调用,具体会在下面分析.
### Dispatcher
Dispatcher保存了一个OkHttpClient里的所有Call,包含同步Call(一个deque保存)和异步Call(两个Deque保存).其中异步Call默认最大支持64请求,单个Host默认最大5个请求.每个dispatcher使用一个{@link ExecutorService}来在内部运行调用。默认使用ExecutorService实现类ThreadPoolExecutor.如图:

<img src="http://okskqdic8.bkt.clouddn.com/okhttp_7_1.png" width = 500/>

同步请求:

在上面RealCall同步请求分析中,需要Dispatch做什么呢?

RealCall

```java
@Override public Response execute() throws IOException {
   ......
    try {
      client.dispatcher().executed(this);
	......
      } finally {
      client.dispatcher().finished(this);
  }
```

Dispatcher

```java
.......
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
.......

void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }
	......
  }
```

可以看到在同步请求中,Dispatcher只是将任务存入了runningSyncCalls集合中.在通过getResponseWithInterceptorChain获取到Response,最终会执行finally语句,将任务移除runningSyncCalls集合.

异步请求:

在RealCall执行异步请求时,最关键的代码是:

```java
@Override public void enqueue(Callback responseCallback) {
	......    
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
Dispatcher中enqueue方法

```java
	synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```

可以看到如果正在执行的异步请求数小于最大请求数并且单个host异步请求数小于 maxRequestsPerHost(5)时,将AsyncCall直接放入正在执行的异步集合中,并立即执行,否则就将该请求放入readyAsyncCalls集合中.AsyncCall是Runnable的子类（间接），因此，在线程池中最终会调用AsyncCall的execute（）方法执行异步请求.


```java
@Override protected void execute() {
	......
       try {
        Response response = getResponseWithInterceptorChain();
           } catch (IOException e) {
		......
          } finally {
        client.dispatcher().finished(this);
       }
```

异步请求,最终也是通过getResponseWithInterceptorChain来获取响应的Response. 区别在于client.dispatcher().finished(this);因为这是另一个异步任务,所以调用的是另外一个finish方法：

```java
 void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
   ......
    synchronized (this) {
    if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
 	......
}
```
因为promoteCalls为true,所以要执行promoteCalls(); 方法:

```java
private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // 已经运行超过最大请求数,则return
    if (readyAsyncCalls.isEmpty()) return; //缓存等待的请求为空,则return

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }
      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```
可以看到如果有等待的异步请求,则会在符合最大请求数以及最大单个host请求数的前提下,继续执行.直到所有请求都执行完毕.

### RealInterceptorChain
在介绍各个拦截器之前,需要先分析RealInterceptorChain,直译就是拦截器链.为什么要先分析这个类呢?因为各个Interceptor就是通过RealInterceptorChain这个类按照顺序调用的!,先看这个类是在哪里调用的,在上面分析RealCall中获取Response的getResponseWithInterceptorChain方法中:

<img src="http://okskqdic8.bkt.clouddn.com/okhttp_8.jpg" width = 500/>

可以看到我们添加的所有interceptors都传给了RealInterceptorChain类中,并通过RealInterceptorChain的proceed方法获取Response.我们在来看proceed方法:

```java
@Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
	......   
	// Call the next interceptor in the chain.
	RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
	Interceptor interceptor = interceptors.get(index);
	Response response = interceptor.intercept(next);
	......
	return response;
}
```
这个方法最重要的就是又重新创建了一个RealInterceptorChain(此时index+1),并且通过interceptors.get(index);获取拦截器后执行拦截器intercept方法.在后续分析中可以看到intercept(Chain chain)方法会继续调用责任链的proceed()方法.这样依次调用,<font color=#ff0000 >当前拦截器的Response依赖于下一个拦截器的Intercept的Response。</font>当执行到最后一个拦截器获取到Response后沿着调用相反的方向依次将Response返回,有点类似递归.

### RetryAndFollowUpInterceptor
根据类名我就可以知道此拦截器用于失败重试,并根据需要进行重定向,并且如果调用被取消它可能会抛出IOException("Canceled")。
具体方法如下:

```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request(); //获取请求信息
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();  //获取RealCall
    EventListener eventListener = realChain.eventListener(); //监听器

    //用于协调链接,流,请求三者之间的关系后边会详细分析
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0; //
    Response priorResponse = null;
    while (true) {
      if (canceled) { //如果请求取消则抛出取消异常
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {
        //调用下一个拦截器
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // 试图通过路由连接失败。请求将不会被发送。
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue; //重试
      } catch (IOException e) {
        // 试图与服务器通信失败。请求可能已经发出。
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue; //继续重试
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // 如果前一个生成的Response 不为空,则将前一次的response添加到本次的response中
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp;
      try {
        //followUpRequest会根据返回的response的code来决定是否进行重试,例如:返回code为200,或者请求超时等则返回null
        //否则会根据code进行调整请求进行重试
        followUp = followUpRequest(response, streamAllocation.route());
      } catch (IOException e) {
        streamAllocation.release();
        throw e;
      }

      //followUp为null,返回当前的Response,如果不是长连接则释放资源
      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      //释放ResponseBody所关联的资源
      closeQuietly(response.body());

      //超过最大重试次数
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      //不是很清楚,看followUpRequest方法好像在超时的时候会有这个情况
      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      //如果请求可以重用此链接sameConnection方法返回true,如果不能重用就获取一个新链接
      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } else if (streamAllocation.codec() != null) {
        //如果是可以重此链接,但是HttpCodec(编码HTTP请求并解码HTTP响应)并没有重置则抛出异常
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }
```

### BridgeInterceptor
从应用程序代码到网络代码的桥梁。首先，它从用户请求构建网络请求。然后它继续调用网络。最后，它从网络响应构建用户响应。其实就是对Request和Response的Header进行操作,主要是为Request添加请求头,为Response添加响应头如图:

<img src = "http://okskqdic8.bkt.clouddn.com/okhttp_9.png"/>

功能很简单,就不贴源码了

### CacheInterceptor

### ConnectInterceptor
#### StreamAllocation
#### RealConnection
#### HttpCodec
### CallServerInterceptor
### Response