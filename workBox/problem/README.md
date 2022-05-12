### 问题

本文会总结出本人在使用service worker遇到的问题（附解决办法）以及需要注意的点。

### 注意点
* service worker 是基于 HTTPS 的，因为service worker中涉及到请求拦截，所以必须使用HTTPS协议来保障安全。如果是本地调试的话，localhost也是可以的。
* service worker 是一个JavaScript worker ,所以它不能直接访问 DOM 。相反, service worker 可以通过postMessage 接口与跟其相关的页面进行通信,发送消息,从而让这些页面在有需要的时候去操纵 DOM 。
* Service worker 是一个可编程的网络代理，允许你去控制如何处理页面的网络请求。
* Service worker 在不使用时将被终止，并会在需要的时候重新启动，因此你不能把onfetch 和 onmessage事件来作为全局依赖处理程序。如果你需要持久话一些信息并在重新启动Service worker后使用他，可以使用 IndexedDBAPI ，service worker 支持。
* Service worker 广泛使用了 promise ，所以如果不熟悉 promise ，就先去了解[promise](https://es6.ruanyifeng.com/#docs/promise)。


### service-worker更新
问题描述：<br>
* service-worker.js也会受http的缓存策略控制
* 如果新的worker未被成功下载，或者解析错误，或者在运行时出错，或者在安装阶段不成功，新的worker会被丢弃，旧的会被保留
* 一旦新的worker被成功安装，更新的worker会进入等待状态，新的worker会等待旧的worker下线才会激活，新的worker和旧的会并存
* self.skipWaiting()会强制跳过等待状态，直接让新的worker在安装后进入激活状态，这样可能会有缓存问题
* 浏览器会 diff 当前打开页面的 service-worker.js，并判断是否更新，如果 diff 结果为更新，则重新安装最新的 service-worker.js，并且全量更新缓存
* 任何静态资源包括 service-worker.js 都会被 HTTP 缓存
* 服务器对某个资源进行`no-cache`设置可以避免 HTTP 缓存


解决办法：<br>
针对上述的情况，service-worker的更新就是必须解决的问题。
1. 在服务器端配置service-worker的header，Cache-control: no-cache，使其不被缓存
2. 前端进行service-worker的版本控制，每次注册都添加版本号进行改写

```javascript
// 版本控制
process.env.CACHE_VERSION = 'v1.0.0'
const cacheVersion =
  process.env.NODE_ENV === "production"
    ? process.env.CACHE_VERSION
    : Date.now();

if ("serviceWorker" in navigator) {
  register(
    `${process.env.BASE_URL}service-worker.js?cacheVersion=${cacheVersion}`,
    {
      ready() {
        console.log(
          "service worker正在从缓存中为app提供服务.\n" +
            "查看更多, 访问 https://goo.gl/AFskqB"
        );
      },
    }
  );
}
```

