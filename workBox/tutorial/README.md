### WorkBox

本文会详细介绍workbox的使用方法。

### 引入方式

* 通过importScripts()方法从谷歌官方CDN中引入。

```javascript
importScripts('https://cdn.bootcdn.net/ajax/libs/workbox-sw/6.5.3/workbox-sw.min.js');
if (workbox) {  
    console.log("Workbox已经加载完成");
} else {  
    console.log("Workbox加载失败");
}

```

* 本地引入，本地需要从npm库中下载相应的workbox包，然后通过import按需导入。

```javascript
import {precaching} from 'workbox-precaching';
import {registerRoute} from 'workbox-routing';
import {BackgroundSyncPlugin} from 'workbox-background-sync';

// 按需引入，然后使用对应模块...

```

### 配置

* 配置缓存名称

通过 DevTools -> Applications -> Caches 可以发现，workbox 对于缓存命名有一些规则的：

```
<prefix>-<ID>-<suffix>

```

每个命名都有个前缀和后缀，中间才是真正的命名 ID，主要是为了更大限度的防止重名的情况发生，可以通过以下这种方式分别对 precache 和 runtime 形式的缓存进行自定义命名：

```javascript
workbox.core.setCacheNameDetails({
    prefix: 'my-app',
    suffix: 'v1',
    precache: 'custom-precache-name',// 不设置的话默认值为 'precache'
    runtime: 'custom-runtime-name' // 不设置的话默认值为 'runtime'
});

```

通过以上设置后，precache 类型的缓存名称为 my-app-custom-precache-name-<ID>-v1，runtime 类型的缓存名称为 my-app-custom-runtime-name-<ID>-v1。workbox 推荐尽量为你的每个项目设置不同的 prefix，这样你在本地 locahost 调试 Service Worker 的时候可以避免冲突。而 suffix 可以用来控制缓存版本，让站点的 Service Worker 更新机制变得清晰维护。<br>
workbox 为了让 Web App 的缓存管理的更加细粒度的清晰可维护，也提供了策略级别的缓存命名设置，可以通过策略 API 的 cacheName 参数进行设置：

```javascript
workbox.routing.registerRoute(
    /.*\.(?:png|jpg|jpeg|svg|gif)/g,
    new workbox.strategies.CacheFirst({
        cacheName: 'my-image-cache',
    })
);

```

这样，对应的图片相关的 cacheFirst 策略的缓存都会以 my-image-cache-<ID> 的形式命名，这里要注意的是：prefix 和 suffix 是不需要设置的

* 指定 development 环境
  workbox 开发过程中是需要 debug 的，所以 workbox 也提供了 logger 机制帮助我们排查问题，但是在生产环境下，我们不希望也产生 logger 信息，所以 workbox 提供了「指定当前环境」 的设置：

```javascript
// 设置为开发模式
workbox.setConfig({debug: true});

// 设置为线上生产模式
workbox.setConfig({debug: false}); // 默认展示日志log,此设置log全部删除

```

### workbox 插件

* workbox 插件通常都是在缓存策略中使用的，可以让开发者的缓存策略更加灵活，workbox 内置了一些插件：
* workbox.backgroundSync.Plugin: 如果网络请求失败，就将请求添加到 background sync 队列中，并且在下一个 sync 事件触发之前重新发起请求。
* workbox.broadcastUpdate.Plugin: 当 Cache 缓存更新的时候将会广播一个消息给所有客户端，类似于 sw-register-webpack-plugin 做的事情。
* workbox.cacheableResponse.Plugin: 让匹配的请求的符合开发者指定的条件的返回结果可以被缓存，可以通过 status, headers 参数指定一些规则条件。
* workbox.expiration.Plugin: 管理 Cache 的数量以及 Cache 的时间长短。

  可以像如下方式来使用 workbox 插件，以 workbox.expiration.Plugin 为例：

