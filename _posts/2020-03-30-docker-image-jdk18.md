---
layout:     post
title:      "构建 Docker Alpine 基础镜像之 Oracle JDK1.8 + Maven3.6 + Git2"
subtitle:   ""
date:       2020-03-30
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Docker
---

[TOC]


# 构建 Docker Alpine 基础镜像之 Oracle JDK1.8 + Maven3.6 + Git2

## 1 背景

### 1.1 为什么不使用OpenJDK？不使用 JRE？

- OpenJDK 与开发中使用的 Oracle JDK 有不同之处，例如 OpenJDK 缺少 javafx 依赖(javax.util.Pair 就出自这个包)，如果你的工程中使用了，会导致用 OpenJDK 编译出错；
- JRE 缺少 rt.jar，tools.jar，如果用到了，编译会报错。

### 1.2 为什么使用 Alpine ？

因为它小，节约空间。

### 1.3 为什么需要自己构建镜像？

因为根据自己的需求构建镜像，可以减少构建业务镜像的时间等待。

## 2 步骤

### 2.1 下载并解压 JDK1.8

```sh
# 使用华为云的备份仓库下载 JDK1.8
$ wget https://repo.huaweicloud.com/java/jdk/8u192-b12/jdk-8u192-linux-x64.tar.gz
# 解压
$ tar -zxvf jdk-8u192-linux-x64.tar.gz
# 删除不必要的文件
$ cd jdk-8u192-linux-x64 && rm -rf COPYRIGHT LICENSE README.html THIRDPARTYLICENSEREADME-JAVAFX.txt THIRDPARTYLICENSEREADME.txt
```

>注：
>
>华为云提供了国内的备份仓库，值得称赞，节约了很多时间。

### 2.2 下载并解压 Maven3.6

```sh
# 下载 maven
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
# 解压
$ tar -zxvf apache-maven-3.6.3-bin.tar.gz
```

### 2.3 下载 glibc

```sh
wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.31-r0/glibc-2.31-r0.apk
```

Oracle JDK 需要 glibc，因此安装下。

### 2.4 编写 Dockerfile

```dockerfile
FROM alpine:3.11
MAINTAINER jiangydev@qq.com

WORKDIR /

COPY glibc-2.31-r0.apk glibc-2.31-r0.apk

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
    apk update && \
    apk add --no-cache ca-certificates && \
    wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub && \
    apk add glibc-2.31-r0.apk && rm -rf glibc-2.31-r0.apk

ADD jdk1.8.0_192/ /usr/java/jdk/
ADD apache-maven-3.6.3/ /usr/maven/

ENV JAVA_HOME /usr/java/jdk
ENV MAVEN_HOME /usr/maven
ENV PATH ${PATH}:${JAVA_HOME}/bin:${MAVEN_HOME}/bin

RUN source /etc/profile && apk add --no-cache git
```

### 2.5 构建基础镜像

```sh
$ docker build -t 10.50.0.25/jiangydev/oracle-jdk-maven:8-192 .
```

