---
layout:     post
title:      "Shiro 架构"
subtitle:   ""
date:       2017-12-23
author:     "jiangydev"
header-img: "img/post-bg-shiro.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Shiro
---

[TOC]

# Get Start with Apache Shiro 4

## Apache Shiro Architecture

Apache Shiro 的设计目标是通过直观（intuitive）易用来简化应用程序安全。Shiro 的核心设计模型（大多数人如何看待应用程序安全）是某人/某物与一个应用程序交互的上下文。

软件应用程序通常基于用户的情景设计，就是说，你经常基于用户如何与软件交互来设计用户界面或服务 APIs。例如，如果与应用程序交互的用户已经登录，给他呈现查看用户信息的按钮。如果用户未登录，给他呈现注册按钮。

Shiro 在它的设计中反映了这些概念。

#### 高级概观

在最高概念化等级，Shiro 框架有3个主要概念：Subject, SecurityManager, Realms.

<div align = "center">
  ![Subject, SecurityManager, Realms 的交互图](http://shiro.apache.org/assets/images/ShiroBasicArchitecture.png)
</div>

1. Subject

  正如前面提及的，Subject 是必要的当前执行用户的一个特定安全“View”。可以是三方服务，守护账号，cron 任务或其他类似的当前与软件交互的任何事物。

  Subject 实例绑定在 SecurityManager 上，当你与 Subject 交互时，这些交互信息会翻译成与 SecurityManager 交互的特定的信息。

2. SecurityManager

  SecurityManager 是 Shiro 的核心的体系结构, 它是一种 '伞形' 对象, 它协调其内部安全组件, 共同构成一个对象图。一旦为应用程序配置了 SecurityManager 及其内部对象关系图，通常会单独使用它，并且几乎把所有时间放在 Subject API 上。

  它的细节将在以后讨论，但是必须认识到当你与 Subject 交互，真的是 SecurityManager 在幕后为所有的 Subject 安全操作提供繁重的 lifting。这反映在上面的基本流程图中。

3. Realms

  Realms 在 Shiro 和你的应用程序的安全数据间充当桥梁或连接器。当需要与安全相关的数据 (如用户帐户) 进行实际交互以执行身份验证 (登录) 和授权 (访问控制) 时, 它会从为应用程序配置的一个或多个域中查找其中的许多内容。

  域是一个特定安全的 DAO：它为数据源封装了连接细节并使得相关数据对 Shiro 可获得。当配置 Shiro 时，你必须特定至少一个域来身份认证或授权。SecurityManager 可以被配置多个域。

  Shiro 提供独特的（out-of-the-box，创造性的）域来连接许多安全数据源（又名目录，aka directory）例如 LDAP，关系型数据库（JDBC），文本配置源像 INI 和属性文件或更多。如果默认域不能满足你的需要，你可以插入你自己的域实现来实现自定义数据源。

#### 架构细节

  <div align = "center">
    ![Shiro 核心架构细节概念图](http://shiro.apache.org/assets/images/ShiroArchitecture.png)
  </div>

  Subject (`org.apache.shiro.subject.Subject`)

    当前与软件交互的实体的特定安全视图。

  SecurityManager (`org.apache.shiro.mgt.SecurityManager`)

    Shiro 架构的核心。协调它管理的组件来确保平顺地一起工作。它管理每个应用程序用户的视图，所以它知道怎样为每个用户执行安全操作。

  Authenticator (`org.apache.shiro.authc.Authenticator`)

    Authenticator 是负责对用户身份认证（登录）尝试的执行和反应的组件。当一个用户尝试登录，Authenticator 执行逻辑。Authenticator 知道如何协调一个或多个存储相关用户/账号信息的域。从那些域中获得的数据被用来验证用户身份。
    
    Authentication Strategy (`org.apache.shiro.authc.pam.AuthenticationStrategy`)
    
      如果配置了超过一个域，AuthenticationStrategy 会协调域来决定在哪种身份认证尝试成功或失败的情况。（例如，如果一个域验证成功了，当其他的失败了，那这次尝试是否成功？必须所有的域都验证成功吗？仅仅一个验证成功就行了？）

  Authorizer (org.apache.shiro.authz.Authorizer)

    Authorizer 是负责决定在应用程序中用户访问控制的组件。通常的说法是一个用户是否允许做某事的机制。像 Authenticator 一样，Authorizer 也知道如何协调多后端 (back-end) 数据源来访问角色和权限信息。Authorizer 使用这个信息来准确决定一个用户是否允许给操作的行为。

  SessionManager (`org.apache.shiro.session.mgt.SessionManager`)

    SessionManager 知道怎样创建和管理用户会话的生命周期，在任何环境为用户提供一个强壮的 (robust) 会话体验。在安全框架的世界，这是一个独特的功能。Shiro 有能力在任何环境管理用户会话，即使没有 Web/Servlet 或 EJB 容器。Shiro 默认使用一个可获得的、已存在的会话机制，但如果没有的话，例如在独立的应用程序或非 Web 环境，它会使用内建的企业会话管理来提供相同的编程体验。存在 SessionDAO 来允许任何数据源被用来保持会话。
    
      SessionDAO (`org.apache.shiro.session.mgt.eis.SessionDAO`)
    
        SessionDAO 代表 SessionManager 执行 Session 的持久化操作 (CRUD). 这允许任何 data store 被插入 Session 管理基础设施。

  CacheManager (`org.apache.shiro.cache.CacheManager`)

    CacheManager 创建和管理缓存实例的生命周期供其他 Shiro 的其他组件使用。由于 Shiro 能为身份认证、授权和会话管理访问许多后端数据源，在使用这些数据源时，缓存总是框架中的 first-class 结构功能。任何现代的开源或企业缓存产品都能插入 Shiro 来提供快速有效的用户体验。

  Cryptography (`org.apache.shiro.crypto.*`)

    Cryptography 是对企业安全框架自然的补充。Shiro 的密码包包含易于使用和理解的加密学密码、哈希（又名文摘）和不同编解码器实现表示。包中所有的类都被仔细设计成简单易于理解。任何使用 Java 原生加密支持的人都知道这就像难驯的动物。Shiro 的加密 APIs 简化了复杂的 Java 机制并使得加密对人来说易于使用。

  Realms (`org.apache.shiro.realm.Realm`)

    Realms 在 Shiro 和你的应用程序安全数据间充当桥梁和连接器。当与像用户账户的安全相关数据交互来执行身份认证（登录）和授权（访问控制）时，Shiro 从一个或多个配置好的 Realms 寻找这些东西。你可以配置你需要足够多的 Realms（通常每个数据源一个），Shiro 将为身份认证和授权协调它们。

#### The SecurityManager

因为 Shiro 的 API 鼓励一种以 Subject 为中心的编程方式，所以大多数应用程序开发者很少直接与 SecurityManager 交互（但是框架开发者发现有时是有用的）。即使这样，知道 SecurityManager 如何让工作仍很重要，尤其是为一个应用程序配置它。

1. Design

  如之前的叙述，应用程序的 SecurityManager 为所有使用者执行安全操作并管理状态。

  SecurityManager 的默认实现：

    * Authentication
    * Authorization
    * Session Management
    * Cache Management
    * Real Management
    * Event Management
    * "Remember Me" Service
    * Subject creation
    * Logout and mores

  但试图在单一组件中管理是一个很大的功能，并且如果所有的东西组成单一的实现类，使它们具有灵活性和可自定义的将会很困难。

  为了简化配置并启用灵活的配置，Shiro 的实现都在设计上高度模块化。因此实际上 SecurSecurityManager 实现的模块化（以及它的类层次结构）根本不用做很多事。相反，SecurityManager 实现主要充当轻量级的容器组件，将几乎所有的行为委派给嵌套/包装 (nested/wrapped) 的组件。这个包装设计反映在上面的详细体系结构图中。

  当组件实际执行逻辑时，SecurityManager 实现知道怎样为正确的行为协调组件。

  SecurityManager 实现和组件都是兼容 JavaBeans 的，这允许你（或配置机制）通过标准 JavaBeans 的访问/设置方法（即 get*/set*）来自定义可插入组件。这意味着 Shiro 的架构模块能为自定义行为转换成非常简单的配置。

  > 简单配置

  > 因为 JavaBeans 的兼容性，可以通过任何支持 JavaBeans 类型配置的机制（例如 Spring, Guice, JBoss 等）非常容易地配置带有自定义组件的 SecurityManager。


**A step-by-step tutorial to enable Shiro in a web application**

  [Apache Shiro Beginner’s Webapp Tutorial](http://shiro.apache.org/webapp-tutorial.html): a step-by-step tutorial to enable Shiro in a web application on 19 November 2013 by Les Hazlewood
