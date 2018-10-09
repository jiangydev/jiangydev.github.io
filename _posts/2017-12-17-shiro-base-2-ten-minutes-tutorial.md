---
layout:     post
title:      "Shiro 十分钟教程"
subtitle:   ""
date:       2017-12-17
author:     "jiangydev"
header-img: "img/post-bg-shiro.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Shiro
---

[TOC]

# Get Start with Apache Shiro 2

## Apache Shiro 十分钟教程

Shiro 能运行在任何环境，从简单的命令行应用程序到最大的企业级 web 和集群应用程序。

#### Download

1. 要求：JDK 1.6+, Maven 3.0.3+

2. 下载 shiro-root-x.x.x-source-release.zip [( Shiro 官方下载页)](http://shiro.apache.org/download.html) 并解压

3. 在 `samples\quickstart\` 目录下执行 `mvn compile exec:java`, 对项目进行编译。

#### Analysis

1. Quickstart.java 文件

```java
public static void main(String[] args) {

  // 最简单的方式是使用配置了域、用户、角色和权限的 INI 配置文件来创建一个 Shiro SecurityManager. （我们使用 Factory 获取 .ini 文件并返回一个 SecurityManager 实例）
  // 这里使用 classpath 下的 shiro.ini，(path: src\main\resources\shiro.ini)
  // (file: and url: prefixes load from files and urls respectively):
  Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro.ini");
  SecurityManager securityManager = factory.getInstance();

  // 这里，我们设置 SecurityManager，可以理解成一个 JVM 单例。
  // 大多数应用程序不采用这种方式，而是依赖容器配置或 web.xml，因此你可以做进一步配置。
  SecurityUtils.setSecurityManager(securityManager);

  // 至此，一个简单的 Shiro 环境已经设置好, 接下来可以做的是:

  // 获取当前执行的用户:
  Subject currentUser = SecurityUtils.getSubject();

  // 用 Session 做一些事 (不需要 web 或 EJB 容器!!!)
  Session session = currentUser.getSession();
  session.setAttribute("someKey", "aValue");
  String value = (String) session.getAttribute("someKey");
  if (value.equals("aValue")) {
      log.info("Retrieved the correct value! [" + value + "]");
  }

  // 登录当前用户，这样我们可以检查角色和权限:
  if (!currentUser.isAuthenticated()) {
      UsernamePasswordToken token = new UsernamePasswordToken("lonestarr", "vespa");
      token.setRememberMe(true);
      try {
          currentUser.login(token);
      } catch (UnknownAccountException uae) {
          log.info("There is no user with username of " + token.getPrincipal());
      } catch (IncorrectCredentialsException ice) {
          log.info("Password for account " + token.getPrincipal() + " was incorrect!");
      } catch (LockedAccountException lae) {
          log.info("The account for username " + token.getPrincipal() + " is locked.  " +
                  "Please contact your administrator to unlock it.");
      }
      // ... 在这捕获更多异常 (可以为你的应用程序自定义异常）
      catch (AuthenticationException ae) {
          // 处理异常
      }
  }

  //say who they are:
  // 输出用户的身份认证主体 (in this case, a username):
  log.info("User [" + currentUser.getPrincipal() + "] logged in successfully.");

  // 测试角色:
  if (currentUser.hasRole("schwartz")) {
      log.info("May the Schwartz be with you!");
  } else {
      log.info("Hello, mere mortal.");
  }

  // 测试类型权限 (非实例级别)
  if (currentUser.isPermitted("lightsaber:weild")) {
      log.info("You may use a lightsaber ring.  Use it wisely.");
  } else {
      log.info("Sorry, lightsaber rings are for schwartz masters only.");
  }

  // 一个非常有效的实例级别权限:
  if (currentUser.isPermitted("winnebago:drive:eagle5")) {
      log.info("You are permitted to 'drive' the winnebago with license plate (id) 'eagle5'.  " +
              "Here are the keys - have fun!");
  } else {
      log.info("Sorry, you aren't allowed to drive the 'eagle5' winnebago!");
  }

  // 所有的做完之后，退出登录
  // 移除所有身份认证信息并使 Session 无效
  currentUser.logout();

  System.exit(0);
}
```
 尽管有一些复杂的东西，在这个罩下，工作如此优雅。

 > Q: 但是谁负责在登录期间获取用户数据 (名和密码、角色和权限等), 以及谁在运行时实际执行这些安全检查？

 > A: 通过实现 Shiro 所谓的 Realm 并将 Realm 塞进 Shiro 的配置。
