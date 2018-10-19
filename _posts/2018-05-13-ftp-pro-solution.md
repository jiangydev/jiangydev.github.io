---
layout:     post
title:      "FTP 使用过程中的一些问题"
subtitle:   ""
date:       2018-05-13
author:     "jiangydev"
header-img: "img/post-bg-ftp.jpg"
header-mask: 0.5
catalog:    true
tags:
    - FTP
---

[TOC]

## FTP 使用过程中的一些问题

### 服务器端返回`227 Enter Passive Mode(172,17,0,6,156,72)`

#### 问题描述

使用 FTP Client 传输文件时，服务器返回了“227 Enter Passive Mode(172,17,0,6,156,72)”信息。起初认为“172,17,0,6,156,72”是被动模式的端口，但是服务器端并未开放这段端口号，通过侦听连接的端口号也并未发现这些端口号。

> 小提示
>
> 可以使用以下命令在 Windows 下过滤侦听与 IP 地址的 TCP 连接：
```cmd
netstat -an -p tcp | find /i "这里填写服务器端的 IP 地址"
```
> Linux 下命令：
```shell
netstat -tnl
```

#### 解决方案

> 小提示
>
> 这不是错误，而是服务器返回的连接信息。

格式：`227 Entering Passive Mode (h1,h2,h3,h4,p1,p2).`

这是服务器给 PASV 命令的响应。它指示服务器已准备好让客户端连接到它, 以便建立数据连接。此响应的格式很重要, 因为客户端软件必须能够分析出它所包含的连接信息。h1-h4 的值是服务器正在侦听的 IP 地址。p1-p2 的值用于计算服务器使用以下公式侦听的端口: PASV 端口 = (p1 * 256) + p2。

参考文献：[227 FTP Reply Code - KB Article #1450](http://www.serv-u.com/kb/1450/227-FTP-Reply-Code)
