---
title: Service Worker 介绍（译）
date: 2017-03-02 10:55:02
categories: ['技术', '前端']
tags: ["PWA", "service-worker"]
---

原文链接地址:[https://developers.google.com/web/fundamentals/getting-started/primers/service-workers](https://developers.google.com/web/fundamentals/getting-started/primers/service-workers)

  丰富的离线体验，定期的后台同步，推送通知的功能（正常是一个本地应用才有的功能）现在Web也支持了，Service Worker是这些特性实现的基础。

### service worker 是什么
sevice worker是一个从网页中分离，运行在浏览器后台的脚本，它开启了不需要网页或者用户交互的一些特性。现在，它已经包括了推送通知和后台同步的特性。将来它将支持像定期同步和地理围栏的特性。在本文讨论的核心特性是监听和处理网络请求的能力，包括响应缓存的管理。

这是一个令人兴奋的API的原因是允许你支持离线和给开发者完全控制的体验。

在service worker之前，有一个叫AppCache的API可以给用户一个离线体验，但AppCache API有一些问题，service workers被设计出来处理来避免这些问题。


sevice worker要注意的点:

-  它是一个Javascript Worker，所以它不能直接访问DOM，相反，service worker可以通过postMessage接口跟它控制的页面通信，而这些页面可以维护这些DOM.


- Service worker是一个可编程的网络代理，允许你控制如何来处理你页面的网络请求。

- 不使用的时候停止运行，下一次需要使用的时候重启，所以你不能在它的onfetch和onmessage事件处理中依赖全局状态。如果有一些信息你需要持久化或者重启后复用，它可以通过访问IndexedDB API来实现。

-  Service workers广泛使用promises，所以如果你是刚接触promises，那你最好停下来先去查看一下promises。

### service worker的生命周期

service worker有一个完全与网页分离的生命周期

为了在你网页中安装service worker，你需要在你的页面脚本中注册service worker。注册service worker将使浏览器在后台开始一个安装步骤。

通常在安装期间，你想要缓存一些静态资源。如果所有的文件都缓存成功，service worker将会变成installed(已安装)。如果有任何文件下载或者缓存失败，然后安装过程将会失败和service worker将不会activite(激活),将不会installed(已安装)。如果发生了这种情况，不要担心，它将会在下次重新尝试。如果它安装了意味着你知道可以从缓存中获取哪些静态资源。

当service worker installed(已安装),随后将会执行activition(激活)步骤，这是一个很好的机会来管理旧的缓存，这一部分我们将在service worker更新部分介绍。

在激活之后，service worker将控制所有属于它作用域的页面。但是首次注册service worker将不会被控制，直到再次页面再次加载。一旦service worker处理控制中，它将会处于两种状态其中之一：一种是service worker将会终止来节省内存，一种是当你页面发生网络请求或者产生消息时，它将处理fetch和message事件。

以下是service worker首次安装生命周期的简要版本

![service worker life cycle](http://o8695iplk.bkt.clouddn.com/sw-lifecycle.png)


### 先决条件

##### 浏览器支持
浏览器的特性不断的在增加，Service workers已经被Firefox和Opera支持，Microsoft Edge正在开发当中，甚至Safari也表明了未来支持的意向，你可以在[Jake Archibald's is Serviceworker](https://jakearchibald.github.io/isserviceworkerready/)关注各个浏览器进程.

##### 你需要Https


在开发期间，你将可以通过localhost来使用service worker, 但是部署站点的时候你需要在你的服务器上安装HTTPS

你可以使用service worker来拦截连接，制造和过滤响应。然而如果一个中间人来使用这些强大的功能可能就不好了，为了避免这种情况，你可以仅仅通过Https来注册你的service worker，这样我们确保service worker接收的东西不会在传输过程中被篡改。

如果你想要让你的服务器支持Https，你需要获得一个TLS证书，然后在你的服务器上安装，安装方式取决你的服务器，请查看服务器文档和[Mozilla's SSL config generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/)最佳实践。



### 注册service worker

为了安装一个service worker，你需要在你的页面注册它来启动该进程。让浏览器知道你的service worker Javascript文件是在哪个位置。



```
if ('serviceWorker' in navigator) {
  window.addEventListener('load', function() {
    navigator.serviceWorker.register('/sw.js').then(function(registration) {
      // 注册成功
      console.log('ServiceWorker registration successful with scope: ', registration.scope);
    }).catch(function(err) {
      // 注册失败
      console.log('ServiceWorker registration failed: ', err);
    });
  });
}
```

这代码检测service worker API是否可用，如果是可用的，一旦页面加载完成， sw.js service worker就被注册。

你可以调用register每次页面加载完，浏览器会根据它注册与否来做相应处理。


一个register方法需要注意的点是service worker文件的定位，在这个例子当中，你看到了service worker文件是域的根目录。这意味着service worker的作用范围是整个源。换句话说，这个service worker将接收域下的任何fetch事件。如果我们注册service worker在`/example/sw.js`，然后service worker将仅仅监测以`/example/`开头的页面的fetch事件(例如 /example/page1/, /example/page2/).


现在你可以通过`chrome://inspect /#service-workers`查看service worker是否启用和寻找你的站点。

![check service workers](http://o8695iplk.bkt.clouddn.com/sw-chrome-inspect.png)

当service worker被首次实现你也可以通过`chrome://serviceworker-internals`来查看你的service worker的详情。如果只是为了了解关于service workers的生命周期。不用惊讶它后面会被`chrome://inspect/#service-workers`完全替代。

你可能会发现在隐身窗口中测试你的service worker是有用的，因为你可以关闭然后重新打开知道前一个service worker不会影响新的窗口。

### 安装service worker

在受控页面启动注册之后，让我们看下service worker处理`install`事件的脚本。

对于最基本的例子，你需要定义一个install事件的回调和决定哪些文件需要缓存。

```
self.addEventListener('install', function(event) {
  // Perform install steps
});
```

在我们的install回调里面，我们需要采取以下步骤：

1. 打开缓存
2. 缓存文件
3. 检查所有需要的资源是否缓存了

```
var CACHE_NAME = 'my-site-cache-v1';
var urlsToCache = [
  '/',
  '/styles/main.css',
  '/script/main.js'
];

self.addEventListener('install', function(event) {
  // Perform install steps
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        console.log('Opened cache');
        return cache.addAll(urlsToCache);
      })
  );
});
```

你可以看到用我们期望的缓存名字来调用`caches.open()`，之后我们调用`cache.addAll()`和传递我们的文件数组。这是一个promises链(`caches.open()`和`cache.addAll()`)。`event.waitUntil()`方法接收一个promise， 通过它可以知道安装花费多长时间和是否安装成功。

如果所有文件缓存成功，service worker就变成installed(已安装)，如果有任何文件下载失败，那安装过程就是失败的。这允许你定义你所有依赖的资源，但也意味着你需要小心决定在安装过程中要缓存的文件列表。定义一个长的文件列表将增加文件缓存失败的机会，导致你的service worker将不会被安装。

`This is just one example, you can perform other tasks in the install event or avoid setting an install event listener altogether.(待译)`

### 缓存和返回请求

现在你已经安装了一个service worker，你可能想要返回你缓存当中的其中一个资源，对吗？

在service worker安装完之后，然后用户导航到一个不同的页面或者刷新，service worker将开始接收fetch事件，下面是一个例子

```
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        //缓存击中
        if (response) {
          return response;
        }
        return fetch(event.request);
      }
    )
  );
});
```

这儿我们定义了`fetch`事件，和在`event.respondWith()`我们通过`caches.match()`传递一个promise, `caches.match()`查看请求和在你service worker创造的任何缓存中查找找缓存结果。

如果我们找到一个匹配的响应，我们返回缓存值，否则我们返回fetch调用的结果，fetch调用会发起网络请求和返回任何可以在网络中查找到的数据。这是一个使用任何在安装过程中缓存的资源的一个简单例子。

如果我们想要缓存新的请求，我们可以通过处理获取请求的响应然后把增加到缓存当中，像下面这样。

```
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        // Cache hit - return response
        if (response) {
          return response;
        }

        // IMPORTANT: Clone the request. A request is a stream and
        // can only be consumed once. Since we are consuming this
        // once by cache and once by the browser for fetch, we need
        // to clone the response.
        
        //注意：复制一个请求，一个请求是一个流仅仅可以被消费一次，因为我们一次通过消费缓存，一次通过浏览器fetch,所以我们需要复制response
        var fetchRequest = event.request.clone();

        return fetch(fetchRequest).then(
          function(response) {
            // Check if we received a valid response
            if(!response || response.status !== 200 || response.type !== 'basic') {
              return response;
            }

            // IMPORTANT: Clone the response. A response is a stream
            // and because we want the browser to consume the response
            // as well as the cache consuming the response, we need
            // to clone it so we have two streams.
            
            var responseToCache = response.clone();

            caches.open(CACHE_NAME)
              .then(function(cache) {
                cache.put(event.request, responseToCache);
              });

            return response;
          }
        );
      })
    );
});
```

这个例子我们做：

1. 增加回调到`.then()`和`fetch`请求
2. 一旦我们得到响应，我们执行以下检查：
3. 确保响应式可用的
4. 检查响应状态是否是200
5. 确保响应类型是基本的，这表明它是我们来源的请求，这也意味着第三方资源的请求是不会缓存的
6. 如果我们通过了检查，我们复制`response`，原因是这样的，因为response是一个流，body仅仅可以被消费一次，因为我们要返回的响应为浏览器使用，以及将它传递给缓存使用，我们需要复制它，以至于我们可以发送一次给浏览器和一次给缓存。

### 更新service worker

有一个时间点你的service worker将需要更新，当该时间到来时，您需要按照以下步骤操作：

1. 更新你的service worker JavaScript文件. 当用户导航到你的站点时, 浏览器尝试重新下载在后台定义service worker脚本文件，如果跟当前文件比起来有字节上的差异，它都被认为是新的。
2. 你新的service worker将会启动和`install`事件将会触发
3. 在这个点老的service worker仍然控制着当前页面而新的service worker将会进入等待状态
4. 在你站点当前打开的页面关闭的时候，老的service worker将会关闭，新的service worker将会控制页面
5. 一旦你新的service worker能够控制页面，它的`activate`事件将被触发。

在`activate`回调中将发生的一个常见任务是缓存管理，你想要在`activate`回调中做这个的原因是如果你在安装步骤中清除任何缓存，任何仍然控制着当前页面的老的能够从该缓存提供文件service worker将突然停掉。

Let's say we have one cache called 'my-site-cache-v1'
假设我们有一个缓存叫`'my-site-cache-v1'`, 我们发现我们想把它分成一个页面缓存和一个博客缓存，这意味着在安装过程中我们创建了两个缓存`'pages-cache-v1'`和`'blog-posts-cache-v1'`，在`activate`步骤我们想要删除我们老的`'my-site-cache-v1'`缓存。

下面的代码将循环遍历所有在service worker的缓存，删除任何不在白名单中的缓存。

```
self.addEventListener('activate', function(event) {

  var cacheWhitelist = ['pages-cache-v1', 'blog-posts-cache-v1'];

  event.waitUntil(
    caches.keys().then(function(cacheNames) {
      return Promise.all(
        cacheNames.map(function(cacheName) {
          if (cacheWhitelist.indexOf(cacheName) === -1) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```

### 粗糙的边缘和陷阱

这东西真的是新的，有一堆的问题，希望这部分不久会被删除，但是现在还是要注意的。

###### 如果安装失败，我们不能很好告诉你
如果一个service worker注册，但是没有出现在`chrome://inspect/#service-workers`或`chrome://serviceworker-internals`，它有可能因为错误抛出或者一个rejected promise被传递到`event.waitUntil()`安装失败。

为了解决这个问题，打开`chrome://serviceworker-internals`和检查“打开DevTools窗口，并在服务工作者启动时暂停JavaScript执行以进行调试"，和在你安装事件的开始输入debugger语句，以及Pause on uncaught exceptions应该可以找到问题。

###### fetch 默认值
###### 默认无凭证
When you use fetch, by default, requests won't contain credentials such as cookies.
当你使用fetch，请求默认不包含像cookies这样的凭证，如果你想要凭证，可以这样调用

```
fetch(url, {
  credentials: 'include'
})
```
这种行为是有意的，并且可以说比XHR更复杂的发送凭据的默认值（如果URL是同源的)，否则省略它们，Fetch的行为更像是其他CORS请求，例如`<img crossorigin>`，它从来不会发送cookie除非你加入`<img crossorigin="use-credentials">`

###### no-CORS默认失败

如果它不支持CORS,获取一个第三方资源默认是会失败的，你可以增加一个no-CORS项到Request来覆盖默认值，虽然这将导致"不明显"的响应，这将意味着你不能知道响应是否成功。

```
cache.addAll(urlsToPrefetch.map(function(urlToPrefetch) {
  return new Request(urlToPrefetch, { mode: 'no-cors' });
})).then(function() {
  console.log('All resources have been fetched and cached.');
});
```

###### 处理响应图片
`srcset`属性或者`<picture>`元素会在运行时选择一张合适的图片，然后发起网络请求。
对于service worker，如果您想在安装步骤期间缓存图像，则有以下几个选项：

























  

