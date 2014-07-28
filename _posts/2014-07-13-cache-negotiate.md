---
layout: post
title: 浏览器与服务器的缓存协商
description: "缓存协商"
modified: 2014-01-13
tags: [server browser]
image:
  background: triangular.png
comments: true
---

>当我们通过浏览器打开一个web页时，浏览器会将下载到的html，图片，js，css等缓存到本地。一旦浏览器向web服务器再次请求同样的内容时，便直接可以从本地获取。这种缓存策略是由浏览器和web服务器协商达成的，这就是http中的“**缓存协商**”

缓存协商的几种方式
###1. Last-Modified

服务器在http响应头插入Last-Modified信息：

```
Last-Modified:Sun, 13 Jul 2014 07:58:28 GMT
```

浏览器收到后，当再次请求同一页面时会带上如下头信息：

```
If-Modified-Since:Sun, 13 Jul 2014 07:58:28 GMT
```

而此时服务器响应变成：

```
HTTP/1.1 304 Not Modified
```

这表示浏览器将使用本地缓存。


###2. ETag

Etag是HTTP/1.1A支持的另一种缓存协商方法，它采用一串编码来标记标记内容，当Etag没有变化时，内容一定没有变化。
Etag首先由web服务器生成，比如lighttpd会给一个静态文件生成ETag。

```
ETag: "1963686687"
```

浏览器获得这个Etag后，在下次请求同一页面时会在HTTP头中附加以下信息来向服务器询问内容是否发生变化。

```
If-None-Match: "1963686687"
```

这时服务器重新计算这个Etag值，并与HTTP头中的附加信息进行比较，如果相同的返回：


```
HTTP/1.1 304 Not Modified
```

这表示浏览器可以使用本地缓存。


###3. Expires

无论是Last-Modified还是ETag模式下，当浏览器需要使用内容是，都要首先询问服务器以确定当前缓存是否可用，待服务器返回304时，浏览器才可以放心的使用本地缓存。而Expires就是为了完全消灭浏览器到服务器的请求的。

Expires告诉浏览器该内容何时过期，暗示在该内容过期之前不需要询问服务器。Expires的格式类似于Last-Modified，它指定了内容过期的绝对时间。比如：

```
Expires:Sun, 07 Jul 2024 03:22:19 GMT
```


###4. Cache-Control
到目前为止还有一个问题，通过Expires指定的过期时间是web服务器的时间，但是如果用户本地时间和服务器时间不一致的话，就会影响到本地缓存有效期检查。为此HTTP/1.1中还有一个Cache-Control。

Cache-Control的格式如下：

```
Cache-Control: max-age=3600
```

其中max-age指定了缓存过期的相对时间，单位是秒。
>当HTTP头中同时有Expires和Cache-Control时，浏览器会优先考虑Cache-Control。
