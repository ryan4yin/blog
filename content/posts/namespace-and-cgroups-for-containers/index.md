---
title: "Linux 容器的工作原理：Namespace 与 Cgroups"
date: 2021-08-16T11:35:13+08:00
draft: true

# lightgallery: true
# resources:
# - name: "featured-image"
#   src: "docker-turtles-networking.jpg"

tags: ["Linux", "容器", "网络", "虚拟化"]
categories: ["技术"]
---


## 前言

>关于如何用 50 行 go 代码来实现一个简单的「容器」环境，请参见参考文章。

Namespace 技术负责环境隔离，而 Cgroups 技术则负责资源隔离与限制，这两项技术再加上 chroot、Dockerfile 与容器镜像的构建、分发机制，组合成了容器技术。

Linux 为 Namepsace 技术提供了六种不同类型的 Namespaces，三个系统调用，以及 /proc 文件系统。

三个系统调用如下：

- unshare 将调用者从原有的 namespace 分离，加入到新的 namespace 中
- clone 创建一个新的子进程，将它放到新建的 namespace 中
- setns 将某个进程放到一个已存在的 namespace 中


这里简单介绍下和 namespace 相关的 Linux 指令

```shell
# 对应 unshare 系统调用
unshare

# 管理 network namespace
ip netns xx
```

## Namespace 技术



## Cgroup 技术



## 参考

- [Linux Namespace 技术与 Docker 原理浅析](https://creaink.github.io/post/Computer/Linux/Linux-namespace.html): 循序渐进，只需要 50 行代码就能实现一个简单的「容器」环境。
- [](https://www.cnblogs.com/ryanyangcs/p/12591372.html)
- [docker 容器基础技术：linux cgroup 简介](https://cizixs.com/2017/08/25/linux-cgroup/)
