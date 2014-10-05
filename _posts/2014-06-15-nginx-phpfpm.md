---
layout: post
title: nginx和php-fpm日志配置
description: "nginx和php-fpm"
modified: 2014-06-15
tags: [nginx]
image:
  background: triangular.png
comments: true
---

配置好nginx和php后发现php-error一直没有日志。google了一下终于搞定了，主要对php-fpm.conf和php.ini做出如下修改:

####1.修改php-fpm.conf
修改php-fpm.conf也可能是pool.d下的www.conf的worker进程配置，将错误信息输出。默认情况下worker进程的错误日志会重定向到/dev/null

```
; Redirect worker stdout and stderr into main error log. If not set, stdout and
; stderr will be redirected to /dev/null according to FastCGI specs.
; Note: on highloaded environement, this can cause some delay in the page
; process time (several ms).
; Default Value: no
catch_workers_output = yes
```

####2.修改php.ini中配置
修改php.ini中配置，不让php将error信息输出到网页，而是输出到文件系统

```
display_errors Off
log_errors On
error_log /var/log/php/php_errors.log
```

现在就应该配置完成了，但是要注意php-fpm的worker进程要有日志文件的写权限。查看一下php-fpm worker进程的owner.

```
ps aux    |    grep php-fpm
```

比如我的系统，php-fpm master进程的owner为root，php-fpm worer进程的owner为www-data，对我而言只需要目录/var/log/php/的owner和group为www-data即可。
