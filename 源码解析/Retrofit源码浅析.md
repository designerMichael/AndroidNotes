---
title: Retrofit源码浅析
date: 2016-08-21 23:33:40
categories: 源码解析
tags: [Retrofit,源码解析]
---


# 网络请求基本流程
![](http://oeiu2t0ur.bkt.clouddn.com/625299-e0f8fda2d0855996.png)

1. 首先build一个request，设置好请求方法，请求参数等
2. 维护一个线程池，把build好的请求enqueue进去，执行你的请求
3. 解析服务器得到的数据后，callback回调给你的上层。

<!--more-->

Retrofit也可以大致分为这三步：

1. 通过注解配置API参数
2. `callFactory`负责创建 HTTP 请求,通过`callAdapter`执行请求，拿到返回数据
3. 通过`Convert`转换成我们需要的数据

# Retrofit基本使用
## 1.定义HTTP API
`Retrofit`把`HTTP API`变成 Java 的接口。
```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```
在`GithubService`接口中有一个方法 `listRepos`，这个方法用了`@GET`的方式注解，这表明这是一个 GET 请求。可以看出 Retrofit 用注解的方式来描述一个网络请求相关的参数。

## 2.创建 Retrofit 对象
```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build();
```
先是构建了一个`Retrofit`对象，其中传入了`baseUrl`参数,添加了`converterFactory`，用于把返回的 `http response `转换成 `Java `对象
## 3.获取 API 实例
`GitHubService github = retrofit.create(GitHubService.class);`

## 4. 发起请求
可以使用`enqueue`或者`execute`来执行发起请求，`enqueue` 是是异步执行，而`execute`是同步执行。

# 源码解析

## Retrofit 的创建
`Retrofit`创建使用的是 `Builder `模式。`Retrofit` 中有如下的几个关键变量：
```java
//用于缓存解析出来的方法
  private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();
  
  //请求网络的OKHttp的工厂，默认是 OkHttpClient
  private final okhttp3.Call.Factory callFactory;
  
  //baseurl
  private final HttpUrl baseUrl;
  
  //请求网络得到的response的转换器的集合 默认会加入 BuiltInConverters     
  private final List<Converter.Factory> converterFactories;
  
  //把Call对象转换成其它类型
  private final List<CallAdapter.Factory> adapterFactories;
  
  //用于执行回调 Android中默认是 MainThreadExecutor
  private final Executor callbackExecutor;
  
  //是否需要立即解析接口中的方法
  private final boolean validateEagerly;
```
再看一下`Retrofit`中的内部类`Builder`的 `builder `方法：
```
public Retrofit build() {
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }

  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
    //默认创建一个 OkHttpClient
    callFactory = new OkHttpClient();
  }

  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
     //Android 中返回的是 MainThreadExecutor
    callbackExecutor = platform.defaultCallbackExecutor();
  }

  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
  adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

  // Make a defensive copy of the converters.
  List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

  return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
      callbackExecutor, validateEagerly);
}
```
在创建 `Retrofit`的时候，如果没有指定 `OkHttpClient`，会创建一个默认的。如果没有指定 `callbackExecutor`，会返回平台默认的，在 Android 中是 `MainThreadExecutor`，并利用这个构建一个 `CallAdapter`加入 `adapterFactories`。

## create 方法
有了`Retrofit`对象后，便可以通过 `create` 方法创建网络请求接口类的实例，代码如下：
```java
public <T> T create(final Class<T> service) {
Utils.validateServiceInterface(service);
if (validateEagerly) {
  //提前解析方法
  eagerlyValidateMethods(service);
}
return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
    new InvocationHandler() {
      private final Platform platform = Platform.get();

      @Override public Object invoke(Object proxy, Method method, Object... args)
          throws Throwable {
        // If the method is a method from Object then defer to normal invocation.如果是Object中的方法，直接调用
        if (method.getDeclaringClass() == Object.class) {
          return method.invoke(this, args);
        }
        //为了兼容 Java8 平台，Android 中不会执行
        if (platform.isDefaultMethod(method)) {
          return platform.invokeDefaultMethod(method, service, proxy, args);
        }
        //下面是重点，解析方法
        ServiceMethod serviceMethod = loadServiceMethod(method);
        OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
        return serviceMethod.callAdapter.adapt(okHttpCall);
      }
});
```

`create `方法接受一个 `Class` 对象，也就是我们编写的接口，里面含有通过注解标识的请求网络的方法。注意 `return` 语句部分，这里调用了 `Proxy.newProxyInstance` 方法，这里用了动态代理模式。关于动态代理模式，可以参考这篇文章：[公共技术点之 Java 动态代理](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)。简单的描述就是，`Proxy.newProxyInstance `根据传进来的接口`Class` 对象生成了一个实例 A，也就是代理类。每当这个代理类 A 执行某个方法时，总是会调用 `InvocationHandler`(`Proxy.newProxyInstance` 中的第三个参数) 的 `invoke` 方法，在这里除了执行真正的逻辑（例如再次转发给真正的实现类对象），我们还可以进行一些有用的操作，例如统计执行时间、进行初始化和清理、对接口调用进行检查等。

为什么要用动态代理？因为对接口的所有方法的调用都会集中转发到 `InvocationHandler#invoke `函数中，我们可以集中进行处理，更方便了。

这里的`Platform`其实是检测`retrofit`所运行的平台，是`java8`还是`android`还是`ios`。在后面build的时候，如果没有设置适配器，那么`retrofit`就会通过运行时的不同平台，然后选择不同的`CallAdapterFactory`。

在`invoke`方法中首先做了几个判断，如果调用的是 `Object` 的方法，例如 `equals，toString`，那就直接调用。如果是 `default` 方法（Java 8 引入），就调用 `default` 方法。当然Android中都不会是这两种情况。
真正关键的代码是下面这三行：
```java
ServiceMethod serviceMethod = loadServiceMethod(method);
OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
return serviceMethod.callAdapter.adapt(okHttpCall);
```
这三个方法对应的关键类分别是：`ServiceMethod`， `OkHttpCall`，`CallAdapter`。

### ServiceMethod<T> 
`ServiceMethod<T>`类的作用正如其 `JavaDoc` 所言：
> Adapts an invocation of an interface method into an HTTP call. 把对接口方法的调用转为一次 HTTP 调用。

一个 `ServiceMethod `对象对应于一个` API interface` 的一个方法，`loadServiceMethod(method)`方法负责创建一个 `ServiceMethod`：
```java
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
这里实现了缓存逻辑，同一个` API` 的同一个方法，只会创建一次。这里由于我们每次获取 API 实例都是传入的 class 对象，而 class 对象是进程内单例的，所以获取到它的同一个方法 `Method` 实例也是单例的，所以这里的缓存是有效的。

再看看 `ServiceMethod `的构造函数：
```java
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
```
以及`build`方法：
```java
public ServiceMethod build() {
  //获取 callAdapter
  callAdapter = createCallAdapter();
  responseType = callAdapter.responseType();
  if (responseType == Response.class || responseType == okhttp3.Response.class) {
    throw methodError("'"
        + Utils.getRawType(responseType).getName()
        + "' is not a valid response body type. Did you mean ResponseBody?");
  }
  //获取 responseConverter 
  responseConverter = createResponseConverter();

  for (Annotation annotation : methodAnnotations) {
    //解析注解
    parseMethodAnnotation(annotation);
    //省略了一些代码
    ...
  }
}
```

重点关注四个成员：`callFactory`，`callAdapter`，`responseConverter` 和 `parameterHandlers`。

#### 1.callFactory
`callFactory` 负责创建 HTTP 请求，HTTP 请求被抽象为了 `okhttp3.Call `类，它表示一个已经准备好，可以随时执行的 HTTP 请求；
`this.callFactory = builder.retrofit.callFactory()`，所以 `callFactory` 实际上由` Retrofit` 类提供，而我们在构造 `Retrofit` 对象时，可以指定 `callFactory`，默认为` okhttp3.OkHttpClient`。


#### 2.callAdapter
`callAdapter`通过`createCallAdapter()`方法获取:
```java
private CallAdapter<?> createCallAdapter() {
  // 省略检查性代码
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { 
    // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create call adapter for %s", returnType);
  }
}
```
可以看到，`callAdapter`还是由 `Retrofit`类提供。在`Retrofit` 类内部，将遍历一个 `CallAdapter.Factory` 列表，让工厂们提供，如果最终没有工厂能（根据 returnType 和 annotations）提供需要的 CallAdapter，那将抛出异常。而这个工厂列表我们可以在构造 Retrofit 对象时进行添加。例如我们最常用的结合RxJava：
`addCallAdapterFactory(RxJavaCallAdapterFactory.create())`
`callAdapter`作用：
`callAdapter` 把 `retrofit2.Call<T> `转为` T`（注意和 `okhttp3.Call` 区分开来，`retrofit2.Call<T>` 表示的是对一个 `Retrofit `方法的调用），这个过程会发送一个 HTTP 请求，拿到服务器返回的数据（通过 `okhttp3.Call` 实现），并把数据转换为声明的 T 类型对象（通过 `Converter<F, T>` 实现）；

#### 3.responseConverter
```java
private Converter<ResponseBody, T> createResponseConverter() {
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.responseBodyConverter(responseType, annotations);
  } catch (RuntimeException e) { 
    // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create converter for %s", responseType);
  }
}
```
同样，`responseConverter`还是由 `Retrofit` 类提供，而在其内部，逻辑和创建 `callAdapter` 基本一致，通过遍历 `Converter.Factory `列表，看看有没有工厂能够提供需要的 `responseBodyConverter`。工厂列表同样可以在构造 `Retrofit` 对象时进行添加。例如最常用的添加Gson转换器:`addConverterFactory(GsonConverterFactory.create())
`
`responseConverter` 是 `Converter<ResponseBody, T>` 类型，负责把服务器返回的数据（JSON、XML、二进制或者其他格式，由 `ResponseBody` 封装）转化为` T` 类型的对象；

#### 4.parameterHandlers
 `parameterHandlers`则负责解析 API 定义时每个方法的参数，每个参数都会有一个` ParameterHandler`，由 `ServiceMethod#parseParameter` 方法负责创建，其主要内容就是解析每个参数使用的注解类型（诸如 `Path，Query，Field` 等），对每种类型进行单独的处理,最终构建成一个完整的HTTP请求。

