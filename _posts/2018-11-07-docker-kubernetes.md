---
layout:     post
title:      "Docker Kubernetes"
subtitle:   ""
date:       2018-12-22
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Docker
    - Kubernetes
---

[TOC]

## 1 Kubernetes 架构

### 1.1 Kubernetes 重要概念

#### 1.1.1 Namespace

- 解释
  - 资源隔离的一种手段
  - 不同 NS 中的资源不能相互访问
  - 多租户的一种常见解决方案。

- 拥有
  - 名称
- 关联
  - Resource

#### 1.1.2 Resource

- 解释
  - 集群中的一种资源对象；
  - 处于某个命名空间中；
  - 可以持久化存储到 Etcd 中；
  - 资源是有状态的；
  - 资源可以关联；
  - 资源可以限定使用配额；
- 拥有
  - 名称；
  - 类型；
  - Label：用于选择；
  - 资源定义模板；
  - 资源实例；
  - 资源生命周期；
  - 事件记录（events）；
  - 特定的一些动作；
- 关联
  - ResourceQuta；
  - Namespace；

> K8s 中几乎所有重要概念都是资源。



#### 1.1.3 Label

- 解释
  - K-V 键值对；
  - 可以任意定义；
  - 可以贴在 Resource 对象上；
  - 可以过滤和选择某种资源
- 拥有
  - Key，Value
- 关联
  - Resource
  - Label Selector

#### 1.1.4 Master 节点

- 解释
  - K8s 集群的管理节点
  - 负责集群的管理
  - 提供集群的资源数据访问入口
- 拥有
  - Etcd 存储服务（可选）
  - API Server 进程
  - Controller Manager 服务进程
  - Schedular 服务进程
- 关联
  - 工作节点：Node

#### 1.1.5 Node

- 解释
  - 一个 Linux 主机
  - K8s 中的工作节点（负载节点）
  - 启动和管理 K8s 中的 Pod 实例
  - 接受 Master 节点的管理指令
  - 具有一定的自修复能力
- 拥有
  - 名称和 IP
  - 系统资源信息（CPU 内存等）
  - 运行 Docker Engine 服务
  - 运行了 K8s 的一个守护进程 Kubelet
  - 运行了 K8s 的负载均衡 kube-proxy
  - 拥有狂态
- 关联
  - Master 节点

#### 1.1.6 Pod

- 解释
  - 一组容器的一个“单一集合体”
  - K8s 中的最小任务调度单元
  - 可以被调度到任意 Node 上恢复
  - 一个 Pod 里的所有容器共享资源（网络、Volumes）
- 拥有
  - 名称和 IP
  - 拥有状态
  - 拥有 Label
  - 一组容器进程
  - 一些 Volumes
- 关联
  - Node
  - Service

#### 1.1.7 Service

- 解释
  - 微服务
  - 容器方式隔离
  - TCP 服务
  - 通常无状态
  - 可以部署多个实例同时服务
  - 属于内部的“概念”，默认外部无法访问
  - 可以滚动升级
- 拥有
  - 一个唯一的名字
  - 一个虚拟访问位置：IP 地址（Cluster IP）+ Port
  - 一个外部系统访问的映射端口 NodePort
  - 对应后端分布于不同 Node 一组服务容器进程
- 关联
  - 多个相同 Label 的 Pod

![k8s-service](C:\Users\HS\Desktop\jiangydev.github.io\img\in-post\docker\k8s\k8s-service.png)

#### 1.1.8 Replication Controller

- 解释
  - Pod 副本控制器
  - 限定某种 Pod 的当前实例个数
  - 属于服务集群的控制范畴
  - 服务的滚动升级就是靠它来实现
- 拥有
  - 一个标签选择器，选择目标 Pod
  - 一个数字，表明目标 Pod 的期望实例数
  - 一个 Pod 模板，用来创建 Pod 实例
- 关联
  - 多个相同 Label 的 Pod

#### 1.1.9 Volumes

- 解释
  - Pod 上的存储卷
  - 能被 Pod 内的多个容器使用
- 拥有
  - 名字
  - 存储空间
- 关联
  - Pod



Volumes（存储卷）是 Pod 中能够被多个容器访问的共享目录。Kubernetes 的 Volumes 概念与 Docker 的 Volumes 比较类似，但并不完全相同。Kubernetes 中的 Volumes 与 Pod 生命周期相同，而不与容器的生命周期相关，当容器终止或重启时，Volumes 中的数据也不会丢失。另外，Kubernetes 支持多种类型的 Volumes，并且一个 Pod 可以同时使用任意多个 Volumes。



##### 1 Empty Dir

一个 EmptyDir Volume 是在 Pod 分配到 Node 时创建的。从它的名称就可以看出，它的厨师内容为空。同一个 Pod 中所有容器可以读和写 EmptyDir 中的相同文件。当 Pod 从 Node 上移除时，EmptyDir 中的数据也会永久删除。

用途如下：

- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无需永久保留；
- 长时间任务的中间过程 CheckPoint 临时保存目录；
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）。

##### 2 Host Path

在 Pod 上挂载宿主机上的文件或目录

用途如下：

- 容器应用程序生成的日志文件需要永久保存，可以使用宿主机的高速文件系统进行存储；
- 需要访问宿主机上 Docker 引擎内部数据结构的容器应用，可以通过定义 hostPath 为宿主机 /var/lib/docker 目录使得容器内部应用可以直接访问 Docker 的文件系统。

> 小提示
>
> 在不同 Node 上具有相同配置的 Pod 可能会因为宿主机上的目录和文件不同而导致对 Volumes 上目录和文件的访问结果不一致。



### 1.2 Kubernetes 架构及原理

![k8s-architecture](C:\Users\HS\Desktop\jiangydev.github.io\img\in-post\docker\k8s\k8s-architecture.png)



### 1.3 Kubernetes 网络机制

![k8s-networks](C:\Users\HS\Desktop\jiangydev.github.io\img\in-post\docker\k8s\k8s-networks.png)

#### 1.3.1 Service Portal 方式访问 Service

![k8s-networks-serviceportal](C:\Users\HS\Desktop\jiangydev.github.io\img\in-post\docker\k8s\k8s-networks-serviceportal.png)

#### 1.3.2 外部应用通过 NodePort 访问 Service

![k8s-networks-nodeport](C:\Users\HS\Desktop\jiangydev.github.io\img\in-post\docker\k8s\k8s-networks-nodeport.png)










