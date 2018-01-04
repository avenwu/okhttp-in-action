![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-01-02-01.png)

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

