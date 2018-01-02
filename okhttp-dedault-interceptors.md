![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-01-02-01.png)

## 前言

我们知道`OkHttp`自带了5个原生拦截器，本文一起分析这几个拦截器的作用和实现。

## RetryAndFollowUpInterceptor

重试/重定向拦截器，是第一个加入列表的拦截器，用于自动处理重定向和请求重试，如果一个请求被取消了会向上抛出IO异常。
其默认重试次数为`20`,这个次数和大多数程序的一致。

```java
 /**
  * How many redirects and auth challenges should we attempt? Chrome follows 21 redirects; Firefox,
  * curl, and wget follow 20; Safari follows 16; and HTTP/1.0 recommends 5.
  */
 private static final int MAX_FOLLOW_UPS = 20;
```

实现重试的主要思路是无线循环配合计步器，还有一些错误情况，会向上抛异常；

```java
int followUpCount = 0;
while(true) {
  // ...
  if (++followUpCount > MAX_FOLLOW_UPS) {
    streamAllocation.release();
    throw new ProtocolException("Too many follow-up requests: " + followUpCount);
  }
  // ...
}
```

核心逻辑在于，请求下一个拦截器，根据其返回的Resposne做相应处理。

```java
try {
  response = realChain.proceed(request, streamAllocation, null, null);
  releaseConnection = false;
} catch (RouteException e) {
  // The attempt to connect via a route failed. The request will not have been sent.
  if (!recover(e.getLastConnectException(), false, request)) {
    throw e.getLastConnectException();
  }
  releaseConnection = false;
  continue;
} catch (IOException e) {
  // An attempt to communicate with a server failed. The request may have been sent.
  boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
  if (!recover(e, requestSendStarted, request)) throw e;
  releaseConnection = false;
  continue;
} finally {
  // We're throwing an unchecked exception. Release any resources.
  if (releaseConnection) {
    streamAllocation.streamFailed(null);
    streamAllocation.release();
  }
}
```

`Chain`和`Interceptor`的关系，如下：

RealCall, RealInterceptorChain:

`chain.proceed(originalRequest)`=>`chain#proceed`=>`next Interceptor（RetryAndFollowUpInterceptor）`=>`realChain.proceed(request, streamAllocation, null, null)`=>`next Interceptor(BridgeInterceptor)`=>


## BridgeInterceptor

## CacheInterceptor

## ConnectInterceptor

## CallServerInterceptor