`Converter.Factory `除了提供上一小节提到的 `responseBodyConverter`，还提供 `requestBodyConverter` 和 `stringConverter`，API 方法中除了` @Body` 和` @Part` 类型的参数，都利用 `stringConverter` 进行转换，而` @Body `和 `@Part` 类型的参数则利用` requestBodyConverter `进行转换。

#### 工厂让各个模块得以高度解耦
上面提到了三种工厂：`okhttp3.Call.Factory`，`CallAdapter.Factory` 和` Converter.Factory`，分别负责提供不同的模块，至于怎么提供、提供何种模块，统统交给工厂，`Retrofit` 完全不掺和，它只负责提供用于决策的信息，例如参数/返回值类型、注解等。

### OkHttpCall

在得到 `ServiceMethod` 对象后，把它连同方法调用的相关参数传给了 `OkHttpCall` 对象，`OkHttpCall`继承于` Call` 接口。`Call` 是`Retrofit`的基础接口，代表发送网络请求与响应调用，它包含下面几个接口方法：

> 
- Response<T> execute() throws IOException; //同步执行请求
- void enqueue(Callback<T> callback); //异步执行请求，callback 用于回调
- boolean isExecuted(); //是否执行过
- void cancel(); //取消请求
- boolean isCanceled(); //是否取消了
- Call<T> clone(); //克隆一条请求
- Request request(); //获取原始的request

