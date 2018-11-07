---
layout:     post
title:      "Docker 容器使用中的问题"
subtitle:   ""
date:       2018-05-13
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Docker
---

[TOC]

# Docker 容器使用中的问题

## 1 Docker

### 1.1 `docker tag`

`docker tag`新建镜像的标签会出现相同ID而TAG不同的镜像，使用`docker rmi`可以删除指定的镜像（该命令在此处用来删除TAG）。若删除TAG的镜像被使用，以此镜像为基础的容器显示使用的镜像ID，而不是镜像的名字。

### 1.2 单文件的挂载

使用参数`-v /file:/file`，该参数将挂载单个文件至容器，但是文件的修改不会即时生效，当重启容器时才起作用。

另外还有`-v /path:/path:rw`，与`-v /path:/path`相同，表示挂载至容器的文件目录无论在容器还是宿主机上修改，都会生效。

而`-v /path:/path:ro`，表示容器中的文件目录与宿主机挂载的目录修改不互相影响。





## 2 Docker Swarm

#### 2.1 Unable to query docker version

##### 问题描述

```shell
$ docker-machine ls
NAME                     ACTIVE   DRIVER       STATE     URL                          SWARM   DOCKER    ERRORS
swm0-worker-local-vb-0   -        virtualbox   Running   tcp://192.168.99.100:2376            Unknown   Unable to query docker version: Get https://192.168.99.100:2376/v1.15/version: dial tcp 192.168.99.100:2376: connectex: No connection could be made because the target machine actively refused it.
```

##### 解决方案

```shell
$ docker-machine env swm0-worker-local-vb-0
Error checking TLS connection: Error checking and/or regenerating the certs: There was an error validating certificates for host "192.168.99.100:2376": dial tcp 192.168.99.100:2376: connectex: No connection could be made because the target machine actively refused it.
You can attempt to regenerate them using 'docker-machine regenerate-certs [name]'.
Be advised that this will trigger a Docker daemon restart which might stop running containers.
```

使用 `docker-machine env [name]` 看到提示使用 `docker-machine regenerate-certs [name]`。

```shell
$ docker-machine regenerate-certs swm0-worker-local-vb-0
Regenerate TLS machine certs?  Warning: this is irreversible. (y/n): y
Regenerating TLS certificates
Waiting for SSH to be available...
Detecting the provisioner...
Unable to verify the Docker daemon is listening: Maximum number of retries (10) exceeded
```

然而又遇到了新的错误：`Unable to verify the Docker daemon is listening`，解决的思路如下。

- 检查 docker 进程是否在运行

  我这里重新运行了 docker 就好了。

  ```shell
  $ docker-machine ssh swm0-worker-local-vb-0
  > sudo /etc/init.d/docker restart
  ```

- 检查防火墙是否放行 TCP 端口 2376