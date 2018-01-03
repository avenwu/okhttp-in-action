![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-01-02-01.png)

## 前言

我们知道`OkHttp`自带了5个原生拦截器，本文一起分析默认的`重定向拦截器`的作用和实现。

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

实现重试的主要思路是`无限循环`+`计步器`，还有一些错误情况，会向上抛异常；

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

## 上抛异常分析

科学来讲，重试器不会对所有情况进行重试，因此针对有些既定情况会直接抛出异常终端重试机制；那么都有哪些中断情况呢，接下来注意看一下几个关键异常的上抛逻辑。

* RouteException

在chain执行proceed的时候，会对其加catch，第一个捕获判断即使`RouteException`，代表链路异常。

```java
} catch (RouteException e) {
// The attempt to connect via a route failed. The request will not have been sent.
if (!recover(e.getLastConnectException(), false, request)) {
  throw e.getLastConnectException();
}
releaseConnection = false;
continue;
} 
```

在异常处理case中，会调用`recover`来确定是否需要进行下一个重试，否则的话直接抛出异常。

`recover`中会依次判断集中情况：

1. client配置是否允许重试
2. 请求体是否可重复发送
3. 请求能否恢复，这一种包含几种异常：ProtocolException，SocketTimeoutException，CertificateException，SSLPeerUnverifiedException
4. 是否达到重试次数上限

```java
/**
 * Report and attempt to recover from a failure to communicate with a server. Returns true if
 * {@code e} is recoverable, or false if the failure is permanent. Requests with a body can only
 * be recovered if the body is buffered or if the failure occurred before the request has been
 * sent.
 */
private boolean recover(IOException e, boolean requestSendStarted, Request userRequest) {
  streamAllocation.streamFailed(e);

  // The application layer has forbidden retries.
  if (!client.retryOnConnectionFailure()) return false;

  // We can't send the request body again.
  if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

  // This exception is fatal.
  if (!isRecoverable(e, requestSendStarted)) return false;

  // No more routes to attempt.
  if (!streamAllocation.hasMoreRoutes()) return false;

  // For failure recovery, use the same route selector with a new connection.
  return true;
}
```


*  IOException

捕获异常的第二个case是常见的IO异常情况，通过类型是否为`ConnectionShutdownException`确定是否发送了请求，然后执行相同的`recover`逻辑。

```java
} catch (IOException e) {
  // An attempt to communicate with a server failed. The request may have been sent.
  boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
  if (!recover(e, requestSendStarted, request)) throw e;
  releaseConnection = false;
  continue;
} 
```

* ProtocolException

接下来会判断重试次数

```java
if (++followUpCount > MAX_FOLLOW_UPS) {
  streamAllocation.release();
  throw new ProtocolException("Too many follow-up requests: " + followUpCount);
}
```

* HttpRetryException

判断请求体是否可以重复请求

```java
if (followUp.body() instanceof UnrepeatableRequestBody) {
  streamAllocation.release();
  throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
}
```


* IllegalStateException

最后一种判断`streamAllocation`的状态
```java
} else if (streamAllocation.codec() != null) {
  throw new IllegalStateException("Closing the body of " + response
      + " didn't close its backing stream. Bad interceptor?");
}
```


## 重试/重定向状态码