```javascript
workbox.routing.registerRoute(
    /\.(?:png|gif|jpg|jpeg|svg)$/,
    new workbox.strategies.cacheFirst({
        cacheName: 'images',
        plugins: [
            new workbox.ExpirationPlugin({
                maxEntries: 60, // 最大的缓存数，超过之后则走 LRU 策略清除最老最少使用缓存
                maxAgeSeconds: 30 * 24 * 60 * 60, // 这只最长缓存时间为 30 天
            }),
        ],
    }),
);
```

### 预缓存功能
预缓存功能可以在service worker安装前将一些静态文件提前缓存下来，这样就能保证service worker安装后可以直接存缓存中获取这些静态资源，可以通过以下代码实现。

```
workbox.precaching.precacheAndRoute([  
    {url: '/index.html', revision: '383676' }, 
    {url: '/styles/app.0c9a31.css', revision: null},
    {url: '/scripts/app.0d5770.js', revision: null},
]);

```

### 路由功能
路由功能是workbx的核心功能，主要是匹配资源路径后，采用相应的缓存策略，或者自定义缓存处理，使用方法如下所示：

```
import {registerRoute} from 'workbox-routing';
import {CacheFirst} from 'workbox-strategies';

registerRoute(  /\.(?:png|jpg|jpeg|svg|gif)$/,  new CacheFirst({    
    cacheName: 'my-image-cache',   
}));
```

`registerRoute`有两个参数，第一个参数是一个正则表达式，用来匹配路径，第二个参数是对匹配路径进行的处理函数，可以用`workbox`封装的缓存策略处理函数，也可以自定义，上述示例就是使用的`workbox`内部封装的`CacheFirst`缓存策略。<br>

如果第二个参数使用自定义函数，那么这个函数有三个默认参数，下面缓存策略自定义策略有示例。


### 缓存策略

Workbox内部封装了以下五种缓存策略：

* 1、NetworkFirst：网络优先
  这种策略就是当请求路由是被匹配的，就采用网络优先的策略，也就是优先尝试拿到网络请求的返回结果，如果拿到网络请求的结果，就将结果返回给客户端并且写入 Cache 缓存，如果网络请求失败，那最后被缓存的 Cache 缓存结果就会被返回到客户端，这种策略一般适用于返回结果不太固定或对实时性有要求的请求，为网络请求失败进行兜底。可以像如下方式使用 Network First 策略：

```javascript
workbox.routing.registerRoute(
    match, // 匹配的路由
    new workbox.strategies.NetworkFirst()
);
```

* 2、CacheFirst：缓存优先
  这个策略的意思就是当匹配到请求之后直接从 Cache 缓存中取得结果，如果 Cache 缓存中没有结果，那就会发起网络请求，拿到网络请求结果并将结果更新至 Cache 缓存，并将结果返回给客户端。这种策略比较适合结果不怎么变动且对实时性要求不高的请求。可以像如下方式使用 Cache First 策略：

```javascript
workbox.routing.registerRoute(
    match, // 匹配的路由
    new workbox.strategies.CacheFirst()
);
```

* 3、NetworkOnly：仅使用正常的网络请求
  比较直接的策略，直接强制使用正常的网络请求，并将结果返回给客户端，这种策略比较适合对实时性要求非常高的请求。可以像如下方式使用 Network Only 策略：

```javascript
workbox.routing.registerRoute(
    match, // 匹配的路由
    new workbox.strategies.NetworkOnly()
);
```

* 4、CacheOnly：仅使用缓存中的资源
  这个策略也比较直接，直接使用 Cache 缓存的结果，并将结果返回给客户端，这种策略比较适合一上线就不会变的静态资源请求。可以像如下方式使用 Cache Only 策略：

```javascript
workbox.routing.registerRoute(
    match, // 匹配的路由
    new workbox.strategies.CacheOnly()
);

```

