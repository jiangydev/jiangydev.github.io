---
layout:     post
title:      "Docker 端口规划与动态扩容"
subtitle:   ""
date:       2018-04-25
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Docker
---

[TOC]

## Docker 端口规划与动态扩容

首先，端口指TCP/IP协议中的端口，端口号的范围从`0-65535`。一个宿主机（即服务器或主机）的端口范围为0-65535；一个容器可以理解为一台主机（其拥有独立的虚拟网卡），那么也有0-65535这些端口号。使用端口时，要注意`每一个端口只能被使用一次`。

Docker容器的重启不会影响端口映射参数，同时容器只有在使用`docker run <容器ID/容器名>`新建时才能指定容器的参数，这也意味着不能用重启的方式修改端口映射关系。使用命令`# docker inspect <容器ID/容器名>`，可以看到容器的详细参数，包括端口的映射关系（此外，容器停止时，解除了宿主机的端口占用）。

### 1 端口规划

> 容器的端口映射在容器启动时指定，因此需要提前规划好端口的使用，防止端口的冲突。

> 端口规划问题可以分为两大类：
>
> * 单IP多容器映射方案：该方案适合在单个宿主机上解决服务器端口与容器端口的映射问题。
>
> * 多IP多容器映射方案：

#### 1.1 单IP多端口映射方案

在一个宿主机中，由于业务的需求和端口被使用的情况，可能出现端口多对多，如FTP被动模式需要打开一些高端口号用于数据传输，由此展开下面的分析。

##### 1.1.1 一对多的几种情况

1. 端口多对一，可行：访问宿主的多个端口可以映射到容器中的同一个端口。

  ```
  [root@jiangydev ~]# docker run -d -p 9000-9001:9000 tomcat:8.5-jre8
  ```

2. 一对多，出错

  ```
  [root@jiangydev ~]# docker run -d -p 9000:9000-9001 tomcat:8.5-jre8
  /usr/bin/docker-current: Invalid ranges specified for container and host Port
  s: 9000-9001 and 9000.
  ```

##### 1.1.2 多对多的几种情况

1. 相同端口数量映射

  9000-9001:9000-9001 可行

  ```
  [root@jiangydev ~]# docker run -d -p 9000-9001:9000-9001 tomcat:8.5-jre8
  ```

  9000-9001:9002-9003 可行

  ```
  [root@jiangydev ~]# docker run -d -p 9000-9001:9000-9001 tomcat:8.5-jre8
  ```

2. 不同端口数量的映射

  9000-9001:9000-9003 出错

  ```
  [root@jiangydev ~]# docker run -d -p 9000-9001:9000-9003 tomcat:8.5-jre8
  /usr/bin/docker-current: Invalid ranges specified for container and host Port
  s: 9000-9003 and 9000-9001.
  ```

  9000-9003:9000-9001 出错

  ```
  [root@jiangydev ~]# docker run -d -p 9000-9003:9000-9001 tomcat:8.5-jre8
  /usr/bin/docker-current: Invalid ranges specified for container and host Port
  s: 9000-9001 and 9000-9003.
  ```

#### 1.2 多IP多容器映射方案

  ```
  [root@jiangydev ~]# docker run -d -p 127.0.0.1:9000:9000 tomcat:8.5-jre8
  ```

  ```
  [root@jiangydev ~]# docker run -d -p 127.0.0.1::9000 tomcat:8.5-jre8
  ```

### 2 端口动态扩容：已创建的容器端口映射

#### 2.1 提交为新的容器并重新指定映射端口

  提交运行中的容器为镜像

  ```
  [root@jiangydev ~]# docker commit test_tomcat:8.5-jre8 jiangjiangy.xyz/my_tomcat:1.0
  ```

  运行新的镜像并添加映射端口

  ```
  [root@jiangydev ~]# docker run -dit -p 8080:80  jiangjiangy.xyz/my_tomcat:1.0 /bin/bash
  ```