`OkHttpCall`是 `Call` 的一个实现类，它里面封装了 `OkHttp` 中的原生 `Call`，在这个类里面实现了` execute` 以及 `enqueue` 等方法，其实是调用了 `OkHttp `中原生` Call` 的对应方法。

#### execute()
```java
@Override 
public Response<T> execute() throws IOException {
  okhttp3.Call call;

  synchronized (this) {
    // 省略部分检查代码

    call = rawCall;
    if (call == null) {
      try {
        call = rawCall = createRawCall();
      } catch (IOException | RuntimeException e) {
        creationFailure = e;
        throw e;
      }
    }
  }

  return parseResponse(call.execute());
}

private okhttp3.Call createRawCall() throws IOException {
  Request request = serviceMethod.toRequest(args);
  okhttp3.Call call = serviceMethod.callFactory.newCall(request);
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}

Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

  // Remove the body's source (the only stateful object) so we can pass the response along.
  rawResponse = rawResponse.newBuilder()
      .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
      .build();

  int code = rawResponse.code();
  if (code < 200 || code >= 300) {
    // ...返回错误
  }

  if (code == 204 || code == 205) {
    return Response.success(null, rawResponse);
  }

  ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
  try {
    T body = serviceMethod.toResponse(catchingBody);
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    // ...异常处理
  }
}
```

