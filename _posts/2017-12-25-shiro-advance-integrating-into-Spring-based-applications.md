---
layout:     post
title:      "Shiro 集成到 Spring 应用中"
subtitle:   ""
date:       2017-12-24
author:     "jiangydev"
header-img: "img/post-bg-shiro.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Shiro
---

[TOC]

# Integrating Apache Shiro into Spring-based Applications

[原文地址](https://shiro.apache.org/spring.html)

本页涉及集成 Shiro 到基于 Spring 应用程序的方式。

Shiro 的 JavaBeans 兼容性使它完美适合通过 Spring XML 或其他基于 Spring 的配置机制来配置。Shiro 应用程序需要一个应用程序单例 `SecurityManager` 实例。注意，这不一定是一个静态单例，但应该只有一个实例被应用程序使用，无论它是不是一个静态单例。

## 独立的应用程序（Standalone Applications）

这有一个最简单的方式在 Spring 应用程序中启用应用程序单例 `SecurityManager`：

```xml
<!-- 定义你想使用的 realm 来连接你的后端安全数据源: -->
<bean id="myRealm" class="...">
    ...
</bean>

<bean id="securityManager" class="org.apache.shiro.mgt.DefaultSecurityManager">
    <!-- 单 realm app. 如果你想多 realms, 使用 'realms' 属性替换. -->
    <property name="realm" ref="myRealm"/>
</bean>

<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>

<!-- 为了最简单的集成, 以便于在所有情况下，所有的 SecurityUtils.* 方法生效, -->
<!-- 使 securityManager bean 为一个静态单例. 不要在 web 应用程序中这个 -->
<!-- 看下面 'Web Applications' 部分. -->
<bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
    <property name="staticMethod" value="org.apache.shiro.SecurityUtils.setSecurityManager"/>
    <property name="arguments" ref="securityManager"/>
</bean>
```

## Web Applications

Shiro 对 Spring Web 应用程序提供了一流的支持。在一个 web 应用程序中，所有 Shiro 可访问的 web 请求必须通过一个主 Shiro 过滤器。这个过滤器本身非常强大，允许任何基于 URL 路径表达式来执行特定的（ad-hoc）自定义过滤器链。

在 Shiro 1.0 之前，你必须在 Spring web 应用程序中使用混合方法，在 web.xml 中定义 Shiro 过滤器及其所有配置属性，但在 Spring XML 中定义 SecurityManager. 这有些沮丧，因为你不能

  1. 在一处地方巩固你的配置
  2. 利用更高级的 Spring 功能的配置能力，像 `PropertyPlaceholderConfigurer` 或抽象 beans 来巩固常见配置。

现在，在 Shiro 1.0 及以后，所有 Shiro 配置在 Spring XML 中完成，提供更健壮的 Spring 配置机制。

这是在一个基于 Spring 的 web 应用程序中如何配置 Shiro.

### web.xml

除了你其他的 Spring web.xml 元素（`ContestLoaderListener`, `Log4jConfigListener` 等），定义接下来的过滤器和过滤映射：

```xml
<!-- The filter-name matches name of a 'shiroFilter' bean inside applicationContext.xml -->
<filter>
    <filter-name>shiroFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    <init-param>
        <param-name>targetFilterLifecycle</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>

...

<!-- 确保任何你想通过 Shiro 的请求被过滤. 捕捉所有请求。 -->
<!-- 为了确保 Shiro 在过滤器链，在子序列过滤器中工作，通常这个过滤器映射被定义在第一个（在所有其他的前面） -->
<filter-mapping>
    <filter-name>shiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

### applicationContext.xml

在你的 applicationContext.xml 文件中，定义启用 web 的 `SecurityManager` 和来自 `web.xml` 引用的“ShiroFilter” bean.

```XML
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
    <property name="securityManager" ref="securityManager"/>
    <!-- 如果你愿意，为特定应用程序 URLs 重写这些:
    <property name="loginUrl" value="/login.jsp"/>
    <property name="successUrl" value="/home.jsp"/>
    <property name="unauthorizedUrl" value="/unauthorized.jsp"/> -->
    <!-- "filter" 属性是不必要的, 因为定义的任何已声明的 javax.servlet.Filter bean 将 -->
    <!-- 通过链定义中的 beanName 自动获取和可用, 但如果您愿意, 可以在此处执行实例重写或命名别名: -->
    <!-- <property name="filters">
        <util:map>
            <entry key="anAlias" value-ref="someFilter"/>
        </util:map>
    </property> -->
    <property name="filterChainDefinitions">
        <value>
            # 一些链定义例子:
            /admin/** = authc, roles[admin]
            /docs/** = authc, perms[document:read]
            /** = authc
            # 这里更多 URL-to-FilterChain 定义
        </value>
    </property>
</bean>

<!-- 在这个应用程序上下文定义任何你想任意处的 javax.servlet.Filter 的 beans。 -->
<!-- 他们将自动获得上面的 ' shiroFilter ' bean, 并提供给 ' filterChainDefinitions ' 属性。 -->
<!-- 或者如果期望的话，你可以手动地/明确地添加它们到 shiroFilter 的 'filters' 映射. 详见它的 JavaDoc。 -->
<bean id="someFilter" class="..."/>
<bean id="anotherFilter" class="..."> ... </bean>
...

<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
    <!-- 单个 realm app. 如果你有多个 realms, 使用 'realms' 属性替换. -->
    <property name="realm" ref="myRealm"/>
    <!-- 默认情况下，Servlet 容器会话将被使用. 取消这行注释来使用 Shiro 的本地会话 : -->
    <!-- <property name="sessionMode" value="native"/> -->
</bean>
<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>

<!-- 定义你想使用的 Shiro Realm 实现来连接你的后端安全数据源: -->
<bean id="myRealm" class="...">
    ...
</bean>
```

## Enabling Shiro Annotations

这两个独立的和 Web 应用程序，你可能想使用 Shiro 的注解来安全检查（例如，`@RequiesRoles`, `@RequiresPermissions` 等）. 这要求 Shiro 的 Spring AOP 集成来为适当的注解类扫描并执行必要的安全逻辑。

这是如何启用这些注解。仅添加这两个 bean 定义到 applicationContext.xml:

```XML
<!-- Enable Shiro Annotations for Spring-configured beans.  Only run after -->
<!-- the lifecycleBeanProcessor has run: -->
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" depends-on="lifecycleBeanPostProcessor"/>
    <bean class="org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor">
    <property name="securityManager" ref="securityManager"/>
</bean>
```

## Secure Spring Remoting

这有两部分 Shiro 的 Spring 远程支持：为客户端远程调用的配置和为服务器端服务接受及处理远程调用的配置。

### Server-side Configuration

当一个远程方法调用来到启用 Shiro 的服务器时，与该 RPC 调用相关的 Subject 必须绑定到接收线程，以便在线程执行期间进行访问。这个通过在 applicationContext.xml 中定义 Shiro 的 `SecureRemoteInvocationExecutor` bean 来完成。

```XML
<!-- Secure Spring remoting:  Ensure any Spring Remoting method invocations -->
<!-- can be associated with a Subject for security checks. -->
<bean id="secureRemoteInvocationExecutor" class="org.apache.shiro.spring.remoting.SecureRemoteInvocationExecutor">
    <property name="securityManager" ref="securityManager"/>
</bean>
```

一旦你已经定义了这个 bean，你必须把它插入到任何你使用的远程 `Exporter` 来导出/暴露你的服务。`Exporter` 实现通过使用的远程机制/协议定义。阅读 Spring 定义 `Exporter` beans 的 [Remoting chapter](http://docs.spring.io/spring/docs/2.5.x/reference/remoting.html).

例如，如果使用基于 HTTP 的远程（注意引用 `secureRemoteInvocationExecutor` bean 属性）：

```xml
<bean name="/someService" class="org.springframework.remoting.httpinvoker.HttpInvokerServiceExporter">
    <property name="service" ref="someService"/>
    <property name="serviceInterface" value="com.pkg.service.SomeService"/>
    <property name="remoteInvocationExecutor" ref="secureRemoteInvocationExecutor"/>
</bean>
```

### Client-side Configuration

当一个远程调用被执行，`Subject` 身份信息必须被附加到远程 payload 上，让服务器知道谁在调用。如果客户端是一个基于 Spring 的，关联通过 Shiro 的 `SecureRemoteInvocationFactory` 完成：

```XML
<bean id="secureRemoteInvocationFactory" class="org.apache.shiro.spring.remoting.SecureRemoteInvocationFactory"/>
```

在你定义了这个 bean 之后，你需要把它插入到使用的指定协议的 Spring 远程 `ProxyFactoryBean`.

例如，如果你使用基于 HTTP 的 remoting（注意关联上面定义的 `secureRemoteInvocationFactory` bean）：

```XML
<bean id="someService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
    <property name="serviceUrl" value="http://host:port/remoting/someService"/>
    <property name="serviceInterface" value="com.pkg.service.SomeService"/>
    <property name="remoteInvocationFactory" ref="secureRemoteInvocationFactory"/>
</bean>
```
