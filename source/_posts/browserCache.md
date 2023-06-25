---
title: 浏览器缓存机制
---

<small>原文发布于 [CSDN](https://blog.csdn.net/RaeZhang/article/details/117528885)，本文系近期迁移，想看更多欢迎访问 [我的 CSDN 主页](https://blog.csdn.net/RaeZhang?type=blog)</small>

# 前言

一个优秀的缓存策略可以缩短网页请求资源的距离，减少延迟，并且由于缓存文件可以重复利用，还可以减少带宽，降低网络负荷。  
 \
 对于一个数据请求来说，可以分为发起**网络请求**、**后端处理**、**浏览器响应**三个步骤。  
 \
浏览器缓存可以帮助我们在第一和第三步骤中优化性能:比如说直接使用缓存而不发起请求，或者发起了请求但后端存储的数据和前端一致，那么就没有必要再将数据回传回来，这样就减少了响应数据。

# 缓存位置

浏览器缓存共有四种，且有一定的优先级，当所有缓存都没有命中时才回去进行网络请求。

1. Service Worker
2. Memory Cache
3. Disk Cache
4. Push Cache

## Service Worker

Service Worker 是运行在浏览器背后的独立线程，一般可以用来实现缓存功能。使用 Service Worker 的话，传输协议必须为 HTTPS。因为 Service Worker 中涉及到请求拦截，所以必须使用 HTTPS 协议来保障安全。Service Worker 的缓存与浏览器其他内建的缓存机制不同，它可以让我们自由控制缓存哪些文件、如何匹配缓存、如何读取缓存，并且缓存是持续性的。

1.  注册 Service worker :在你的 index.html 加入以下内容

```javascript
/* 判断当前浏览器是否支持serviceWorker */
if ("serviceWorker" in navigator) {
  /* 当页面加载完成就创建一个serviceWorker */
  window.addEventListener("load", function () {
    /* 创建并指定对应的执行内容 */
    /* scope 参数是可选的，可以用来指定你想让 service worker 控制的内容的子目录。 在这个例子里，我们指定了 '/'，表示 根网域下的所有内容。这也是默认值。 */
    navigator.serviceWorker
      .register("./serviceWorker.js", { scope: "./" })
      .then(function (registration) {
        console.log(
          "ServiceWorker registration successful with scope: ",
          registration.scope
        );
      })
      .catch(function (err) {
        console.log("ServiceWorker registration failed: ", err);
      });
  });
}
```

2.安装 worker：在我们指定的处理程序 serviceWorker.js 中书写对应的安装及拦截逻辑

```javascript
/* 监听安装事件，install 事件一般是被用来设置你的浏览器的离线缓存逻辑 */
this.addEventListener("install", function (event) {
  /* 通过这个方法可以防止缓存未完成，就关闭serviceWorker */
  event.waitUntil(
    /* 创建一个名叫V1的缓存版本 */
    caches.open("v1").then(function (cache) {
      /* 指定要缓存的内容，地址为相对于跟域名的访问路径 */
      return cache.addAll(["./index.html"]);
    })
  );
});

/* 注册fetch事件，拦截全站的请求 */
this.addEventListener("fetch", function (event) {
  event.respondWith(
    // magic goes here

    /* 在缓存中匹配对应请求资源直接返回 */
    caches.match(event.request)
  );
});
```

当 Service Worker 没有命中缓存的时候，我们需要去调用 fetch 函数获取数据。也就是说，如果我们没有在 Service Worker 命中缓存的话，会根据缓存查找优先级去查找数据。但是不管我们是从 Memory Cache 中还是从网络请求中获取的数据，浏览器都会显示我们是从 Service Worker 中获取的内容。

## Memory Cache

Memory Cache 也就是内存中的缓存

优点：读取速度快

缺点：一旦我们关闭 Tab 页面，内存中的缓存也就被释放了。
当我们访问过页面以后，再次刷新页面，可以发现很多数据都来自于内存缓存

当我们刷新一个已经打开的页面，通常可以看到许多 memory cache 的资源。
![image](./img/browserCache/memoryCache.png)

## Disk Cache

Disk Cache 也就是存储在硬盘中的缓存，读取速度慢点，但是什么都能存储到磁盘中，比之 Memory Cache 胜在容量和存储时效性上。

在所有浏览器缓存中，Disk Cache 覆盖面基本是最大的。它会根据 HTTP Herder 中的字段判断哪些资源需要缓存，哪些资源可以不请求直接使用，哪些资源已经过期需要重新请求。并且即使在跨站点的情况下，相同地址的资源一旦被硬盘缓存下来，就不会再次去请求数据。绝大部分的缓存都来自 Disk Cache，关于 HTTP 的协议头中的缓存字段，我们会在下文进行详细介绍。

浏览器会把哪些文件丢进内存中？哪些丢进硬盘中？
关于这点，网上说法不一，不过以下观点比较靠得住：

对于大文件来说，大概率是不存储在内存中的，反之优先
当前系统内存使用率高的话，文件优先存储进硬盘

## Push Cache

Push Cache（推送缓存）是 HTTP/2 中的内容，当以上三种缓存都没有命中时，它才会被使用。它只在会话（Session）中存在，一旦会话结束就被释放，并且缓存时间也很短暂，在 Chrome 浏览器中只有 5 分钟左右，同时它也并非严格执行 HTTP 头中的缓存指令。

浏览器可以拒绝接受已经存在的资源推送
你可以给其他域名推送资源
如果以上四种缓存都没有命中的话，那么只能发起请求来获取资源了。

# 缓存过程分析

![image](./img/browserCache/cacheProcess.png)
根据是否需要向服务器重新发起 HTTP 请求将缓存过程分为两个部分，分别是强缓存和协商缓存。

# 强缓存

不会向服务器发送请求，直接从缓存中读取资源。

在 chrome 控制台的 Network 选项中可以看到该请求返回 200 的状态码，并且 Size 显示**from disk cache**或**from memory cache**。
如
![image](./img/browserCache/memoryCacheExample.png)

强缓存可以通过设置两种 HTTP Header 实现：Expires 和 Cache-Control。

1. Expires
   缓存过期时间，用来指定资源到期的时间，是服务器端的具体的时间点。

2. Cache-Control
   比如：Cache-Control:max-age=300 时，则代表在这个请求正确返回的 5 分钟内再次加载资源，就会命中强缓存。
   ![image](./img/browserCache/cacheControlExample.png)

## Expires 和 Cache-Control 的差别

其实这两者差别不大，区别就在于 Expires 是 http1.0 的产物，Cache-Control 是 http1.1 的产物，两者同时存在的话，Cache-Control 优先级高于 Expires；在某些不支持 HTTP1.1 的环境下，Expires 就会发挥用处。所以 Expires 其实是过时的产物，现阶段它的存在只是一种兼容性的写法。

强缓存判断是否缓存的依据来自于是否超出某个时间或者某个时间段，而不关心服务器端文件是否已经更新，这可能会导致加载文件不是服务器端最新的内容，那我们如何获知服务器端内容是否已经发生了更新呢？此时我们需要用到协商缓存策略。

# 协商缓存

协商缓存就是强制缓存失效后，浏览器携带缓存标识向服务器发起请求，由服务器根据缓存标识决定是否使用缓存的过程。
协商缓存可以通过设置两种 HTTP Header 实现：Last-Modified 和 ETag 。

## Last-Modified 和 If-Modified-Since

Last-Modified 指的是这个资源在服务器上的最后修改时间，
浏览器下一次请求这个资源，浏览器检测到有 Last-Modified 这个 header，于是添加 If-Modified-Since 这个 header，值就是 Last-Modified 中的值；服务器再次收到这个资源请求，会根据 If-Modified-Since 中的值与服务器中这个资源的最后修改时间对比，如果没有变化，返回 304 和空的响应体，直接从缓存读取
![image](./img/browserCache/negotiationCacheProcess.png)

如下面这个请求：
request headers 中有 if-modified-since,表明浏览器之前请求过这个资源
![image](./img/browserCache/negotiationCacheExample.png)
response headers 返回状态 304,表明协商缓存成功，last-modified 与请求头中的 if-modified-since 一致。
![image](./img/browserCache/negotiationCacheExample2.png)

如果 If-Modified-Since 的时间小于服务器中这个资源的最后修改时间，说明文件有更新，协商缓存失效，返回 200 和请求结果
![image](./img/browserCache/withoutCacheProcess.png)

### 弊端：

如果本地打开缓存文件，即使没有对文件进行修改，但还是会造成 Last-Modified 被修改，服务端不能命中缓存导致发送相同的资源。

因为 Last-Modified 只能以秒计时，如果在不可感知的时间内修改完成文件，那么服务端会认为资源还是命中了，不会返回正确的资源。

既然根据文件修改时间来决定是否缓存尚有不足，能否可以直接根据文件内容是否修改来决定缓存策略？所以在 HTTP / 1.1 出现了 ETag 和 If-None-Match

## ETag 和 If-None-Match

Etag 是服务器响应请求时，返回当前资源文件的一个唯一标识(由服务器生成)，只要资源有变化，Etag 就会重新生成。
浏览器在下一次加载资源向服务器发送请求时，会将上一次返回的 Etag 值放到 request header 里的 If-None-Match 里，服务器只需要比较客户端传来的 If-None-Match 跟自己服务器上该资源的 ETag 是否一致，就能很好地判断资源相对客户端而言是否被修改过了。如果服务器发现 ETag 匹配不上，那么直接以常规 GET 200 回包形式将新的资源（当然也包括了新的 ETag）发给客户端；如果 ETag 是一致的，则直接返回 304 知会客户端直接使用本地缓存即可。

![image](./img/browserCache/EtagProcess.png)

## 协商缓存的两者对比

精度：Etag 要优于 Last-Modified。
性能：Last-Modified 要优于 Etag。
优先级：服务器校验优先考虑 Etag

# 实际使用策略

对与频繁变动的资源：
使用 Cache-Control: no-cache，使浏览器每次都请求服务器，然后配合 ETag 或者 Last-Modified 来验证资源是否有效。这样的做法虽然不能节省请求数量，但是能显著减少响应数据大小。

对于不常变化的资源：
通常在处理这类资源时，给它们的 Cache-Control 配置一个很大的 max-age=31536000 (一年)，这样浏览器之后请求相同的 URL 会命中强制缓存。而为了解决更新的问题，就需要在文件名(或者路径)中添加 hash， 版本号等动态字符，之后更改动态字符，从而达到更改引用 URL 的目的，让之前的强制缓存失效 (其实并未立即失效，只是不再使用了而已)。

# 用户行为如何触发缓存

打开网页，地址栏输入地址： 查找 disk cache 中是否有匹配。如有则使用；如没有则发送网络请求。
普通刷新 (F5)：因为 TAB 并没有关闭，因此 memory cache 是可用的，会被优先使用(如果匹配的话)。其次才是 disk cache。
强制刷新 (Ctrl + F5)：浏览器不使用缓存，因此发送的请求头部均带有 Cache-control: no-cache(为了兼容，还带了 Pragma: no-cache),服务器直接返回 200 和最新内容。

# 参考

[深入理解浏览器的缓存机制](https://www.jianshu.com/p/54cc04190252)
[浏览器缓存：memory cache、disk cache、强缓存协商缓存等概念](https://blog.csdn.net/weixin_43972437/article/details/105513486)
[service worker 是什么？看这篇就够了](https://zhuanlan.zhihu.com/p/115243059)
