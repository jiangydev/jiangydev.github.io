---
layout:     post
title:      "在 CentOS7 上安装 Docker"
subtitle:   "设置镜像加速器"
date:       2017-07-08
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Docker
    - Linux
---

[TOC]

## 在 CentOS7 上安装 Docker

### CentOS 系统环境

1. 必须是 64 位操作系统, `内核在 3.8 以上`.

通过以下命令查看您的 CentOS 内核：

`[root@localhost ~]# uname -r`

2. 升级本地yum包

`[root@localhost ~]# yum update `

### Docker 安装

Docker 软件包已经包括在默认的 CentOS-Extras 软件源里。因此想要安装 docker，只需要运行下面的 yum 命令：

`[root@localhost ~]# yum install docker`

### 启动 Docker 服务

安装完成后，使用下面的命令来启动 docker 服务，并将其设置为开机启动：

`[root@localhost ~]# service docker start`

`[root@localhost ~]# chkconfig docker on`

（LCTT 译注：此处采用了旧式的 sysv 语法，如采用CentOS 7中支持的新式 systemd 语法，如下：

`[root@localhost ~]# systemctl start docker.service`

`[root@localhost ~]# systemctl enable docker.service`

### 设置镜像加速

```json
# /etc/docker/daemon.json
{
  "registry-mirrors": ["https://o7zhcmyv.mirror.aliyuncs.com"]
}
```

### 注意点

1. 挂载宿主机路径时，如果容器内挂载的路径不存在，则会创建；如果存在，宿主机对应路径目录放入容器中文件或目录（容器目录下的原文件会被删除）。

2. 容器删除时，宿主机挂载的文件及目录不会被删除。

### 容器时间调整

```
RUN echo "Asia/Shanghai" > /etc/timezone && ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
```


### 安全规则

1. 删除已有规则

先查看规则的编号，然后删除指定的规则：

```sh
$ iptables -L --line-numbers
$ iptables -D INPUT 3
# 清除预设表filter中的所有规则链的规则
$ iptables -F
# 清除预设表filter中使用者自定链中的规则
$ iptables -X

# 设定预设规则
$ iptables -P INPUT DROP
$ iptables -P OUTPUT ACCEPT
$ iptables -P FORWARD DROP
```


2. 添加规则

```sh
#SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
#HTTP
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
#HTTPS
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
#POP3
iptables -A INPUT -p tcp --dport 110 -j ACCEPT
#SMTP
iptables -A INPUT -p tcp --dport 25 -j ACCEPT
#FTP
iptables -A INPUT -p tcp --dport 21 -j ACCEPT
iptables -A INPUT -p tcp --dport 20 -j ACCEPT
#DNS
iptables -A INPUT -p tcp --dport 53 -j ACCEPT

#打开主动模式21端口
iptables -A INPUT -p tcp --dport 21 -j ACCEPT

# 打开被动模式50000~50020之间的端口
iptables -A INPUT -p tcp --dport 50000:50020 -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED -j ACCEPT

# 允许 icmp
iptables -A OUTPUT -p icmp -j ACCEPT (OUTPUT设置成DROP的话)
iptables -A INPUT -p icmp -j ACCEPT  (INPUT设置成DROP的话)
```

3. 保存iptables规则

```sh
$ /usr/sbin/iptables-save
```