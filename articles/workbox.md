# 离线缓存优化


将应用中的静态资源缓存是目前最主流的性能优化方法，甚至能让应用秒开！目前常见的缓存方式有`http缓存`、`memory cache`、`disk cahce`、`localstorage`、`Service worker缓存`等方式，本文介绍的Workbox就是实现Service worker离线缓存的一个工具。

那么问题来了，Service worker离线缓存和传统的缓存方式对比，有什么优势和劣势呢，service worker之所以越来越流行，是因为它让前端缓存脱离了服务端，不需要服务端再额外做些什么，前端工程师自己就可以实现缓存，而且**缓存内容完全可控**，下面是我搜索的几条主流缓存方式的介绍和对比。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/2/18/170585ae9234f008~tplv-t2oaga2asx-image.image)

上图从[深入理解浏览器缓存](https://www.jianshu.com/p/54cc04190252)处参考

http缓存依赖于服务端配置，memory cache和disk cache缓存内容不可控，而且只缓存一些静态资源，push cache是临时缓存，localstorage适用于缓存一些全局的数据，对于静态资源很少用它。

**service worker缓存的优缺点：**

优点：

* 非侵入式缓存
* 缓存内容开发者完全可控
* 持续性缓存
* 独立于主线程之外，不堵塞进程


缺点：

* 权限太大，能拦截所有fetch请求，需要控制一下
* 发版更新处理比较麻烦

# Workbox简介

Workbox 是 Google Chrome 团队推出的一套 PWA 的解决方案，这套解决方案当中包含了核心库和构建工具，因此我们可以利用 Workbox 实现 [Service Worker](https://developers.google.cn/web/fundamentals/primers/service-workers?hl=zh_cn) 的快速开发。

## 引入方式

有两种方式可以引入workbox：

第一种最为方便，就是通过`importScripts()`方法从谷歌官方CDN中引入。

```js
importScripts('https://storage.googleapis.com/workbox-cdn/releases/5.0.0/workbox-sw.js');
if (workbox) {  
    console.log(`Yay! Workbox is loaded 🎉`);
} else {  
    console.log(`Boo! Workbox didn't load 😬`);
}
```

第二种方式就是从本地引入，本地需要从npm库中下载相应的workbox包，然后通过`import`按需导入，本文的例子就是这种方式。

```js
import {precaching} from 'workbox-precaching';
import {registerRoute} from 'workbox-routing';
import {BackgroundSyncPlugin} from 'workbox-background-sync';

// 按需引入，然后使用对应模块...
```

详细介绍请查阅[官方文档](https://developers.google.cn/web/fundamentals/primers/service-workers?hl=zh_cn)

## 配置

Workbox可以修改缓存名称，可以用`setCacheNameDetails`设置预缓存和运行时缓存的名称，还可以通过`workbox.core.cacheNames.precache` 和 `workbox.core.cacheNames.runtime` 获取当前定义的预缓存和动态缓存名称。

```js
// 修改默认配置
workbox.core.setCacheNameDetails({
  prefix: 'app',
  suffix: 'v1',
  precache: 'precache',
  runtime: 'runtime'
})

// 打印修改结果

// 将打印 'app-precache-v1'
console.log(worbox.core.cacheNames.precache)
// 将打印 'app-runtime-v1'
console.log(workbox.core.cacheNames.runtime)
```

更多配置下信息请参考[官方文配置文档](https://developers.google.cn/web/tools/workbox/guides/configure-workbox?hl=zh_cn)

## 预缓存功能

预缓存功能可以在service worker安装前将一些静态文件提前缓存下来，这样就能保证service worker安装后可以直接存缓存中获取这些静态资源，可以通过以下代码实现。

```js
import {precacheAndRoute} from 'workbox-precaching';

precacheAndRoute([  
    {url: '/index.html', revision: '383676' }, 
    {url: '/styles/app.0c9a31.css', revision: null},
    {url: '/scripts/app.0d5770.js', revision: null},
]);
```

更多预缓存请参考[官方预缓存功能文档](https://developers.google.cn/web/tools/workbox/guides/configure-workbox?hl=zh_cn)

## 路由功能

路由功能是workbx的核心功能，主要是匹配资源路径后，采用相应的缓存策略，或者自定义缓存处理，使用方法如下所示：   

```js
import {registerRoute} from 'workbox-routing';
import {CacheFirst} from 'workbox-strategies';

registerRoute(  /\.(?:png|jpg|jpeg|svg|gif)$/,  new CacheFirst({    
    cacheName: 'my-image-cache',   
}));
```
    
registerRoute有两个参数，第一个参数是一个正则表达式，用来匹配路径，第二个参数是对匹配路径进行的处理函数，可以用workbox封装的缓存策略处理函数，也可以自定义，上述示例就是使用的workbox内部封装的`CacheFirst`缓存策略。

如果第二个参数使用自定义函数，那么这个函数有三个默认参数，示例如下：

```js
import {registerRoute} from 'workbox-routing';

const handler = async ({url, event, params}) => {   
    // Response will be "A guide to Workbox"  
    return new Response(`A ${params.type} to ${params.name}` );
};
registerRoute(/\.css$/, handler);
```

## 缓存策略

Workbox内部封装了以下五种缓存策略：

* NetworkFirst：网络优先
* CacheFirst：缓存优先
* NetworkOnly：仅使用正常的网络请求
* CacheOnly：仅使用缓存中的资源
* StaleWhileRevalidate：从缓存中读取资源的同时发送网络请求更新本地缓存

五种缓存策略使用方法一致，各适用于不同的场景，具体适用场景可在[离线指南](https://developers.google.cn/web/fundamentals/instant-and-offline/offline-cookbook/?hl=zh_cn#serving-suggestions)中查看。

# Webpack+Workbox构建离线应用
  
目前大部分前端项目都离不开webpack，为了方便我们使用workbox，谷歌官方给我们提供了workbox的webpack插件，通过这个插件，我们能在项目中快速引入workbox，通过配置来定制化我们的缓存。

通过以下四个步骤，我们能将webpack引入到一个由webpack构建的应用中并实现缓存。
  
## 第一步：使用workbox-webpack-plugin 

1. **安装**

```js
npm install workbox-webpack-plugin
```

2. **在webpack 配置文件中引入并配置**

workbox-webpack-plugin有两种配置方式：

**第一种：GenerateSW**

通过配置自动在项目中引入service-worker.js，代码如下：

```js
const WorkboxPlugin = require('workbox-webpack-plugin');

module.exports = {  
    // Other webpack config...  
    plugins: [    
        // Other plugins...
        new WorkboxPlugin.GenerateSW({ 
            // Do not precache images
            exclude: [/\.(?:png|jpg|jpeg|svg)$/],
            // Define runtime caching rules.
            runtimeCaching: [{        
                // Match any request that ends with .png, .jpg, .jpeg or .svg.       
                 urlPattern: /\.(?:png|jpg|jpeg|svg)$/,        
                // Apply a cache-first strategy.        
                handler: 'CacheFirst',        
                options: {         
                     // Use a custom cache name.          
                    cacheName: 'images',          
                    // Only cache 10 images.          
                    expiration: {            
                        maxEntries: 10,          
                    },       
                },      
            }],    
        })  
    ]
};
```

**适用于：**

* 预缓存静态文件
* 只简单的应用运行时的缓存功能


**不适用：**

* 需要使用service worker 其他功能的场景，如push等
* 需要在service worker中导入其他脚本或添加其他逻辑
* 具体的配置文件可查阅官方文档，本文示例使用的是第二种方式

**第二种：InjectManifest**

通过已有的service-worker.js文件生成新的service-worker.js，示例如下：

```js
new workboxPlugin.InjectManifest({  
    // 目前的service worker 文件
    swSrc: './src/sw.js',  
    // 打包后生成的service worker文件，一般存到disk目录 
    swDest: 'sw.js'
})
```

**适用于：**

* 预缓存文件
* 更多的定制化缓存需求
* 使用service worker其他特性

如果你只想简单的引入service worker，建议使用第一种方式

## 第二步：注册Service Worker

配置好插件之后，我们需要在项目中注册service worker。

注册比较简单，只需要在项目入口文件中进行注册即可，代码如下：

```js
// Check that service workers are supported
if ('serviceWorker' in navigator) { 
    // Use the window load event to keep the page load performant  
    window.addEventListener('load', () => {    
        navigator.serviceWorker.register('/sw.js');  
    });
}
```

上述代码是最简单的注册方式，在我们的项目中我们使用`register-service-worker`npm包注册service worker并添加一下自定义事件，方便后期进行更新和离线事件的处理。

代码如下：

```js
import { register } from 'register-service-worker';

export default function(swDest) {
    console.log("注册sw")
    register(`/${swDest}`, {
        ready(registration) {
            // 此方法是我们项目中自己封装的创建自定义事件的公共方法
            dispatchServiceWorkerEvent('sw.ready', registration);
        },
        registered(registration) {
            dispatchServiceWorkerEvent('sw.registered', registration);
        },
        cached(registration) {
            dispatchServiceWorkerEvent('sw.cached', registration);
        },
        updatefound(registration) {
            dispatchServiceWorkerEvent('sw.updatefound', registration);
        },
        updated(registration) {
            dispatchServiceWorkerEvent('sw.updated', registration);
        },
        offline() {
            dispatchServiceWorkerEvent('sw.offline', {});
        },
        error(error) {
            dispatchServiceWorkerEvent('sw.error', error);
        },
    });
}
```

在入口文件中引入注册文件：

```js
import registerSW from './sw-register';
registerSW('sw.js');
```

## 第三步：自定义Service Worker缓存配置

如果我们使用injectMainfest的方式引入servicce worker，需要在src目录下创建一个sw.js(命名自定义，但需要和webpack配置中一致)，在这个文件中我们可以进行预缓存等操作。代码示例如下（sw.js）：

```js
import { setCacheNameDetails, clientsClaim } from 'workbox-core';
import { precacheAndRoute } from 'workbox-precaching/precacheAndRoute';
import {createHandlerBoundToURL} from 'workbox-precaching';
import {NavigationRoute, registerRoute} from 'workbox-routing';
import {StaleWhileRevalidate,NetworkOnly} from 'workbox-strategies';

// 设置缓存名称
setCacheNameDetails({
    prefix: 'app',
    suffix: 'v1.0.2',
});

// 更新时自动生效
clientsClaim();

// 预缓存文件，self.__WB_MANIFEST是workbox生成的文件地址数组，项目中打包生成的所有静态文件都会自动添加到里面
precacheAndRoute(self.__WB_MANIFEST || []);

// 单页应用需要应用NavigationRoute进行缓存，此处可自定义白名单和黑名单
// 跳过登录和退出页面的拦截
const handler = createHandlerBoundToURL('/index.html');
const navigationRoute = new NavigationRoute(handler, {
  denylist: [
    /login/,
    /logout/,
  ],
});
registerRoute(navigationRoute);
// 运行时缓存配置
// 接口数据使用服务端数据
registerRoute(/^api/,new NetworkOnly());

//图片cdn地址，属于跨域资源，我们使用StaleWhileRevalidate缓存策略
registerRoute(/^https:\/\/img.xxx.com\//,new StaleWhileRevalidate());
```

上述的代码有一段针对单页应用处理的逻辑，应为单页应用依靠路由变化来加载不同的内容，使用`navigationRoute`可以匹配导航请求，从而从换从中加载index.html，但默认情况会拦截所有导航请求，如果需要控制，可以在方法中添加白名单和黑名单加以控制。

## 第四步：处理Service Worker的更新和离线状态

**更新状态**

配置完成后，我们需要注意service worker的更新和离线状态，service worker的更新较为复杂，如果处理不当回引发各种问题，目前主流的方式就是每次发版，提醒用户更新，如果用户点击确定更新，新发版的service worker会替换掉旧的service worker，此代码我们项目中放在了入口文件中（webpack配置的入口文件），示例代码如下：

```js
// sw.updated是在注册文件中添加的自定义事件
window.addEventListener('sw.updated', event => {
    const e = event;

    const reloadSW = async () => {
      // Check if there is sw whose state is waiting in ServiceWorkerRegistration
      // https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration
      const worker = e.detail && e.detail.waiting;

      if (!worker) {
        return true;
      } // Send skip-waiting event to waiting SW with MessageChannel

      await new Promise((resolve, reject) => {
        const channel = new MessageChannel();

        channel.port1.onmessage = msgEvent => {
          if (msgEvent.data.error) {
            reject(msgEvent.data.error);
          } else {
            resolve(msgEvent.data);
          }
        };

        worker.postMessage(
          {
            type: 'skip-waiting',
          },
          [channel.port2],
        );
      }); 
      // Refresh current page to use the updated HTML and other assets after SW has skiped waiting

      window.location.reload(true);
      return true;
    };

    const key = `open${Date.now()}`;
    const btn = (
      <Button
        type="primary"
        onClick={() => {
          notification.close(key);
          reloadSW();
        }}
      >
        确定
      </Button>
    );
    notification.open({
      message: "新版本发布",
      description: "Boss发布新版本了，请点击页面更新",
      btn,
      key,
      onClose: async () => {},
    });
});
```

对应sw.js文件里面要监听主线程传递过来的更新事件，代码如下：

```js
// service worker通过message和主线程通讯
addEventListener('message', event => {
    const replyPort = event.ports[0];
    const message = event.data;
    console.log(message)
    if (replyPort && message && message.type === 'skip-waiting') {
      event.waitUntil(
        self.skipWaiting().then(
          () =>
            replyPort.postMessage({
              error: null,
            }),
          error =>
            replyPort.postMessage({
              error,
            }),
        ),
      );
    }
});
```

**离线状态**

对于离线状态的监听比较简单，在入口文件中添加以下代码即可：

```js
window.addEventListener('sw.offline', () => {
    message.warning("当前处于离线状态",0);
});
```

# 检查效果

经过上述四个步骤，我们就能将service worker引入到我们已有的用webpack构建的项目上。

如果正常引入，我们可以在控制台中看到下图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/2/18/1705885a5c973ac0~tplv-t2oaga2asx-image.image)

# 总结
* **service worker实现缓存有非侵入、持久化、缓存内容可控等优点**
* **Workbox可以理解为service worker的库，利用它可以快速进行service worker开发**
* **通过workbox-webpack-plugin可以将workbox引入到现有的用webpack构建的项目中**

本文对workbox的接口的解释较少，需要各位去官网查阅[api](https://developers.google.cn/web/tools/workbox/reference-docs/latest?hl=zh_cn)。

# 参考文献
* [Workbox官方文档](https://developers.google.cn/web/tools/workbox?hl=zh_cn)
* [深入理解浏览器的缓存机制](https://www.jianshu.com/p/54cc04190252)
* [PWA应用实践](https://lavas-project.github.io/pwa-book/chapter05/5-workbox.html)