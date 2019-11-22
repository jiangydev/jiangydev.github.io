---
layout:     post
title:      "Kubernetes 使用问题"
subtitle:   ""
date:       2018-12-25
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - solution
    - Kubernetes
---

[TOC]

> 环境说明：
>
> 物理机
>
> 宿主机 CentOS 7.4 x64

## 1 由于无法拉取镜像导致 Pod 状态一直处于 ContainerCreating

### 问题描述

根据《Kubernetes权威指南第二版》中操作步骤，使用 yaml 文件方式启动一个 RC 类型的 Pod 实例。但使用命令查看 Pod 信息（`kubectl get pods`），状态一直为 ContainerCreating。进一步查看 Pod 的详细信息（`kubectl describe pod mysql-wqtz3`），出现如下错误信息：

```
Error syncing pod, skipping: failed to "StartContainer" for "POD" with ImagePullBackOff: "Back-off pulling image \"registry.access.redhat.com/rhel7/pod-infrastructure:latest\""

Error syncing pod, skipping: failed to "StartContainer" for "POD" with ErrImagePull: "image pull failed for registry.access.redhat.com/rhel7/pod-infrastructure:latest, this may be because there are no credentials on this request.  details: (open /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt: no such file or directory)"
```

### 问题分析

错误信息显示 docker 拉取镜像失败，镜像名称 registry.access.redhat.com/rhel7/pod-infrastructure:latest。拉取失败的原因是，请求中没有证书（本地缺少 redhat-ca.crt，这是一个软链接，指向`/etc/rhsm/ca/redhat-uep.pem`，但是没有这个 pem 文件）。

网上的方案大部分是手动安装 [python-rhsm-1.19.9-1.el7.x86_64.rpm][1]，和 [python-rhsm-certificates-1.19.9-1.el7.x86_64.rpm][2]，安装后会在`/etc/rhsm/ca/` 目录下生成 `redhat-uep.pem` 文件。

**重点：**思路可行，但是手动安装会出现两种状况。

1. 已安装 kubernetes，安装上面两个 rpm 包版本失败。（若只卸载冲突的 `subscription-manager-rhsm` 会导致 docker、kubernetes 依赖包都被移除！）
2. 先手动安装上面两个 rpm 包，再安装 kubernetes 时包被替换为  `subscription-manager-rhsm.x86_64 0:1.21.10-3.el7.centos`。 

> 小提示
>
> 若链接失效，请自行查找相应的 rpm 包。

### 解决方案

最终的效果是， `redhat-uep.pem` 在 `/etc/rhsm/ca/` 目录下（即要确保 kubernetes 安装好的情况下，这个文件是存在的）。

操作步骤如下：

```shell
# 移除当前系统中 rhsm 的包（该步骤会导致 docker, k8s 也被卸载，后续步骤会再安装回来）
$ yum remove subscription-manager-rhsm*
# 获取需要的 rhsm 包
$ wget https://buildlogs.centos.org/c7.1708.00/python-rhsm/20170804153103/1.19.9-1.el7.x86_64/python-rhsm-1.19.9-1.el7.x86_64.rpm
$ wget https://buildlogs.centos.org/c7.1708.00/python-rhsm/20170804153103/1.19.9-1.el7.x86_64/python-rhsm-certificates-1.19.9-1.el7.x86_64.rpm
# 安装下载好的两个 rpm 包，并拷贝 redhat-uep.pem 到根目录备用
$ rpm -ivh python-rhsm* && cp /etc/rhsm/ca/redhat-uep.pem /redhat-uep.pem
# 安装 k8s(docker也会被安装)
$ yum install -y kubernetes
# 将 redhat-uep.pem 拷贝至 /etc/rhsm/ca/
$ cp /redhat-uep.pem /etc/rhsm/ca/redhat-uep.pem
# 接下来启动相应服务即可
```

> 小提示
>
> 若安装 rpm 提示缺少其他依赖包，请自行下载。



## 2 外部无法访问 k8s 部署的服务

### 问题描述

宿主机中可以用 curl 访问到服务，物理机与宿主机可以互相 ping  通，但宿主机无法访问到宿主机中对应端口的服务。

### 问题分析

已关闭防火墙，不明原因

### 解决方案

宿主机执行如下命令。

```shell
$ iptables -P FORWARD ACCEPT
```

kube-proxy 是实现 K8s Services 通信与负载均衡机制的重要组件，基于 iptables，虽然不启动 iptables，但 kube-proxy 还是用 iptables 的规则，因此也不能把 iptalbes 卸载。



