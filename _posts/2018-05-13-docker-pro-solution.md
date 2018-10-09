---
layout:     post
title:      "Docker 容器使用中的问题"
subtitle:   ""
date:       2018-05-13
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Docker
---

[TOC]

## Docker 容器使用中的问题

### `docker tag`

`docker tag`新建镜像的标签会出现相同ID而TAG不同的镜像，使用`docker rmi`可以删除指定的镜像（该命令在此处用来删除TAG）。若删除TAG的镜像被使用，以此镜像为基础的容器显示使用的镜像ID，而不是镜像的名字。

### 单文件的挂载

使用参数`-v /file:/file`，该参数将挂载单个文件至容器，但是文件的修改不会即时生效，当重启容器时才起作用。

另外还有`-v /path:/path:rw`，与`-v /path:/path`相同，表示挂载至容器的文件目录无论在容器还是宿主机上修改，都会生效。

而`-v /path:/path:ro`，表示容器中的文件目录与宿主机挂载的目录修改不互相影响。
