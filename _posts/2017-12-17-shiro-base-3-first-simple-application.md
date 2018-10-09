---
layout:     post
title:      "Shiro 第一个简单应用"
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

# Get Start with Apache Shiro 3

## First Apache Shiro Application

#### 设置

你需要自己创建一个工程或目录完成下面的教程。

1. 测试 Maven 是否安装

  测试命令 `mvn --version`

2. pom.xml

  ```xml
   <?xml version="1.0" encoding="UTF-8"?>
      <project xmlns="http://maven.apache.org/POM/4.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

      <modelVersion>4.0.0</modelVersion>
      <groupId>org.apache.shiro.tutorials</groupId>
      <artifactId>shiro-tutorial</artifactId>
      <version>1.0.0-SNAPSHOT</version>
      <name>First Apache Shiro Application</name>
      <packaging>jar</packaging>

      <properties>
          <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
      </properties>

      <build>
          <plugins>
              <plugin>
                  <groupId>org.apache.maven.plugins</groupId>
                  <artifactId>maven-compiler-plugin</artifactId>
                  <version>2.0.2</version>
                  <configuration>
                      <source>1.5</source>
                      <target>1.5</target>
                      <encoding>${project.build.sourceEncoding}</encoding>
                  </configuration>
              </plugin>

          <!-- 则会个插件仅仅用来测试运行这个小应用程序，在大多数 Shiro 应用中非必需的 -->
              <plugin>
                  <groupId>org.codehaus.mojo</groupId>
                  <artifactId>exec-maven-plugin</artifactId>
                  <version>1.1</version>
                  <executions>
                      <execution>
                          <goals>
                              <goal>java</goal>
                          </goals>
                      </execution>
                  </executions>
                  <configuration>
                      <classpathScope>test</classpathScope>
                      <mainClass>Tutorial</mainClass>
                  </configuration>
              </plugin>
          </plugins>
      </build>

      <dependencies>
          <dependency>
              <groupId>org.apache.shiro</groupId>
              <artifactId>shiro-core</artifactId>
              <version>1.1.0</version>
          </dependency>
          <!-- Shiro 使用 SLF4J 来记录日志. -->
          <dependency>
              <groupId>org.slf4j</groupId>
              <artifactId>slf4j-simple</artifactId>
              <version>1.6.1</version>
              <scope>test</scope>
          </dependency>
      </dependencies>
  </project>
  ```

3. src/main/java/Tutorial.java

  ```java
  import org.apache.shiro.SecurityUtils;
  import org.apache.shiro.authc.*;
  import org.apache.shiro.config.IniSecurityManagerFactory;
  import org.apache.shiro.mgt.SecurityManager;
  import org.apache.shiro.session.Session;
  import org.apache.shiro.subject.Subject;
  import org.apache.shiro.util.Factory;
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;

  public class Tutorial {

      private static final transient Logger log = LoggerFactory.getLogger(Tutorial.class);

      public static void main(String[] args) {
          log.info("My First Apache Shiro Application");
          System.exit(0);
      }
  }
  ```
  现在先不用担心导入语句，不久会理解他们。如果执行这段程序会输出`My First Apache Shiro Application`并退出。

#### 测试运行

测试运行语句：`mvn compile exec:java`

成功运行的结果：

```
... a bunch of Maven output ...
0 [Tutorial.main()] INFO Tutorial - My First Apache Shiro Application
```

#### Shiro 生效

对应用程序的 Shiro 环境而言，SecurtyManager 是核心部分且必须存在，与`java.lang.SecurityManager`不是同一个东西。

1. 配置

  文件所在路径：`src/main/resources/shiro.ini`

  *Shiro 的 SecurityManager 实现和支持的内容兼容所有的 JavaBeans. 这意味着 Shiro 可以通过 XML、YAML、JSON、Grovy Builder markup或更多工具完成配置。当其他无法配置时，可以使用 INI 类型（适应任何环境）*

  ```shell
  [users]
  # user 'root' with password 'secret' and the 'admin' role
  root = secret, admin
  # user 'guest' with the password 'guest' and the 'guest' role
  guest = guest, guest
  # user 'presidentskroob' with password '12345' ("That's the same combination on my luggage!!!" ;)), and role 'president'
  presidentskroob = 12345, president
  # user 'darkhelmet' with password 'ludicrousspeed' and roles 'darklord' and 'schwartz'
  darkhelmet = ludicrousspeed, darklord, schwartz
  # user 'lonestarr' with password 'vespa' and roles 'goodguy' and 'schwartz'
  lonestarr = vespa, goodguy, schwartz

  # -----------------------------------------------------------------------------
  [roles]
  # 'admin' role has all permissions, indicated by the wildcard '*'
  admin = *
  # The 'schwartz' role can do anything (*) with any lightsaber:
  schwartz = lightsaber:*
  # The 'goodguy' role is allowed to 'drive' (action) the winnebago (type) with license plate 'eagle5' (instance specific id)
  goodguy = winnebago:drive:eagle5
  ```
  正如你所看到的，上面配置了静态用户账户集，对于这个第一个应用足够了。在后面的章节。你会明白怎么使用关系型数据库、LADP ActiveDirectory 或其他。

