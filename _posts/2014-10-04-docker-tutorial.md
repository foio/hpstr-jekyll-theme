---
layout: post
title: docker初探
description: "docker安装php,ssh,nginx服务"
modified: 2014-10-04
tags: [docker]
image:
  background: triangular.png
comments: true
---

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖环境到一个可移植的容器中，然后发布到任何流行的 Linux 机器上。接下来我首先简单的解释一下docker的原理，然后了解一下docker镜像的构建基本操作，最后演示如何运行docker镜像。

----

###1.原理篇

Docker可以把应用程序和它所依赖的从操作系统到软件包的环境打包成一个镜像文件。听起来好像一个典型的虚拟机，但是Docker要比虚拟机轻量的多。那么，Docker和一个完全的虚拟机有何区别？Docker是如何做到提供一个完整的文件系统，独立的网络环境等等这些功能，同时还没有如此庞大？

一个功能完整的虚拟机要在宿主机器上重新运行一个Guest Os已达到隔离环境的作用，在Docker的哲学里，这是严重的资源浪费。下图基本上描述了Docker容器和虚拟机的区别。

![enter image description here](http://infoqstatic.com/resource/articles/docker-core-technology-preview/zh/resources/0731013.jpg)

Docker核心是一个操作系统级虚拟化方法, 理解起来可能并不像VM那样直观。我们从虚拟化方法的四个方面：隔离性、资源限制性、便携性、安全性来详细介绍Docker的技术细节。

####(1).隔离性
传统的vm用一个操作系统实现隔离性，而Docker基于LXC(linux container)实现隔离性。LXC是通过kernel namespace实现隔离性的。简单说就是同一个linux操作系统中可以存在多个不同的命名空间(容器)，不同namesapce中可以存在相同的pid、net、ipc、mnt、uts等。


####(2).资源限制性
Docker通过使用cgroups 实现了对资源的配额和度量。 cgroups 的使用非常简单，可实现对单个进程(容器)的资源控制。groups可以限制blkio、cpu、cpuacct、cpuset、devices、freezer、memory、net_cls、ns九大子系统的资源

####(3).便携性
Docker通过使用AUFS (AnotherUnionFS)实现便携性。AUFS(AnotherUnionFS) 是一种 Union FS, 简单来说就是支持将不同目录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)的文件系统,  AUFS 里有一个类似分层的概念, 通过将一个 readonly 的 branch 和一个 writeable 的 branch 联合在一起，就可以实现对 readonly 权限的 branch 进行增量的进行修改(不影响 readonly 部分的)。

在Docker利用 union mount 的方式将一个 readwrite 文件系统挂载在 readonly 的rootfs之上，并且允许再次将下层的 FS(file system) 设定为readonly 并且向上叠加, 这样一组readonly和一个writeable的结构构成一个container的运行时态, 每一个FS被称作一个FS层。如下图:

![enter image description here](http://infoqstatic.com/resource/articles/docker-core-technology-preview/zh/resources/0731016.png)

####(4).安全性

关于安全性相关文章请参考[Docker官方文档](http://docs.docker.com/articles/security/)。

----

###2.构建镜像

Docker需要linux kernel 3.8以上，安装过程非常简单，在此不再赘述，详细步骤参考[官方安装文档](http://docs.docker.com/installation/ubuntulinux/)。

在Docker中开发者可以打包他们的应用以及依赖环境到一镜像中，并可以在任意docke容器中运行这个镜像。在docker的[官方register](https://registry.hub.docker.com/)中，有各种操作系统的镜像，我们可以基于这些镜像构建自己的镜像。构建镜像的有两种方法。

####(1).通过Dockerfile创建镜像
Dockerfile基本上是自解释的，从一个基础镜像开始，通过RUN命令在基本镜像上安装环境，然后通过CMD命令指定镜像启动的命令。下面的Dockerfile从ubuntu14.04镜像开始，安装并配置好sshd服务。

```
FROM     ubuntu:14.04
MAINTAINER Thatcher R. Peskens "thatcher@dotcloud.com"
RUN apt-get update && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo 'root:screencast' | chpasswd
RUN sed -i 's/PermitRootLogin without-password/PermitRootLogin yes/' /etc/ssh/sshd_config

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

编辑好Dockerfile后，可以通过docker build命令生成image。

```
sudo docker build -t test_sshd .
```

然后可以通过如下命令查看刚才生成的镜像。运行image的命令请参考docker[官方文档](http://docs.docker.com/reference/commandline/cli/)。

```
sudo docker images
```

####(2).通过commit创建镜像

我们可以先以交互的方式启动一个操作系统镜像，然后在其中做各种修改，最后通过commit命令提交。具体步骤如下：
首先通过交互的方式启动一个镜像，这样会进入镜像操作系统shell中：

```
sudo docker run -t -i utuntu
```

然后在镜像操作系统中做各种修改，退出镜像操作系统。

```
root@040acbb6c8c:/# exit
```

记下镜像标志：040acbb6c8c，最后通过commit命令提交刚才所做的修改。

```
sudo docker commit -m 'your commit message' -a 'your author info' 040acbb6c8c your-image-name
```

这样一个新的image就生成了。通过以下命令就可以查看刚才生成的镜像了。

```
sudo docker images
```

----

###3.运行镜像

镜像的运行还是比较简单的，比如如下命令运行ubuntu镜像并用echo输出hello world。

```
sudo docker run ubuntu echo 'hello world'
```

这条命令使用ubuntu镜像，如果本地docker镜像库中没有ubuntu镜像，会首先从docker的官方register中下载ubuntu镜像。echo输出完成后，改镜像就执行完成并退出了。

如何让一个镜像在docker容器中长期运行呢？比如如下命令启动一个带有sshd服务的镜像，并长期运行。

```
sudo docker run -d ubuntu-sshd /bin/bash -c ‘sshd -D’
```

两个方面保证了该镜像在容器中长期运行，第一是run命令的-d参数，表示容器以detach的方式运行；第二是镜像初始化脚本中sshd的-D参数，-D参数表示sshd以非daemonized，非detach的方式运行。镜像运行后，可以通过

```
sudo docker ps
```

查看运行的容器实例。并通过

```
sudo docker inspect
```

查看容器的ip地址等信息。docker运行image相关的参数有很多，包括容器和宿主机器之间的端口映射等参数，本文都没有涉及，请参考官方手册。

-------------
关于docker的虚拟化相关讨论问题可参考[这篇文章](http://www.geekbus.cn/docker-core-technology-preview/)。</br>
