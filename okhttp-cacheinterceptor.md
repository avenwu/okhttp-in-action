![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-01-04-01.png)

## 前言

前文分析过程中，我们知道`BridgeInterceptor`桥接拦截器调用过程中，将触发缓存拦截逻辑`CahceInterceptor`，本文一起缓存拦截器的实现逻辑。

## CacheInterceptor

谈到缓存，笔者的第一反应就是`OkHttp`默认是开启缓存的么？缓存在什么位置？

```java
interceptors.add(new CacheInterceptor(client.internalCache()));
```
通过查阅拦截器的构造，可以知道，实际上缓存默认是未开启的，也就是所开发主需要主动配置缓存才会生效。

查看`OkHttpClient`的接口会发现有两个同名方法可以设置cache，不过接受的参数类型不同，实际上我们应当使用的是`public Builder cache(@Nullable Cache cache)`:

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
