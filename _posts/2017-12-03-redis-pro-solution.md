---
layout:     post
title:      "Jedis 使用问题及解决方案"
subtitle:   ""
date:       2017-12-03
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
tags:
    - Redis
    - Java
    - Jedis
---

## Jedis Could not get a resource from the pool

#### 问题分析

1. 检查 Redis 所在服务器的防火墙是否放行 Redis 服务的端口；

2. Redis pool （连接池）参数设置存在问题；

3. 编写代码时，使用完资源未释放，导致资源不够用，与 (2) 有关系。


#### 解决方案

1. 关闭防火墙或放行端口 ( 以 CentOS 7.0 为例 )
```shell
# 查看防火墙状态
systemctl status firewalld
# 关闭防火墙
systemctl disable firewalld
# 暂时关闭防火墙
systemctl stop firewalld
# 放行 6379 端口，并使配置生效
firewall-cmd --zone=public --add-port=6379/tcp --permanent
firewall-cmd --reload
```
>CentOS 7.0 版本中，防火墙服务为 firewalled。

2. 调整 Redis pool 的参数
```xml
<!-- Redis 连接池配置 -->
<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
    <!-- 最大连接数 -->
    <property name="maxTotal" value="${redis.maxTotal}" />
    <!-- 最大空闲连接数 -->
    <property name="maxIdle" value="${redis.maxIdle}" />
    <!-- 获取连接时的最大等待毫秒数,小于零:阻塞不确定的时间,默认-1 -->
    <property name="maxWaitMillis" value="${redis.maxWaitMillis}" />
</bean>
```
>`maxActive`: 可用连接实例的最大数目，为负值时没有限制。
>这个属性在 Redis 的新版本中被去除，可以认为是`maxTotal`。

3. 调用资源后需要手动释放
```java
// Jedis 2.9 以上版本使用 close() 方法释放资源
jedisPool.close();
// 以下两种方法是不赞成的（deprecated）
// jedisPool.returnBrokenResource();
// jedisPool.returnResource();
```
