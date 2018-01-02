![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-28-01.png)

## 前言

在各种网络库百花齐放的开源世界中，如果一定要说`OkHttp`有什么过人之处的话，绝对不能忽略他的拦截器`interceptor`。

`OkHttp`通过链式拦截器，使得对请求Request，响应Response的拦截修饰变得异常简洁，本文聊聊拦截器的使用。

*注意：本文所述的拦截器，指代`interceptor`，后文将回混用*

## 拦截器使用

关于拦截器的使用Wiki上有一篇介绍文档可以参考: [Interceptor](https://github.com/square/okhttp/wiki/Interceptors)

顾名思义，拦截器是用于拦截的，可以对发出的Request做修正，比如修改header，参数，添加额外日志，重试机制等，也可以修改Response；

实现一个拦截器，只需要继承`Interceptor`接口，并实现唯一的方法:

> 
@Override public Response intercept(Interceptor.Chain chain) throws IOException

并在其中调用`chain.proceed`，下面看一个官方给出的一个demo示例：


```java
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();

    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}
```

这是一个日志拦截器，在请求前输出请求时间和请求的参数信息，在获得响应后再次输出时间和响应信息；

将我们实现的拦截器运用到`OkHttp`中可以通过`addInterceptor`或者`addNetworkInterceptor`方法，这两个方法都可以添加拦截器，区别在于拦截器应用场景不同：

* addInterceptor 应用拦截器，对单个请求生效，不区分单词请求的内部重定向等
* addNetworkInterceptor 网络拦截器，针对每次网络请求，区分内部重定向等变化

看一张图就明白两者的使用差异了：

![interceptors@2x.png](http://7u2jir.com1.z0.glb.clouddn.com/img/interceptors@2x.png)

## 拦截器分类

还记得5个内部拦截器的遍历么？
其中`interceptors.addAll(client.interceptors());`就是添加应用拦截器；
`interceptors.addAll(client.networkInterceptors());`则是添加网络拦截器（websocket除外）。

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

现在可以测试一下我们的日志拦截器，分别设置为不同类别，来观察效果。

首先我们有一个url：`http://www.publicobject.com/helloworld.txt`如果请求这个地址，会重定向到`https://publicobject.com/helloworld.txt`。


### 应用拦截器

通过`addInterceptor`添加：

```java
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new LoggingInterceptor())
    .build();

Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();

Response response = client.newCall(request).execute();
response.body().close();
```

触发程序，得到的日志如下：

```
INFO: Sending request http://www.publicobject.com/helloworld.txt on null
User-Agent: OkHttp Example

INFO: Received response for https://publicobject.com/helloworld.txt in 1179.7ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```

可以看到请求和响应日志各输出一次，并且起始url和最终url是不同的。这是因为我们的`OkHttp`自动完成了重定向操作，并将最终结果返回给客户端。

### 网络拦截器

接下来通过网络拦截器，我们将得到不同的日志：

```java
OkHttpClient client = new OkHttpClient.Builder()
    .addNetworkInterceptor(new LoggingInterceptor())
    .build();

Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();

Response response = client.newCall(request).execute();
response.body().close();
```
由于存在一次重定向，因此日志将会包含两次请求日志和两次响应日志：

```
INFO: Sending request http://www.publicobject.com/helloworld.txt on Connection{www.publicobject.com:80, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=none protocol=http/1.1}
User-Agent: OkHttp Example
Host: www.publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip

INFO: Received response for http://www.publicobject.com/helloworld.txt in 115.6ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/html
Content-Length: 193
Connection: keep-alive
Location: https://publicobject.com/helloworld.txt

INFO: Sending request https://publicobject.com/helloworld.txt on Connection{publicobject.com:443, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA protocol=http/1.1}
User-Agent: OkHttp Example
Host: publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip

INFO: Received response for https://publicobject.com/helloworld.txt in 80.9ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```

## 拦截器选择

由于不同拦截器作用是不同的，如果我们自定义拦截器需要考虑自身的业务场景，来确定具体选用哪一种类型。下面是翻译的一些特性：

### Application interceptor特点

* 无需关注中间响应，比如重试和重定向；
* 总是被触发一次，即使响应数据是来自本地缓存；
* 关注程序的初衷，不关心插入的头信息，比如`If-None-Match`
* 允许中断，不调用Chain.proceed
* 允许重试，调用多次Chain.proceed

### Network Interceptors特点

* 可以控制中间响应，包括重定向和重试；
* 缓存数据响应不会触发拦截器；
* 可以访问请求通道；

无论哪种拦截器，在修改请求和响应是必须十分慎重，特别是修改数据，以避免非预期的坑。
最后一点是，拦截器适用于`OkHttp 2.2`以上版本，并且不适用`OkUrlFactory`