主要包括三步：
> 
创建 okhttp3.Call，包括构造参数；
执行网络请求；
解析网络请求返回的数据；

`createRawCall()`函数中，我们调用了` serviceMethod.toRequest(args)` 来创建 `okhttp3.Request`，然后我们再调用 `serviceMethod.callFactory.newCall(request) `来创建 `okhttp3.Call`，这里之前准备好的 `callFactory` 同样也派上了用场，由于工厂在构造` Retrofit` 对象时可以指定，所以我们也可以指定其他的工厂（例如使用过时的 `HttpURLConnection `的工厂），来使用其它的底层 HttpClient 实现。

我们调用` okhttp3.Call#execute()` 来执行网络请求，这个方法是阻塞的，执行完毕之后将返回收到的响应数据。收到响应数据之后，我们进行了状态码的检查，通过检查之后我们调用了 `serviceMethod.toResponse(catchingBody)` 来把响应数据转化为了我们需要的数据类型对象。在 `toResponse` 函数中，会调用之前的`responseConverter` 生成T返回。

#### enqueue(Callback<T> callback)
这里的异步交给了 `okhttp3.Call#enqueue(Callback responseCallback)` 来实现，并在它的 `callback` 中调用 `parseResponse` 解析响应数据，并转发给传入的 `callback`。

关于上面的`execute`和`enqueue`在`OkHttp`源码分析中有更多详细介绍。

### CallAdapter
`CallAdapter<T>#adapt(Call<R> call)` 函数负责把 `retrofit2.Call<R>` 转为 另外一种类型。比如默认情况下的`callAdapter`为`ExecutorCallAdapterFactory`,转成成的T就是`retrofit2.Call<R>`这个`Call`对象。当 `Retrofit` 和` RxJava `结合使用的时候，可以返回 `Observable<R>`，这里相当于适配器模式。

至此，一次对 API 方法的调用是如何构造并发起网络请求、以及解析返回数据，这整个过程大致是分析完毕了。大致流程图如下：

![](http://oeiu2t0ur.bkt.clouddn.com/625299-29a632638d9f518f.png)

## retrofit-adapters 模块
`retrofit `模块内置了 `DefaultCallAdapterFactory` 和 `ExecutorCallAdapterFactory`，它们都适用于 API 方法得到的类型为 `retrofit2.Call `的情形，前者生产的 `adapter` 啥也不做，直接把参数返回，后者生产的 `adapter` 则会在异步调用时在指定的 Executor 上执行回调。
`retrofit-adapters` 的各个子模块则实现了更多的工厂：`GuavaCallAdapterFactory`，`Java8CallAdapterFactory `和 `RxJavaCallAdapterFactory`。

## retrofit-converters 模块
`retrofit `模块内置了` BuiltInConverters`，只能处理 `ResponseBody`， `RequestBody` 和 `String` 类型的转化（实际上不需要转）。而 `retrofit-converters` 中的子模块则提供了` JSON，XML，ProtoBuf` 等类型数据的转换功能，而且还有多种转换方式可以选择，比如最常用的 `GsonConverterFactory`。

# 总结
`Retrofit`主要就是通过动态代理的方式把 `Java` 接口中的解析为响应的网络请求，然后交给` OkHttp` 去执行。并且可以适配不同的` CallAdapter`，可以方便与` RxJava` 结合使用。

# 参考
[拆轮子系列：拆 Retrofit](http://blog.piasy.com/2016/06/25/Understand-Retrofit/)
[Android Retrofit源码解析](https://segmentfault.com/a/1190000006767113)
[Retrofit分析-经典设计模式案例](http://www.jianshu.com/p/fb8d21978e38)