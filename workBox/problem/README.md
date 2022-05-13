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
* 浏览器会 diff 当前打开页面的 service-worker.js，并判断是否更新，如果 diff 结果为更新，则重新安装最新的 service-wroker.js，并且全量更新缓存
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

### service-worker激活
问题描述：<br>
由于浏览器的内部实现原理，当页面切换或者自身刷新时，浏览器是等到新的页面完成渲染之后再销毁旧的页面。这表示新旧两个页面中间有共同存在的交叉时间，因此简单的切换页面或者刷新是不能使得 service worker 进行更新的。

解决方案：<br>
既然service-worker的激活无法通过刷新解决，那么还有个skipWaiting可以用。<br>
但是最好不要直接skipWaiting(跳过等待阶段)， 推荐的做法应该是在浏览器发现更新后，给用户弹出提示。然后用户点击重新加载时，一方面刷新页面 (location.reload())，一方面让新的 SW 接管页面 (skipWaiting)。

具体流程：<br>
1. 在注册service-worker时就监听sw的更新状况，
2. 如果有更新，显示更新按钮
3. 用户点击更新按钮发送事件
4. sw.js文件中，监听发送的事件，来更新sw

示例：
```javascript
  register(
    `${process.env.BASE_URL}service-worker.js?cacheVersion=${cacheVersion}`,
    {
      ready() {
        console.log(
          "service worker正在从缓存中为app提供服务.\n" +
            "查看更多, 访问 https://goo.gl/AFskqB"
        );
      },
      // 第一步: 此处监听sw的更新状况
      updated(registration) {
        const worker = registration.waiting;
        if (worker) {
          // 第二步: 如果有更新，显示更新按钮
          Dialog.confirm({
            title: "提示",
            message: "有新内容可用；请刷新。",
          }).then(() => {
            // 第三步：用户点击更新会进入这里，触发更新操作
            navigator.serviceWorker
              .getRegistration()
              .then(() => {
                skipWaiting(registration); // skipWaiting 代码在下方
              })
              .then(() => {
                window.location.reload();
              });
          });
        }
      }
    }
  )

// 第四步：sw.js文件中，监听事件，来更新sw。下面的self就是sw实例。
self.addEventListener('message', event => {
  const replyPort = event.ports[0]
  const message = event.data
  if (replyPort && message && message.type === 'skip-waiting') {
    event.waitUntil(
      self.skipWaiting()
        .then(() => replyPort.postMessage({ error: null }))
        .catch(error => replyPort.postMessage({ error }))
    )
  }
})

function skipWaiting(registration: any) {
  const worker = registration.waiting;
  if (!worker) {
    return Promise.resolve();
  }
  // 这里是参考vue-press的写法
  // 利用MessageChannel返回一个promise
  return new Promise((resolve, reject) => {
    const channel = new MessageChannel();
    channel.port1.onmessage = (event) => {
      if (event.data.error) {
        reject(event.data.error);
      } else {
        resolve(event.data);
      }
    };
    worker.postMessage({ type: "skip-waiting" }, [channel.port2]);
  });
}

```

### 当service worker安装失败怎么处理
问题描述：<br>
当service-worker新版本的更新出现问题，如何保证用户看到的版本是最新的。
解决办法：<br>
卸载当前的sw，用线上的文件，并且不再安装当前错误版本的。


### 离线浏览无效果
问题描述：<br>
当断网的时候发现没法浏览，还是显示未连接到互联网。

解决办法：<br>
经过我的摸索实验发现，vue这种单页面项目，必须要预缓存 index.html，不然就没有离线浏览的效果。<br>
(因为vue这种单页面应用，主出口就是index.html，如果没有缓存index.html，其他的js根本没法使用。)

```javascript
// 预缓存index.html
workbox.precaching.precacheAndRoute([
    {
      url: "/index.html",
      revision: "1.0.0",
    },
]);

```

### 删除sw缓存后，页面不停刷新
问题描述：<br>
本人在调试的时候，删除了Cache Storage里面的所有内容，然后刷新页面，发现页面不停刷新，不会进入到首页。

解决办法：<br>
排查发现，以前缓存js css用的cacheFirst缓存策略，导致删除之后，页面去缓存里找js等文件，一直找不到，就出现你不停刷新的bug,之后我改为staleWhileRevalidate(或者 networkFirst)缓存策略后，就解决了。
上面说的缓存策略，在workbox使用栏有详细介绍，可前往查阅。

```javascript
workbox.routing.registerRoute(
  /.*.(?:js|css|json|ico)/,
  new workbox.strategies.StaleWhileRevalidate({ // 此处改为使用StaleWhileRevalidate缓存策略
    cacheName: "js-css-json-ico-caches",
    plugins: [
      // 需要缓存的状态筛选
      new workbox.cacheableResponse.CacheableResponsePlugin({
        statuses: [0, 200, 304],
      }),
      // 缓存时间
      new workbox.expiration.ExpirationPlugin({
        maxEntries: 20,
        maxAgeSeconds: 60 * 60,
      }),
    ],
  })
);

```
