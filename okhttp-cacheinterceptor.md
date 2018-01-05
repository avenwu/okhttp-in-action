![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-01-04-01.png)

## 前言

前文分析过程中，我们知道`BridgeInterceptor`桥接拦截器调用过程中，将触发缓存拦截逻辑`CahceInterceptor`，本文一起缓存拦截器的实现逻辑。

## CacheInterceptor

谈到缓存，笔者的第一反应就是`OkHttp`默认是开启缓存的么？缓存在什么位置？

```java
interceptors.add(new CacheInterceptor(client.internalCache()));
```
通过查阅拦截器的构造，可以知道，实际上缓存默认是关闭的，也就是说开发主可选着性配置缓存，配置的时候就可以指定缓存目录，不配置则不缓存。

查看`OkHttpClient#Builder`会发现有两个同名方法可以设置cache，两者接受的参数类型不同。实际上我们应当使用的是`public Builder cache(@Nullable Cache cache)`:

```java
/** Sets the response cache to be used to read and write cached responses. */
void setInternalCache(@Nullable InternalCache internalCache) {
  this.internalCache = internalCache;
  this.cache = null;
}

/** Sets the response cache to be used to read and write cached responses. */
public Builder cache(@Nullable Cache cache) {
  this.cache = cache;
  this.internalCache = null;
  return this;
}
```

缓存配置比较简单，只需要实例化一个`Cache`即可，但是对缓存是使用和相关头部介绍，还是比较复杂的。

```java
new OkHttpClient.Builder()
    .cache(new Cache(context.getCacheDir(), 10 * 1024 * 1024))
    .build();		
```

## 缓存检测逻辑

接下来看看拦截器里面的具体逻辑。首先根据请求尝试获取缓存的响应。

```java
Response cacheCandidate = cache != null
    ? cache.get(chain.request())
    : null;
```

在Cache中生成url的MD5消息摘要，然后以十六进制的字符串作为缓存的key，磁盘缓存用的是LRU，相关实现在`DiskLruCache`中。

```java
public static String key(HttpUrl url) {
  return ByteString.encodeUtf8(url.toString()).md5().hex();
}
```

得到缓存中的Response（可能为空）后，结合Request，构造一个缓存策略的辅助类`CacheStrategy`。这个类主要是根据Request的各种配置，比如header信息，和缓存配置信息来构造的，因此可以看到里面有很对对Request的判断。

```java
/** Returns a strategy to use assuming the request can use the network. */
private CacheStrategy getCandidate() {
  // No cached response.
  if (cacheResponse == null) {
    return new CacheStrategy(request, null);
  }

  // Drop the cached response if it's missing a required handshake.
  if (request.isHttps() && cacheResponse.handshake() == null) {
    return new CacheStrategy(request, null);
  }

  // If this response shouldn't have been stored, it should never be used
  // as a response source. This check should be redundant as long as the
  // persistence store is well-behaved and the rules are constant.
  if (!isCacheable(cacheResponse, request)) {
    return new CacheStrategy(request, null);
  }

  CacheControl requestCaching = request.cacheControl();
  if (requestCaching.noCache() || hasConditions(request)) {
    return new CacheStrategy(request, null);
  }

  CacheControl responseCaching = cacheResponse.cacheControl();
  if (responseCaching.immutable()) {
    return new CacheStrategy(null, cacheResponse);
  }

  long ageMillis = cacheResponseAge();
  long freshMillis = computeFreshnessLifetime();

  if (requestCaching.maxAgeSeconds() != -1) {
    freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
  }

  long minFreshMillis = 0;
  if (requestCaching.minFreshSeconds() != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
  }

  long maxStaleMillis = 0;
  if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
  }

  if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    Response.Builder builder = cacheResponse.newBuilder();
    if (ageMillis + minFreshMillis >= freshMillis) {
      builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
    }
    long oneDayMillis = 24 * 60 * 60 * 1000L;
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
      builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
    }
    return new CacheStrategy(null, builder.build());
  }

  // Find a condition to add to the request. If the condition is satisfied, the response body
  // will not be transmitted.
  String conditionName;
  String conditionValue;
  if (etag != null) {
    conditionName = "If-None-Match";
    conditionValue = etag;
  } else if (lastModified != null) {
    conditionName = "If-Modified-Since";
    conditionValue = lastModifiedString;
  } else if (servedDate != null) {
    conditionName = "If-Modified-Since";
    conditionValue = servedDateString;
  } else {
    return new CacheStrategy(request, null); // No condition! Make a regular request.
  }

  Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
  Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

  Request conditionalRequest = request.newBuilder()
      .headers(conditionalRequestHeaders.build())
      .build();
  return new CacheStrategy(conditionalRequest, cacheResponse);
}
```
上面的逻辑比较多，可以自行阅读。接着看拦截器中的逻辑。

```java
CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
Request networkRequest = strategy.networkRequest;
Response cacheResponse = strategy.cacheResponse;
```
在`CacheStrategy`构造完之后，可以获取networkRequest和cacheResponse。

两者均为空，则返回504的响应：

```java
// If we're forbidden from using the network and the cache is insufficient, fail.
if (networkRequest == null && cacheResponse == null) {
  return new Response.Builder()
      .request(chain.request())
      .protocol(Protocol.HTTP_1_1)
      .code(504)
      .message("Unsatisfiable Request (only-if-cached)")
      .body(Util.EMPTY_RESPONSE)
      .sentRequestAtMillis(-1L)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();
}
```

只有网络请求Request为空，则命中了缓存，直接返回。
```java
// If we don't need the network, we're done.
if (networkRequest == null) {
  return cacheResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .build();
}
```

网络请求Request不为空，继续走下一个拦截器的业务逻辑去联网请求。

```java
Response networkResponse = null;
try {
  networkResponse = chain.proceed(networkRequest);
} finally {
  // If we're crashing on I/O or otherwise, don't leak the cache body.
  if (networkResponse == null && cacheCandidate != null) {
    closeQuietly(cacheCandidate.body());
  }
}
```

网络请求回来后，可能出现304 `HTTP_NOT_MODIFIED`，那么使用不为空的缓存结果。

最后将缓存response和网络response构造为一个新的response，并且将数据缓存，清除非GET请求的缓存。

```java
Response response = networkResponse.newBuilder()
    .cacheResponse(stripBody(cacheResponse))
    .networkResponse(stripBody(networkResponse))
    .build();

if (cache != null) {
  if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
    // Offer this request to the cache.
    CacheRequest cacheRequest = cache.put(response);
    return cacheWritingResponse(cacheRequest, response);
  }

  if (HttpMethod.invalidatesCache(networkRequest.method())) {
    try {
      cache.remove(networkRequest);
    } catch (IOException ignored) {
      // The cache cannot be written.
    }
  }
}
```