* 5、StaleWhileRevalidate：从缓存中读取资源的同时发送网络请求更新本地缓存
  这种策略的意思是当请求的路由有对应的 Cache 缓存结果就直接返回，在返回 Cache 缓存结果的同时会在后台发起网络请求拿到请求结果并更新 Cache 缓存，如果本来就没有 Cache 缓存的话，直接就发起网络请求并返回结果，这对用户来说是一种非常安全的策略，能保证用户最快速的拿到请求的结果，但是也有一定的缺点，就是还是会有网络请求占用了用户的网络带宽。可以像如下的方式使用 State While Revalidate 策略：

```javascript
workbox.routing.registerRoute(
    match, // 匹配的路由
    new workbox.strategies.StaleWhileRevalidate()
);
```

* 自定义策略
  如果以上的那些策略都不太能满足你的请求的缓存需求，那就得想想办法自己定制一个合适的策略，甚至是不同情况下返回不同的请求结果，workbox 也考虑到了这种场景，当然，最简单的方法是直接在 Service Worker 文件里通过最原始的 fetch 事件控制缓存策略。也可以使用 workbox 提供的另一种方式：传入一个带有对象参数的回调函数，对象中包含匹配的 url 以及请求的 fetchEvent 参数，回调函数返回的就是一个 response 对象，具体用法如下所示：

```javascript
workbox.routing.registerRoute(
    ({url, event}) => {
        return {
            name: 'workbox',
            type: 'guide',
        };
    },
    ({url, event, params}) => {
        // 返回的结果是：A guide on workbox
        return new Response(
            `A ${params.type} on ${params.name}`
        );
    }
);

```

无论使用何种策略，你都可以通过自定义一个缓存来使用或添加插件来定制路由的行为（以何种方式返回结果）。<br>
五种缓存策略使用方法一致，各适用于不同的场景，具体适用场景可在[离线指南](https://link.juejin.cn/?target=https%3A%2F%2Fdevelopers.google.cn%2Fweb%2Ffundamentals%2Finstant-and-offline%2Foffline-cookbook%2F%3Fhl%3Dzh_cn%23serving-suggestions)中查看。<br>

### 第三方请求的缓存
如果有些请求的域和当前 Web 站点不一致，那可以被认为是第三方资源或请求，针对第三方请求的缓存，因为 Workbox 无法获取第三方路由请求的状态，当请求失败的情况下 workbox 也只能选择缓存错误的结果，所以 workbox 原则上默认不会缓存第三方请求的返回结果。也就是说，默认情况下如下的缓存策略是不会生效的：<br>

当然，并不是所有的策略在第三方请求上都不能使用，workbox 可以允许 networkFirst 和 stalteWhileRevalidate 缓存策略生效，因为这些策略会有规律的更新缓存的返回内容，毕竟每次请求后都会更新缓存内容，要比直接缓存安全的多。<br>
```javascript
workbox.routing.registerRoute(
    'https://notzoumiaojiang.com/example-script.min.js', 
    new workbox.strategies.NetworkFirst(),
);
```

对于设置了 CORS 的跨域请求的图片资源，可以通过配置 fetchOptions 将策略中 Fetch API 的请求模式设置为 cors：

```javascript
workbox.routing.registerRoute(
  /^https:\/\/third-party-site\.com\/.*\.(jpe?g|png)/,
  new workbox.strategies.CacheFirst({
    fetchOptions: {
      mode: 'cors'
    },
    matchOptions: {
      ignoreSearch: true // 假设图片资源缓存的存取需要忽略请求 URL 的 search 参数，可以通过设置 matchOptions 来实现
    }
  })
)

```


### 删除缓存

* 删除过期缓存
  每次进入页面，都需要删除掉已经过期的缓存，以免用户看到的还是旧的数据。

```javascript
  workbox.precaching.cleanupOutdatedCaches();

```

* 删除所有缓存
  删除所有缓存，直接使用cache,客户端自带的属性。

```javascript
caches.keys().then(function (cacheList) {
    return Promise.all(
      cacheList.map(function (cacheName) {
        return caches.delete(cacheName);
      })
    );
});

```
