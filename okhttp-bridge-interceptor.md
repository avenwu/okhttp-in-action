![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-01-04-01.png)

## 前言

前面我们分析了第一个拦截器`RetryAndFollowUpInterceptor`，他在`intercept`方法中调用了chain的proceed，从而触发了下一个拦截器的调用，也就是`BridgeInterceptor`，本文将一起分析这个拦截器的作用与实现逻辑。

## BridgeInterceptor

由于命名都是英文单词，如果要给一个中文翻译来表示`BridgeInterceptor`的话，首先要搞明白他的作用:

```java
/**
 * Bridges from application code to network code. First it builds a network request from a user
 * request. Then it proceeds to call the network. Finally it builds a user response from the network
 * response.
 */
public final class BridgeInterceptor implements Interceptor
```

总结一下，起作用就是桥接业务层逻辑到网络层代码。将根据开发者的请求内容，构造一个网络请求，然后调用proceed去请求网络，最后从网络响应构造一个向上传递的用户响应体。
如此，我们姑且称`BridgeInterceptor`为`桥接拦截器`。


![flow](http://7u2jir.com1.z0.glb.clouddn.com/img/BridgeInterceptor-flow.png)


拦截器开始处理后，立刻构造了一份新的Request对象，初始化的赋值和原有Request一致。

```java
Request userRequest = chain.request();
Request.Builder requestBuilder = userRequest.newBuilder();
```

这个模式在`OkHttp`中大量出现，后续也会频繁遇到。既然构造的是相同的Reuqest实例，为什么不直接使用还需要`克隆`一份呢？
这个问题笔者是这样理解的：

>
桥接器所对应的Request是用于网络请求的，克隆一份后，可以对其做新增，删除等操作，而不会影响到开发者上层构造的实例，者可以保证内部逻辑相对独立，表现为一个黑盒状态，上层不需要关注这些header的变更，只需关注业务层的一些参数。

## Header

整个`桥接拦截器`的核心逻辑在于对网络请求`Request`的构造，涉及一些header的添加与删除操作。关于完整的Http协议的头部定义，可以查阅[RFC7231#section-5](https://tools.ietf.org/html/rfc7231#section-5) 。协议的定义内容比较多，而且是因为，简单起见也可以参考[维基百科](https://zh.wikipedia.org/wiki/HTTP%E5%A4%B4%E5%AD%97%E6%AE%B5)


在这里，我们会涉及的头包括：

* Content-Type
* Content-Length
* Transfer-Encoding
* Host
* Connection
* Accept-Encoding
* Cookie
* User-Agent
* Content-Encoding

## 请求体相关Header

一个请求可能带有请求体也可能没有请求体，如果是有请求体的，需要处理`Content-Type`，`Content-Length`，`Transfer-Encoding`来告诉服务端我们的请求体是什么类型，长度多少，编码是什么。

```java
RequestBody body = userRequest.body();
if (body != null) {
  MediaType contentType = body.contentType();
  if (contentType != null) {
    requestBuilder.header("Content-Type", contentType.toString());
  }

  long contentLength = body.contentLength();
  if (contentLength != -1) {
    requestBuilder.header("Content-Length", Long.toString(contentLength));
    requestBuilder.removeHeader("Transfer-Encoding");
  } else {
    requestBuilder.header("Transfer-Encoding", "chunked");
    requestBuilder.removeHeader("Content-Length");
  }
}
```
### Content-Type

根据 [RFC7321#section-3.1.1.5](https://tools.ietf.org/html/rfc7231#section-3.1.1.5) 的定义
`Content-Type`属于`Representation Metadata`，标识性的元数据。当请求包含请求体时，用`Content-Type`来表示它的类型和编码。
例如：

> 
Content-Type: text/html; charset=ISO-8859-4

严格来说客户端只要携带了请求体，都应该正确设置`Content-Type`，一遍服务端解析，但是如果客户端确实不知道数据类型，或者么有设置，后端可能默认将理解为[application/octet-stream](https://tools.ietf.org/html/rfc2046#section-4.5.1)，或者根据内容解析后再判断。

### Content-Length

根据[RFC7230#section-3.3.2](https://tools.ietf.org/html/rfc7230#section-3.3.2)的定义，`Content-Length`属于`Payload Semantics`
以八位字节数组（8位的字节）表示的请求体的长度，代表确切长度，和`Transfer-Encoding`是互斥的。
>
Content-Length: 348

### Transfer-Encoding

用来将实体安全地传输给用户的编码形式,包括：分块（chunked）、compress、deflate、gzip和identity。	

>
Transfer-Encoding: chunked

以下解释摘抄自：[https://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81#%E6%A0%BC%E5%BC%8F](https://zh.wikipedia.org/wiki/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81#%E6%A0%BC%E5%BC%8F)
```
如果一个HTTP消息（请求消息或应答消息）的Transfer-Encoding消息头的值为chunked，那么，消息体由数量未定的块组成，并以最后一个大小为0的块为结束。
每一个非空的块都以该块包含数据的字节数（字节数以十六进制表示）开始，跟随一个CRLF （回车及換行），然后是数据本身，最后块CRLF结束。在一些实现中，块大小和CRLF之间填充有白空格（0x20）。
最后一块是单行，由块大小（0），一些可选的填充白空格，以及CRLF。最后一块不再包含任何数据，但是可以发送可选的尾部，包括消息头字段。
消息最后以CRLF结尾
```

**示例**

```
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

25
This is the data in the first chunk

1C
and this is the second one

3
con

8
sequence

0
```

## Host

请求的域名，可以省略标准端口号，比如默认http是80，https是443

>
Host: en.wikipedia.org:80
Host: en.wikipedia.org

```java
public static String hostHeader(HttpUrl url, boolean includeDefaultPort) {
  String host = url.host().contains(":")
      ? "[" + url.host() + "]"
      : url.host();
  return includeDefaultPort || url.port() != HttpUrl.defaultPort(url.scheme())
      ? host + ":" + url.port()
      : host;
}
```

```java
public static int defaultPort(String scheme) {
  if (scheme.equals("http")) {
    return 80;
  } else if (scheme.equals("https")) {
    return 443;
  } else {
    return -1;
  }
}
```

## Connection

连接类型，这里如果开发者不指定，那么默认设置为`Keep-Alive`,复用连接
[RFC7230#section-6.1](https://tools.ietf.org/html/rfc7230#section-6.1)
>
Connection: Keep-Alive
Connection: Upgrade

## Accept-Encoding

接受的编码类型，常见的是gzip和deflate

* deflate – 基于deflate算法（定义于RFC 1951）的压缩，使用zlib数据格式（RFC 1950）封装
* gzip – GNU zip格式（定义于RFC 1952）。此方法截至2011年3月，是应用程序支持最广泛的方法。[5]

`OkHttp`默认支持gzip在，这里有个限制，如果开发者设置了断点续传的`Range`,那么将不会设置gzip。

```java
// If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
// the transfer stream.
boolean transparentGzip = false;
if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
  transparentGzip = true;
  requestBuilder.header("Accept-Encoding", "gzip");
}
```

## Cookie
浏览器中常说的cookie设置。

>
Cookie: $Version=1; Skin=new;	

```java
List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
if (!cookies.isEmpty()) {
  requestBuilder.header("Cookie", cookieHeader(cookies));
}
```

## User-Agent

客户端标识，比如客户端名称，版本号等。

>
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:12.0) Gecko/20100101 Firefox/21.0


在`OkHttp`中这个字段的取值如下：

```java
public static String userAgent() {
  return "okhttp/${project.version}";
}
```

## 响应gzip处理

前面`Content-Encoding`中提到了请求体的编码，响应体也可以有这个字段，用于客户端解析。

>
注意header是大小写不敏感的，习惯上我们一般首字母大写。


```java
if (transparentGzip
    && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
    && HttpHeaders.hasBody(networkResponse)) {
  GzipSource responseBody = new GzipSource(networkResponse.body().source());
  Headers strippedHeaders = networkResponse.headers().newBuilder()
      .removeAll("Content-Encoding")
      .removeAll("Content-Length")
      .build();
  responseBuilder.headers(strippedHeaders);
  String contentType = networkResponse.header("Content-Type");
  responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
}
```