我们已经看了很多重试的逻辑判断，那么到底哪些情况会触发链式重试机制呢？这个需要需要结合HTTP的状态码来看。HTTP的相关协议可以查看：[https://www.w3.org/Protocols/](https://www.w3.org/Protocols/)

下面我们逐一分析`OkHttp`识别的几种常见码：

```java 
import static java.net.HttpURLConnection.HTTP_CLIENT_TIMEOUT;
import static java.net.HttpURLConnection.HTTP_MOVED_PERM;
import static java.net.HttpURLConnection.HTTP_MOVED_TEMP;
import static java.net.HttpURLConnection.HTTP_MULT_CHOICE;
import static java.net.HttpURLConnection.HTTP_PROXY_AUTH;
import static java.net.HttpURLConnection.HTTP_SEE_OTHER;
import static java.net.HttpURLConnection.HTTP_UNAUTHORIZED;
import static okhttp3.internal.Util.closeQuietly;
import static okhttp3.internal.http.StatusLine.HTTP_PERM_REDIRECT;
import static okhttp3.internal.http.StatusLine.HTTP_TEMP_REDIRECT
```
绝大部分状态码来自`java.net.HttpURLConnection`,也有两个是定义在`okhttp3.internal.http.StatusLine`里面。

* HTTP_PROXY_AUTH|407

```java
/**
 * HTTP Status-Code 407: Proxy Authentication Required.
 */
public static final int HTTP_PROXY_AUTH = 407;
```

注释写的很明白了，407是server要求权限，属于鉴权类的。

```java
case HTTP_PROXY_AUTH:
  Proxy selectedProxy = route != null
      ? route.proxy()
      : client.proxy();
  if (selectedProxy.type() != Proxy.Type.HTTP) {
    throw new ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy");
  }
  return client.proxyAuthenticator().authenticate(route, userResponse);
```

* HTTP_UNAUTHORIZED|401

401是鉴权失败，也就是即使客户端的权限不足。

```java
/**
 * HTTP Status-Code 401: Unauthorized.
 */
public static final int HTTP_UNAUTHORIZED = 401;
```

* HTTP_PERM_REDIRECT|308

重定向这一块，有永久重定向308和临时重定向307,处理是一致的：
首先识别是不是GET或者HEAD请求，不是的话返回null。

```java
case HTTP_PERM_REDIRECT:
case HTTP_TEMP_REDIRECT:
  // "If the 307 or 308 status code is received in response to a request other than GET
  // or HEAD, the user agent MUST NOT automatically redirect the request"
  if (!method.equals("GET") && !method.equals("HEAD")) {
    return null;
  }
 // fall-through
```

```java
/** Numeric status code, 307: Temporary Redirect. */
public static final int HTTP_TEMP_REDIRECT = 307;
public static final int HTTP_PERM_REDIRECT = 308;
public static final int HTTP_CONTINUE = 100;
```

* HTTP_MULT_CHOICE|300

包括前面的307，308以及300，301，302，303都贯串统一处理：
如果配置允许重定向，则根据header中的`Location`字段获取新的目标url，构造新的Request。

```java
  // fall-through
case HTTP_MULT_CHOICE:
case HTTP_MOVED_PERM:
case HTTP_MOVED_TEMP:
case HTTP_SEE_OTHER:
  // Does the client allow redirects?
  if (!client.followRedirects()) return null;

  String location = userResponse.header("Location");
  if (location == null) return null;
  HttpUrl url = userResponse.request().url().resolve(location);

  // Don't follow redirects to unsupported protocols.
  if (url == null) return null;

  // If configured, don't follow redirects between SSL and non-SSL.
  boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
  if (!sameScheme && !client.followSslRedirects()) return null;

  // Most redirects don't include a request body.
  Request.Builder requestBuilder = userResponse.request().newBuilder();
  if (HttpMethod.permitsRequestBody(method)) {
    final boolean maintainBody = HttpMethod.redirectsWithBody(method);
    if (HttpMethod.redirectsToGet(method)) {
      requestBuilder.method("GET", null);
    } else {
      RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
      requestBuilder.method(method, requestBody);
    }
    if (!maintainBody) {
      requestBuilder.removeHeader("Transfer-Encoding");
      requestBuilder.removeHeader("Content-Length");
      requestBuilder.removeHeader("Content-Type");
    }
  }

  // When redirecting across hosts, drop all authentication headers. This
  // is potentially annoying to the application layer since they have no
  // way to retain them.
  if (!sameConnection(userResponse, url)) {
    requestBuilder.removeHeader("Authorization");
  }

  return requestBuilder.url(url).build();
```

* HTTP_CLIENT_TIMEOUT|408
最后一种是408,请求超时，从注释上看说是实际很少用到408，不过HAProxy可能会返回这个码。这种情况不需要更换url，直接使用原来的request即可。

```java
case HTTP_CLIENT_TIMEOUT:
  // 408's are rare in practice, but some servers like HAProxy use this response code. The
  // spec says that we may repeat the request without modifications. Modern browsers also
  // repeat the request (even non-idempotent ones.)
  if (!client.retryOnConnectionFailure()) {
    // The application layer has directed us not to retry the request.
    return null;
  }

  if (userResponse.request().body() instanceof UnrepeatableRequestBody) {
    return null;
  }

  if (userResponse.priorResponse() != null
      && userResponse.priorResponse().code() == HTTP_CLIENT_TIMEOUT) {
    // We attempted to retry and got another timeout. Give up.
    return null;
  }

  return userResponse.request();
```