#### 2.2 给运行中的容器添加映射端口

  Docker的端口映射是由iptables实现的，那么修改iptables规则就能实现端口的动态扩容。

  查看iptable转发端口。

  ```
  [root@jiangydev ~]# iptables -t nat -nvL --line-numbers
  ```

  > 注意：容器停止状态下，没有端口转发条目。

  获得容器IP。

  ```
  # 这里的容器ID只用了前3位，能唯一确定该容器即可，用容器名也是可以的。
  [root@jiangydev ~]# docker inspect 1f7 | grep IPAddress
  ```

  将宿主机的8080端口映射到docker容器IP为172.17.0.4的8080端口。

  ```
  [root@jiangydev ~]# iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.4:8080
  ```

  > 通过端口转发的方式，若不保存iptable规则，重启系统后失效；在iptable规则生效的情况下，重启docker容器不影响端口映射。
  >
  > 若要删除规则可以使用以下命令：
  > 查看规则：`[root@jiangydev ~]# iptables -t nat -nvL --line-numbers`
  > 删除需要删除的规则：`[root@jiangydev ~]# iptables -D DOCKER 4 -t nat `
  > *注意：删除规则的时候要注意num。因为如果有5个规则，删除了第4个，原来的第五个会变成第4个。*

#### 2.3 修改容器的配置文件来实现端口映射的修改

  修改文件`/var/lib/docker/containers/<容器ID>/config.v2.json`中容器暴露的端口。

  ```json
  "ExposedPorts":{
    "8080/tcp":{},
    "9003/tcp":{},
    "9004/tcp":{}
  }
  ```

  修改文件`/var/lib/docker/containers/<容器ID>/hostconfig.json`中容器与宿主机的映射关系。

  ```json
  "PortBindings":{
    "9003/tcp":[{
      "HostIp":"",
      "HostPort":"9001"
      }],
    "9004/tcp":[{
      "HostIp":"",
      "HostPort":"9002"
      }]
  }
  ```

  修改文件后保存，并重启Docker.

  ```
  [root@jiangydev ~]# systemctl restart docker
  ```

  > 该方法`必须重启Docker服务`才能使修改生效。若仅仅重启容器是无效的，如`docker restart`命令，修改的文件内容会被覆盖。

### 3 容器的配置信息

其实容器的配置信息都是保存在相应的文件中，通过`find`命令可以找到容器在宿主机中对应的文件。

#### 3.1 查找容器在宿主机中的文件目录

  先查看容器ID（数了一下，共64位），再使用find命令查找其目录。

  > 这里用容器ID`1f7766ab9e9fe73fcaabc5605979b85c39d496fed409d61517ff4dd4837af307`做测试。

  ```
  [root@jiangydev ~]# find -name 1f77
  ......
  /var/lib/docker/containers/1f7766ab9e9fe73fcaabc5605979b85c39d496fed409d61517ff4dd4837af307
  ......
  ```

  只需要关注`/var/lib/docker/containers/`这一目录下的文件即可，接下来一一解释。

|文件名|文件类型|作用|
|:---:|:---:|:---:|
|checkpoints|目录||
|config.v2.json|文件|容器的详细配置，包括环境变量、该目录下其他配置文件的设置等|
|hostconfig.json|文件|主机相关的配置|
|hostname|文件|主机名，为容器ID的前12位，如1f7766ab9e9f。|
|hosts|文件|IP与域名之间的映射关系，如`172.17.0.5    1f7766ab9e9f`|
|resolv.conf|文件|routines的集合，提供对Internet域名系统（DNS）的访问，如`nameserver 100.100.2.136`|
|resolv.conf.hash|文件|resolv.conf哈希值|
|secrets|目录|敏感信息|
|shm|目录|为容器分配的一个内存文件系统，用于挂载到容器中的/dev/shm|

 容器没有启动的时候，checkpoints、secrets、shm目录下的内容为空。
