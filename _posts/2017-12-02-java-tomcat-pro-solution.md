---
layout:     post
title:      "Tomcat 使用中出现的问题及解决方案"
subtitle:   ""
date:       2017-12-02
author:     "jiangydev"
header-img: "img/post-bg-java.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Java
    - Tomcat
---

[TOC]

## Tomcat 防止文件被锁

#### 问题分析

1. 现象

Maven 工程使用`tomcat7-maven-plugin`时，第一次 clean & deploy 项目 build success 后可以正常访问，但是之后对项目进行修改，再次 redeploy & bulid success 后页面出现 404。检查部署的 war 包内容已更新，但 Tomcat 自动部署的项目目录下只有 /WEB-INF/lib/xx.jar，没有其他内容，即在项目部署过程中出现问题。

2. 检查 Tomcat 输出

```
org.apache.catalina.startup.ExpandWar delete
严重: [${path}\${project name}] could not be completely deleted. The presence of the remaining files may cause problems
```

3. 原因：文件被锁

>Tomcat 会为每个应用创建一个单独的 ClassLoader(WebappClassLoader)，负责加载应用使用的 Java 类和资源。

>WebappClassLoader 是 java.net.URLClassLoader 的子类，URLClassLoader 在加载资源的时候会使用 getResource 方法去访问资源，如果资源文件在Jar包里，那就会打开到 Jar 包的 URL 连接，而 URLConnection 缺省会打开一个缓存、将创建过连接的内容缓存起来，一旦内容被缓存，那相应的 Jar 包就会被锁定。

#### 解决方案

1. 配置方法

>修改文件`apache-tomcat-x.x.x\conf\context.xml`中的内容，如下：

```xml
<Context antiJARLocking="true" antiResourceLocking="true">
    ...
</Context>
```
**若 Tomcat 8.x，去掉`antiJARLocking="true"`属性。**

2. 注意点

   - antiJARLocking

     从 Tomcat 5.0 开始，可以配置该属性，但在 Tomcat 8.0 之后被移除。该属性不一定会生效，在一些情况下，资源的获取并不是 WebappClassLoader 去做的，而是其父加载器的 getResource 方法或父类的 findResource 方法去做的，WebappClassLoader 的父类是 URLClassLoader、父加载器是 URLClassLoader 实例。

   - antiResourceLocking

     从 Tomcat 5.5 开始，可以配置该属性.当 antiResourceLocking 设置为true的时候，Tomcat不会锁定应用下的任何文件。将应用的 doc base 移到临时目录下( java.io.tmpdir 默认指向Tomcat的temp目录)，让 Tomcat 不会占用 webapps下的文件。

3. antiResourceLocking 的副作用

   - 延长应用的启动时间，因为多了临时目录的清理和往临时目录拷贝应用内容的操作；
   - 如果不知道这个属性的原理，修改 webapps下应用的JSP，那就不会动态重加载到新的页面内容了，因为应用的 doc base 已经不再在 webapps下；
   - 停止Tomcat的时候，临时目录下实际的doc base会被删掉.


>另：网上有人提供第二种方案

*该方法在 Tomcat 8.5.16，7.0.79 上实测仍然无法解决 jar 资源的占用问题*
- 配置方法

  修改文件`apache-tomcat-x.x.x\conf\server.xml`中的内容，如下：

  ```xml
  <Server>
      ...
      <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" urlCacheProtection="true"/>
      ...
  </Server>
  ```

- 原理

  在 Server（Tomcat的顶级容器）初始化之前就将 URLConnection 的缓存关掉。

- 注意点
  默认未配置，需要手动配，配置后对所有应用生效。



## JavaWeb: 报错信息The superclass "javax.servlet.http.HttpServlet" was not found on the Java Build Path

#### 问题分析

Javaweb工程类中没有添加Tomcat运行时相关类。

#### 解决方案

(1)右击 Web 工程 -> Build Path -> Configure Build Path-> Libraries -> Add Library... -> Server Runtime -> Tomcat Server;
(2)切换到 Java Build Path 界面中的 Orader and Export，选择 Apache Tomcat。

    *注：按以上方法操作时，若打开 Server Runtime 后空白，需要设置Apache服务器。
    设置方法为：Window -> Preferences -> Server -> Runtime Environment -> Search... -> Tomcat 安装目录*