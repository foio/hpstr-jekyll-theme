---
layout: post
title: firefox profile
description: "firefox"
modified: 2014-05-10
tags: [firefox]
image:
  background: triangular.png
---

>profile是firefox的高级概念，开发者在扩展开发过程中一般使用不同的profile模拟不同的浏览器环境，用于测试扩展的兼容性。

什么是profile
====
```
profile是firefox用户数据的集合，包括书签，设置信息，扩展，密码信息，浏览历史等数据。
```

-------

如何创建新的profile
====

```
使用firefox的命令行创建新的profile，该命令要求所用的firefox实例都关闭：
firefox.exe -P
```
会打开如下窗口：
<figure>
    <img src="http://sztqb.sznews.com/res/1/641/2010-11/04/C04/res01_attpic_brief.jpg"/>
    <figcaption>firefox profile窗口</figcaption>
</figure>

-----

如何使用新的profile
===

```

使用命令行启动firefox，假设上一步创建的新的profile文件为TestProfile，可以使用如下命令使用新的profile启动firefox浏览器
firefox.exe -P  “TestProfile”

```

