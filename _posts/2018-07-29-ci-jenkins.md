---
layout:     post
title:      "Jenkins + Docker 持续集成方案及案例"
subtitle:   ""
date:       2018-07-29
author:     "jiangydev"
header-img: "img/post-bg-java.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Java
    - Docker
    - Jenkins
    - CI
---

[TOC]

# Jenkins CI 方案及案例

## 1 架构图

![Docker + Jenkins 实现CI](/img/in-post/jenkins/Docker + Jenkins 实现CI.png)



## 2 Jenkins 安装（Docker方式）

### 2.1 前提

1. Jenkins Docker 官方镜像地址：[https://hub.docker.com/_/jenkins/](https://hub.docker.com/_/jenkins/)
2. 

### 2.2 具体步骤

- 在服务器/本地计算机，安装Docker。

  > （不清楚？请移步[Docker安装]()）

- 启动jenkins镜像（若本地无该镜像，会从docker.io拉取），并运行在8080（用户访问界面端口）和5000（仓库端口），该容器名”jiangydev/jenkins“。

  ```sh
  # 简单的
  $ docker run -d --name jiangydev_jenkins -p 8080:8080 -p 50000:50000 jenkinsci/jenkins
  
  # 如果需要挂载本地目录，可以使用下面的语句
  # 要注意目录权限，权限不足可能会导致容器启动失败
  $ docker run -d --name jiangydev_jenkins -p 8080:8080 -p 50000:50000 -v /opt/jenkins:/var/jenkins_home jenkinsci/jenkins
  ```
  当返回一串十六进制数，表示创建容器成功。

- 查看容器运行状态，`Up`则说明正常运行，此时进行下一步操作，否则要根据容器日志检查错误。

  ```sh
  # 查看容器运行状态
  $ docker ps -a
  CONTAINER ID   IMAGE   COMMAND   CREATED    STATUS   PORTS    NAMES
  2e90aab75665  jenkins  "/bin/tini..." 5 minutes ago  Up 5 minutes   0.0.0.0:50000->50000/tcp, 0.0.0.0:8080->8080/tcp   jiangydev_jenkins
  
  # 查看容器的运行日志
  $ docker logs jiangydev_jenkins
  ```

- 使用浏览器访问Jenkins服务，如`http://<ip>:8080`。

  若访问正常，界面如下。

  ![Jenkins开始界面](/img/in-post/jenkins/Jenkins开始界面.png)

  > 小提示：
  >
  > 若遇到页面无响应，请
  >
  > 1. 检查宿主机（云服务器）的防火墙规则是否放行8080端口，（不清楚？请移步[配置防火墙规则]()）；
  > 2. 检查云服务器服务提供商一侧是否配置了安全规则放行8080端口。

- 解锁Jenkins，将容器中`/var/jenkins_home/secrets/initialAdminPassword `文件内密码复制到页面，并点击Continue。

  ```sh
  # 进入Docker Jenkins容器内
  [root@jiangydev ~] $ docker exec -it jiangydev_jenkins /bin/bash
  # 在容器内用cat命令查看密码文件信息
  jenkins@2e90aab75665:/$ cat /var/jenkins_home/secrets/initialAdminPassword
  caa1aec888534665bc9acff7e490e7ed
  ```

  将查看的密码填入`Administrator password `输入框即可。

- 这里选择`Install suggested plugins `，安装推荐的插件

  ![Jenkins Install suggested plugins](/img/in-post/jenkins/Jenkins Install suggested plugins.png)

  需要等待片刻...

  ![Jenkins起步-插件安装界面](/img/in-post/jenkins/Jenkins起步-插件安装界面.png)

- 最后一步，创建第一个管理员用户界面，并`Save and Finish`。

  ![Jenkins起步-创建第一个管理员用户界面](/img/in-post/jenkins/Jenkins起步-创建第一个管理员用户界面.png)

## 3 通用配置

### 3.1 插件安装

安装需要一些时间，等待安装完成即可。

![Jenkins插件安装界面](/img/in-post/jenkins/Jenkins插件安装界面.png)

> 小提示：
>
> 安装后，要等待重启 Jenkins 服务。



### 3.2 凭据添加

凭据用来自动验证用户信息。

这里可以选择验证类型，默认用 用户名/密码 的方式，如有需要，可以对应修改。

![Jenkins-添加凭据](/img/in-post/jenkins/Jenkins-添加凭据.png)



### 3.3 系统设置 - Publish over SSH

#### 3.3.1 目的

项目构建后，通过 SSH 连接目的服务器地址，发送文件和 Command。

#### 3.3.2 安装必要的插件

- 需要安装的插件列表，若已存在，不用安装。

  | 插件名称                                                     | 插件说明                      | 可选性 |
  | ------------------------------------------------------------ | ----------------------------- | ------ |
  | [Publish Over SSH](http://wiki.jenkins-ci.org/display/JENKINS/Publish+Over+SSH+Plugin) | Send build artifacts over SSH | 必选   |

- 安装插件

  > （不清楚？请移步[Jenkins CI 方案-通用配置-插件安装]()）

#### 3.3.3 具体步骤

- 知识点 - SSH 免密登录（不清楚？请移步[Linux运维-SSH 免密登录]()）

> 小提示：
>
> Remote Directory 的设置，会对项目构建使用 Public over SSH 传输文件保存的目录可见性产生影响！！！

![Jenkins-系统配置-public over ssh](/img/in-post/jenkins/Jenkins-系统配置-public over ssh.png)



### 3.5 小技巧

- 手动重启 Jenkins 服务

  除了登录服务器重启 Jenkins 服务外，还可以访问 `http://<JENKINS_URL>/updateCenter`，勾选`安装完成后重启Jenkins(空闲时)`（取消勾选，则会 Cancel 重启操作），如下。

  ![Jenkins-小技巧-重启服务](/img/in-post/jenkins/Jenkins-小技巧-重启服务.png)



## 4 Jenkins + (Git)Gogs + Tomcat 实现 war 包自动化部署

### 4.1 前提及部署思路

#### 4.1.1 前提

- Gogs 仓库中存在一个能够用 Maven 打包成 war 包的项目；
- 有一个 Tomcat，用来远程 deploy 项目。

#### 4.1.2 思路

Maven打包项目成war，然后 deploy 到远程的 Tomcat 上。

类似于使用 Maven Tomcat 构建插件时，执行命令`mvn tomcat7:redeploy`，用到的如下pom.xml配置：

```xml
<!-- Tomcat插件 -->
<plugin>
    <groupId>org.apache.tomcat.maven</groupId>
    <artifactId>tomcat7-maven-plugin</artifactId>
    <version>2.2</version>
    <configuration>
        <url>http://jiangjiangy.xyz:8080/manager/text</url>
        <port>8080</port>
        <path>/</path>
        <username>tomcat</username>
        <password>tomcat</password>
    </configuration>
</plugin>
```

### 4.2 具体步骤

#### 4.2.1 安装必要的插件

- 需要安装的插件列表，若已存在，不用安装。

  | 插件名称                                                     | 插件说明                                                     | 可选性           |
  | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------- |
  | [Deploy to container Plugin](http://wiki.jenkins-ci.org/display/JENKINS/Deploy+Plugin) | This plugin allows you to deploy a war to a container after a successful build. | 必选             |
  | [Generic Webhook Trigger](https://wiki.jenkins-ci.org/display/JENKINS/Generic+Webhook+Trigger+Plugin) | Trigger that can receive any HTTP request, extract any JSONPath/XPath values and trigger a job with those values available as variables. | 必选             |
  | [Gogs plugin](https://wiki.jenkins-ci.org/display/JENKINS/Gogs+Webhook+Plugin) | This plugin adds Gogs integration to Jenkins.                | 使用Gogs时，必选 |

- 安装插件

  > （不清楚？请移步[Jenkins CI 方案-通用配置-插件安装]()）

#### 4.2.2 配置构建环境

项目的构建过程可以由Jenkins完成，此时需要Jenkins所在的 Docker 容器支持 JDK，Maven等，可以在容器内手动安装，但不推荐，Jenkins为我们提供了界面操作。

>小提示：
>
>自动安装：从网络上获取JDK包，并自动安装。
>
>非自动安装：选择本地JDK安装路径。

1. JDK工具配置

   ![Jenkins-JDK工具配置](/img/in-post/jenkins/war/Jenkins-JDK工具配置.png)

2. Maven工具配置

   ![Jenkins-Maven工具配置](/img/in-post/jenkins/war/Jenkins-Maven工具配置.png)

#### 4.2.3 添加构建所需凭据

> （不清楚？请移步[Jenkins CI 方案-通用配置-凭据添加]()）

添加Tomcat、Git仓库身份验证凭据：

![Jenkins-war包所需凭据配置](/img/in-post/jenkins/war/Jenkins-war包所需凭据配置.png)



#### 4.2.4 在Jenkins创建一个项目构建

![Jenkins-创建一个项目构建](/img/in-post/jenkins/war/Jenkins-创建一个项目构建.png)

这里选择`构建一个自由风格的软件项目 `，该选择自定义性强，下面只列出重要的选择项配置。

1. 项目基本信息

   ![Jenkins-Job1](/img/in-post/jenkins/war/Jenkins-war1.png)

2. 源码管理

   - Respository URL 填写 git clone 的地址

   ![Jenkins-Job2-源码管理](/img/in-post/jenkins/war/Jenkins-war2-源码管理.png)

3. 设置构建触发器

   - 设置好身份验证令牌和 Webhook 触发器后，Webhook 钩子的地址为`http://<当前Jenkins服务器地址>:<port>/generic-webhook-trigger/invoke?token=test`

   ![Jenkins-Job3-构建触发器](/img/in-post/jenkins/war/Jenkins-war3-构建触发器.png)

4. 构建

   构建只打包成WAR文件。

   ![Jenkins-Job4-构建](/img/in-post/jenkins/war/Jenkins-war4-构建.png)

5. 构建后操作

   部署到远程 Tomcat，要注意 deploy 的地址会被拼接成`<Tomcat URL>/manager/html`。

   > （不清楚？请移步[使用 Maven Tomcat 插件部署项目]()）

   ![Jenkins-Job5-构建后操作](/img/in-post/jenkins/war/Jenkins-war5-构建后操作.png)

#### 4.2.5 部署一个 Tomcat 容器

Tomcat设置好用户名/密码，要提供给 Jenkins 以实现远程部署。 

> （不清楚？请移步[Docker Tomcat 安装及配置]()）

#### 4.2.6 在Gogs中设置Web钩子

![Jenkins-Gogs Webhook-war](/img/in-post/jenkins/Jenkins-Gogs Webhook-war.png)

#### 4.2.7 检查自动部署是否成功

访问Jenkins，查看构建详细过程及结果。

![Jenkins-查看构建结果](/img/in-post/jenkins/Jenkins-查看构建结果.png)

## 5 Jenkins + (Git)Gogs + 静态文件 实现自动化部署

### 5.1 前提及部署思路

#### 5.1.1 前提

- Gogs 仓库中存在一个项目；
- 有一个可以远程 SSH 登录的服务器（当然，使用本机也是可以的）。

#### 5.1.2 思路

从 Git 仓库拉取项目文件到 Jenkins 本地，使用 Public over SSH 将项目文件再发送到目的服务器相应目录中，并发送一些操作指令。

### 5.2 具体步骤

#### 5.2.1 安装必要的插件

- 需要安装的插件列表，若已存在，不用安装。

  | 插件名称                                                     | 插件说明                      | 可选性 |
  | ------------------------------------------------------------ | ----------------------------- | ------ |
  | [Publish Over SSH](http://wiki.jenkins-ci.org/display/JENKINS/Publish+Over+SSH+Plugin) | Send build artifacts over SSH | 必选   |

- 安装插件

  > （不清楚？请移步[Jenkins CI 方案-通用配置-插件安装]()）



#### 5.2.2 配置构建环境

构建环境无要求

#### 5.2.3 添加构建所需凭据

> （不清楚？请移步[Jenkins CI 方案-通用配置-添加凭据]()）

添加Git仓库身份验证凭据：

![Jenkins-file所需凭据配置](/img/in-post/jenkins/file/Jenkins-file所需凭据配置.png)



#### 5.2.4 在Jenkins创建一个项目构建

![Jenkins-创建一个file项目](/img/in-post/jenkins/file/Jenkins-创建一个file项目.png)

这里选择`构建一个自由风格的软件项目 `，该选择自定义性强，下面只列出重要的选择项配置。

1. 项目基本信息

   ![Jenkins-file1](/img/in-post/jenkins/file/Jenkins-file1.png)

2. 源码管理

   ![Jenkins-file2-源码管理](/img/in-post/jenkins/file/Jenkins-file2-源码管理.png)

3. 设置构建触发器

   ![Jenkins-file3-构建触发器](/img/in-post/jenkins/file/Jenkins-file3-构建触发器.png)

4. 构建

   将拉取的文件发送到目的服务器并发送一些指令。

   > 小提示：
   >
   > 1. Remote directory：受限于系统配置中的远程目录，只能是其子目录。即使使用绝对路径，其根也是系统配置中的远程目录。
   > 2. Exec command 是发送到 SSH 服务器的命令，这里**可以操作到服务器的根目录！**
   > 3. 使用 SSH 方式传文件，会覆盖，而不会删除目的端目录下的原文件。

   ![Jenkins-file4-使用SSH发送构建的文件](/img/in-post/jenkins/file/Jenkins-file4-使用SSH发送构建的文件.png)

5. 构建后操作

   可以无构建后操作，也可以在这里使用邮件发送构建结果等。

#### 4.2.5 搭建一个免密登录 SSH 服务器

设置好用户名/密码或密钥，提供给 Jenkins 以实现远程部署。 

> （不清楚？请移步[Linux运维-SSH 免密登录]()）

#### 4.2.6 在Gogs中设置Web钩子

![Jenkins-Gogs Webhook-war](/img/in-post/jenkins/Jenkins-Gogs%20Webhook-war.png)

#### 4.2.7 检查自动部署是否成功

访问Jenkins，查看构建详细过程及结果。

> 小提示：
>
> Jenkins 能获取到命令执行的结果，若有错误，但不影响部署，则会提示不稳定。

![Jenkins-file5-查看file部署结果](/img/in-post/jenkins/file/Jenkins-file5-查看file部署结果.png)