---
layout: post
title: "HTML5 浏览器存储的过去、现在与将来"
description: ""
category:
tags: [HTML5 browser]
---

Web app 中会有很多场景想在浏览器端保存一些数据，这篇文章中我们列举一下那些浏览器端本地存储的方案，以及各自的优缺点，还有各浏览器厂商的支持情况。

Cookies (Past, Present and Future)
------
提到浏览器的本地存储，第一个想到的可能就是使用已久的 cookie ，cookie 在 Web 开发中被广泛使用，如 Session、Authentication 等，之前一些电商网站甚至使用 cookie 保存用户放进购物车的商品。但是 cookie 有一些限制与缺点：

- 在每次的 HTTP request 中，都会传输 cookie ，当 cookie 用于 authentication 时，这是必要的；但是对于其他场景，这样不必要的传输会拖慢 web app 以及浪费带宽；
- 如果没有使用 SSL ，cookie 是明文发送的，会有安全性问题；
- Cookie 最大只能为 4KB；

Cookie 有 Expires 属性可以为 cookie 设置过期时间，当不为 cookie 设置过期时间时，用户关闭浏览器后 cookie 被删除；设置了过期时间的 cookie 被称为 Persistent cookie ，如果设置的时间较长，用户关闭浏览器后 cookie 不会被删除，只有到了过期时间才会被删除。这个功能被用来实现登录界面的 'Remeber me for two weeks' 那个 checkbox。

Cookie 在浏览器发明之初就被设计出来并广泛使用，并沿用至今，可以预见未来 cookie 仍会被一直使用。

HTML5 Web Storage (Present and Future)
------
Web storage 为浏览器本地存储提供了两个对象，这两个对象有相同的 API ：

- `window.localStorage` 保存持久化数据；
- `window.sessionStorage` 保存 session-only 数据，这些数据在 Tab 关闭后被删除；

Web storage 提供的 API：

```javascript
window.localStorage.setItem(key, value);
window.localStorage.getItem(key);
window.localStorage.removeItem(key);
window.localStorage.clear();
```

另外，可以注册监听 Web storage 的 `onstorage` 事件，当 Web storage 中的内容有改变时，接收通知。

Web storage 的一些特性：

- Web storage 提供 key/value pairs 的存储，其中 key 与 value 都只能是 String ；
- 不像 Cookie ，使用 Web storage 存储的数据不会在 server 与 client 之间反复传输；
- 最大容量为 5M；
- 当前桌面与手机的浏览器几乎都支持，包括 IE8+（两行感动的热泪）；

AngularJS 可以使用 `ngStorage` 使用 Web storage 。

HTML5 Web SQL Database (Past)
------
Web SQL Database 想将 SQL-based relational database 带到浏览器中，在 Safari、Chrome、Opera以及 iOS 3.0+、Android 2.0+ 中被实现，但被 Mozzila 与微软抵制。

Web SQL Database 使用浏览器内嵌的 SQlite ，允许使用 Javasript 执行 SQL 语句 进行一系列数据库操作。

优点：

- designed for robust client-side data storage and access based on SQL;
- Web SPA 也可以像 Android、iOS 一样使用 SQLite 了；

缺点：

- 使用之前必须先创建 schema ；如果 schema 改变，可能还要做 migration ；
- the W3C specification was abandoned in 2010；

基本上 Web SQL Database 还未流行就前途未卜，因此不建议使用。

HTML5 IndexedDB (Future)
------
IndexedDB provides a structured, transactional, high-performance NoSQL-like data store with a synchronous and asynchronous API.

相对于 Web SQL Database， IndexedDB 提供了更好用的接口，执行 SQL 在 Javascript 中的确是一件比较痛苦的事情。

起初我以为 IndexedDB 提供的是类似于 Redis 这样的 object store ，但并不是这样的，IndexedDB 的 object 类似于 SQL Database 的 record 。关于 IndexedDB 最新的浏览器支持情况比较好，但是在生产环境中真实使用还需再观察一段时间。

跳转 [An early walk-through of IndexedDB](https://hacks.mozilla.org/2010/06/comparing-indexeddb-and-webdatabase/) 了解更多关于 IndexedDB 。

总结
------
现在除了 cookie ，最成熟的浏览器本地存储方案就是 Web Storage 了，Key/Value pairs 对于大多数应用也够用了。

在 Chrome 的开发者工具中，可以查看这几种存储机制的内容。
![Chrome 开发者工具-Resources](/assets/images/chrome_develop_tool_resources.png)

References
------
- [THE PAST, PRESENT & FUTURE OF LOCAL STORAGE FOR WEB APPLICATIONS](http://diveintohtml5.info/storage.html)
- [HTML5 Browser Storage: the Past, Present and Future](http://www.sitepoint.com/html5-browser-storage-past-present-future/)
- [HTML5 Local Storage](http://tutorials.jenkov.com/html5/local-storage.html)
