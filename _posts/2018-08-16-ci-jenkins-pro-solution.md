---
layout:     post
title:      "Jenkins 使用问题及解决方案"
subtitle:   ""
date:       2018-08-16
author:     "jiangydev"
header-img: "img/post-bg-java.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Java
    - Jenkins
    - CI
---

[TOC]

# Jenkins 使用问题及解决方案

## 1 Jenkins 远程部署出现 `java: command not found`

#### 1.1 问题分析

使用 Jenkins 向远程服务器发送 shell 命令 `java -jar xxx.jar &`，自动部署一直失败，出现`java: command not found`的错误，此时检查远程服务器 JDK 环境变量配置有效。

在 Publish Over SSH 中发送`echo $PATH`命令，发现环境变量非远程服务器的。

#### 1.2 解决方案

在 Jenkins 配置 Publish Over SSH 时，在`Command Exec`中输入`export PATH=<远程服务器的环境变量>` 



## 2 使用命令 Kill 进程

#### 1.1 问题分析

#### 1.2 解决方案

```shell
$ ps -aux | grep project-name | grep -v grep | awk '{print $2}' | xargs kill -9
```

#### 1.3 注意点

要注意添加`grep -v grep`命令，否则会 kill 当前 grep 命令，导致执行命令异常退出（Jenkins 判断构建不稳定）。

```
ERROR: Exception when publishing, exception message [Exec exit status not zero. Status [-1]]
Build step 'Send build artifacts over SSH' changed build result to UNSTABLE
Finished: UNSTABLE
```

