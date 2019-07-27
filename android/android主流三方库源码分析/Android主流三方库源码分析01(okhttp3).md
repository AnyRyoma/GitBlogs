## OkHttp的整体流程

整个流程是:通过**OkHttpClient**将构建的**Request**转换为**Call**，然后在**RealCall**中进行**异步或同步任务**，最后通过一些的拦截器**interceptor发出网络请求和得到返回的response**。总体流程用下面的图表示

![img](https://upload-images.jianshu.io/upload_images/9984264-3f4ba9f2e1dc097f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/685/format/webp)

## 拆组件

在整体流程中，主要的组件是**OkHttpClient**,其次有**Call**,**RealCall**,**Disptcher**,**各种Interceptors**，**Request和Response**组件。Request和Response已经在上一篇的对其结构源码进行了分析。

### **1. OkHttpClient对象：网络请求的主要操控者**

创建OkHttpClient对象

```java
//通过Builder构造OkHttpClient
 OkHttpClient.Builder builder = new OkHttpClient.Builder()
                .connectTimeout(20, TimeUnit.SECONDS)
                .writeTimeout(20, TimeUnit.SECONDS)
                .readTimeout(20, TimeUnit.SECONDS);
        return builder.build();
```

`OkHttpClient.Builder`类有很多变量，OkHttpClient有很多的成员变量：

```java
final Dispatcher dispatcher;  //重要：分发器,分发执行和关闭由request构成的Call
    final Proxy proxy;  //代理
    final List<Protocol> protocols; //协议
    final List<ConnectionSpec> connectionSpecs; //传输层版本和连接协议
    final List<Interceptor> interceptors; //重要：拦截器
    final List<Interceptor> networkInterceptors; //网络拦截器
    final ProxySelector proxySelector; //代理选择
    final CookieJar cookieJar; //cookie
    final Cache cache; //缓存
    final InternalCache internalCache;  //内部缓存
    final SocketFactory socketFactory;  //socket 工厂
    final SSLSocketFactory sslSocketFactory; //安全套接层socket 工厂，用于HTTPS
    final CertificateChainCleaner certificateChainCleaner; // 验证确认响应证书 适用 HTTPS 请求连接的主机名。
    final HostnameVerifier hostnameVerifier;    //  主机名字确认
    final CertificatePinner certificatePinner;  //  证书链
    final Authenticator proxyAuthenticator;     //代理身份验证
    final Authenticator authenticator;      // 本地身份验证
    final ConnectionPool connectionPool;    //连接池,复用连接
    final Dns dns;  //域名
    final boolean followSslRedirects;  //安全套接层重定向
    final boolean followRedirects;  //本地重定向
    final boolean retryOnConnectionFailure; //重试连接失败
    final int connectTimeout;    //连接超时
    final int readTimeout; //read 超时
    final int writeTimeout; //write 超时
```

OkHttpClient完成整个请求设计到很多参数，都可以通过OkHttpClient.builder使用创建者模式构建。事实上，你能够通过它来设置改变一些参数，因为他是通过建造者模式实现的，因此你可以通过builder()来设置。如果不进行设置，在Builder中就会使用默认的设置：

```java
 public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
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
```

### 2，RealCall：真正的请求执行者

#### 2.1之前文章中的Http发起同步请求的代码：

```
 Request request = new Request.Builder()
      .url(url)
      .build();
  Response response = client.newCall(request).execute();
```

`client.newCall(request).execute()`创建了Call执行了网络请求获得response响应。重点看一看这个执行的请求者的内部是什么鬼。

```
/**
   * Prepares the {@code request} to be executed at some point in the future.
   */
//OkHttpClient中的方法，可以看出RealCall的真面目
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```

RealCall的构造函数：

```
 private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    //client请求
    this.client = client;
    //我们构造的请求
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    //负责重试和重定向拦截器
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }
```

### **3，在获得相应之前经过的最后一关就是拦截器Interceptor**

> the whole thing is just a stack of built-in interceptors.

可见 **Interceptor** 是 OkHttp 最核心的一个东西，不要误以为它只负责拦截请求进行一些额外的处理（例如 cookie），**实际上它把实际的网络请求、缓存、透明压缩等功能都统一了起来**，每一个功能都只是一个 Interceptor，**它们再连接成一个 Interceptor.Chain**，环环相扣，最终圆满完成一次网络请求。

从 **getResponseWithInterceptorChain** 函数我们可以看到，**Interceptor.Chain** 的分布依次是：

![img](https://upload-images.jianshu.io/upload_images/9984264-324b9849ff8f5656.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700/format/webp)

（1）在配置 **OkHttpClient**时设置的**interceptors**；
 （2）负责失败重试以及重定向的 **RetryAndFollowUpInterceptor**；
 （3）负责把**用户构造的请求转换为发送到服务器的请求**、把**服务器返回的响应转换为用户友好的响应的BridgeInterceptor**；
 （4）负责读取缓存直接返回、更新缓存的 **CacheInterceptor**
 （5）负责和服务器建立连接的**ConnectInterceptor**；
 （6）配置 OkHttpClient 时设置的 networkInterceptors；
 （7）负责**向服务器发送请求数据、从服务器读取响应数据**的 **CallServerInterceptor**

在这里，位置决定了功能，**最后一个 Interceptor 一定是负责和服务器实际通讯的**，**重定向、缓存等一定是在实际通讯之前的**

### 责任链拦截器Interceptor

**RetryAndFollowUpInterceptor**:负责失败重试以及重定向
 **BridgeInterceptor**：负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应的 。
 **ConnectInterceptor**:建立连接
 **NetworkInterceptors**：配置OkHttpClient时设置的 NetworkInterceptors
 **CallServerInterceptor**：发送和接收数据