2. 参考配置

  修改 main 方法，创建 SecurityManager 实例。

  ```java
  public static void main(String[] args) {

      log.info("My First Apache Shiro Application");

      // 1. 使用 Factory 模式 ingest（摄入） Shiro INI 文件
      // classpath: 前缀用来告诉 Shiro 加载文件的路径；file:, url: 同样被支持
      Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro.ini");

      // 2. 调用 getInstance() 方法分析 INI 文件并映射配置返回一个 SecurityManager 实例
      SecurityManager securityManager = factory.getInstance();

      // 3. 设置 SecurityManager 为静态单例（memory），通过 JVM 访问。
      // 这种方式在单 JVM，多 Shiro 生效的环境不令人满意的，虽然在这个简单的程序中 ok，但更多复杂的应用环境通常会在特定的应用程序内存中放置 SecurityManager 对象（例如 web app 的 ServletContext 或 Spring, Guice ，JBoss DI 容器等）。
      SecurityUtils.setSecurityManager(securityManager);

      System.exit(0);
  }
  ```

#### 使用 Shiro

Q: 当我们安全化应用程序时，问的最相关的问题就是“当前用户是谁？”或“当前用户是否允许做某事？”。

A: 应用程序通常基于用户情景构建，并且你会希望基于每个用户来呈现或安全化。所以最自然的想法就是基于应用程序内的当前用户。Shiro 的 API 用 **Subject** 的概念解释了“当前用户”。

1. get Subject

  ```java
  // 通过 getSubject() 在独立应用程序中基于用户数据获取当前执行操作的 Subject。
  // 如果是在服务器环境（例如 web app），则基于用户数据相关的当前线程或即将来的请求
  Subject currentUser = SecurityUtils.getSubject();
  ```
  *Subject 和 User 不一样，User 通常和人联系在一起。在安全的世界中，Subject 指人、三方进程，cron job, 守护进程或其他类似的。简单说，Subject 指当前与软件交互的事物。*

2. do with Subject

  ```java
  // 获取当前用户的 Session。
  Session session = currentUser.getSession();
  session.setAttribute( "someKey", "aValue" );
  ```
  Shiro 实例的 Session 提供普通 HttpSession 的大部分，但有些额外的东西和一点不同：它不要求 HTTP 环境。

  如果在一个 web 应用程序中部署，默认的 Session 将基于 HttpSession。但是，在一个非 web 环境中，Shiro 自动使用它的默认的企业会话管理。这意味着可以使用相同的 API，无论部署环境。

  ```java
  if ( !currentUser.isAuthenticated() ) {
    // 从 GUI 收集用户主体和凭据，例如用户名、密码表单，X509 证书, OpenID 等.
    // 在这里，我们将使用用户名/密码
    UsernamePasswordToken token = new UsernamePasswordToken("lonestarr", "vespa");

    //this is all you have to do to support 'remember me' (不要配置 - 内建的!):
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
    // ... catch more exceptions here (maybe custom ones specific to your application?
    catch (AuthenticationException ae) {
        //unexpected condition?  error?
    }

    //print their identifying principal (in this case, a username):
    log.info( "User [" + currentUser.getPrincipal() + "] logged in successfully." );

    //test a role:
    if (currentUser.hasRole("schwartz")) {
        log.info("May the Schwartz be with you!");
    } else {
        log.info("Hello, mere mortal.");
    }

    //test a typed permission (not instance-level)
    if (currentUser.isPermitted("lightsaber:weild")) {
        log.info("You may use a lightsaber ring.  Use it wisely.");
    } else {
        log.info("Sorry, lightsaber rings are for schwartz masters only.");
    }

    //a (very powerful) Instance Level permission:
    if (currentUser.isPermitted("winnebago:drive:eagle5")) {
        log.info("You are permitted to 'drive' the winnebago with license plate (id) 'eagle5'.  " +
                "Here are the keys - have fun!");
    } else {
        log.info("Sorry, you aren't allowed to drive the 'eagle5' winnebago!");
    }

    //all done - log out!
    currentUser.logout();

    System.exit(0);
  }
  ```

#### 总结

希望这个介绍教程帮助您了解如何设置在一个基本的应用, 以及 Shiro 的主要设计概念, Subject 和 SecurityManager。

Q: 如果我不想使用 INI 用户帐户, 而是希望连接到更复杂的用户数据源, 该怎么办？

A: 需要对 Shiro 的体系结构和支持配置机制有更深一点的理解。接下来我们将介绍 Shiro 的架构。
