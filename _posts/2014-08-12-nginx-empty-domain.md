---
layout: post
title: nginx,防止他人域名指向自己机器
description: "nginx"
modified: 2014-08-12
tags: [nginx]
image:
  background: triangular.png
comments: true
---

总有一些人，热衷于窃取他人的劳动果实。比如好事者用一个空头域名指向你的主机，从而利用你的网站内容提升他自己的域名排名。比如我就发现有两个空头域名sunrisenet.cn、mbagroup.cn指向了我的网站，我们该如何防止这种情况发生呢?

##方法1：将来自其他域名的访问导向自己的域名


```
 server {
     listen 80 default;
     server_name _;
     rewrite ^(.*) http://www.xuanhao360.com;
 }
 server {
     listen 80;
     server_name  www.xuanhao360.com;
     root /var/www;
     index  index.html;
     ......
 }
```


##方法2：对来自其他域名的访问返回500错误

```
server {
        listen 80 default;
        server_name _;
        return 500;
}
 server {
     listen 80;
     server_name  www.xuanhao360.com;
     root /var/www;
     index  index.html;
     ......
 }
```

##方法3：黑名单

```
 server {
     listen 80 default;
     server_name sunrisenet.cn;
     return 500;
 }
 server {
     listen 80;
     server_name  www.xuanhao360.com;
     root /var/www;
     index  index.html;
     ......
 }
```
