---
layout:     post
title:      "Shiro Web Support"
subtitle:   ""
date:       2017-12-17
author:     "jiangydev"
header-img: "img/post-bg-shiro.jpg"
tags:
    - Shiro
---

# Apache Shiro Web Support

[原文地址](https://shiro.apache.org/web.html)

## Configuration

集成 Shiro 到任何 web application 中，最简单的方式是在了解怎样读取 Shiro 的 INI 配置的 web.xml 中配置一个 Servlet ContextListener 和 Filter。INI 配置格式本身的大部分是在 Configuration pages 的 [INI Sections](https://shiro.apache.org/configuration.html#Configuration-INISections) 节定义的，但我们将在这涉及一些其它的 web 特定部分。

> 使用 Spring？
>
> Spring Framework 用户将不用执行这个步骤，如果你使用 Spring，你将想阅读 [Spring-specific web configuration](https://shiro.apache.org/spring.html#[[#]]#Spring-WebApplications).

### web.xml

#### Shiro 1.2 and later

Shiro 1.2 及以后，标准 web applications 初始化 Shiro 通过添加接下来的 XML 块到 web.xml:

```xml
<listener>
    <listener-class>org.apache.shiro.web.env.EnvironmentLoaderListener</listener-class>
</listener>

...

<filter>
    <filter-name>ShiroFilter</filter-name>
    <filter-class>org.apache.shiro.web.servlet.ShiroFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>ShiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>FORWARD</dispatcher>
    <dispatcher>INCLUDE</dispatcher>
    <dispatcher>ERROR</dispatcher>
</filter-mapping>
```

这假定一个 Shiro INI 配置文件被放在接下来的两个位置，使用首先找到的那个(按顺序)：

  1. /WEB-INF/shiro.ini
  2. classpath:shiro.ini

上面的配置做了以下的事：

  * `EnvironmentLoaderListener` 初始化一个 Shiro WebEnvironment 实例（包含 Shiro 需要操作的任何东西，包含 SecurityManager）并使它在 ServletContext 可拿到。如果你需要在任何时候获得这个 WebEnvironment 实例，你可以调用 `WebUtils.getRequiredWebEnvironment(servletContext)`.
  * ShiroFilter 将用这个 WebEnvironment 为任何过滤的请求执行所有必要的安全操作。
  * 最后，定义 filter-mapping 确保所有请求通过 ShiroFilter 被过滤, 被大多数 web applications 推荐来确保任何请求是安全的。

  > ShiroFilter filter-mapping
  >
  > 通常期望在任何其他“filter-mapping”声明前定义，来确保 Shiro 可以在那些过滤中工作。

##### Custom WebEnvironment Class

通过默认的 `EnvironmentLoaderListener` 将创建一个基于 INI 配置的 IniWebEnvironment 实例。如果你喜欢，你可以特定一个自定义 WebEnvironment 实例，通过特定一个 ServletContext context-param 在 web.xml 中:

```xml
<context-param>
    <param-name>shiroEnvironmentClass</param-name>
    <param-value>com.foo.bar.shiro.MyWebEnvironment</param-value>
</context-param>
```

这允许你自定义如何一个配置格式被分析和呈现为一个 WebEnvironment 实例。您可以为自定义行为 IniWebEnvironment 现有的子类, 或完全支持不同的配置格式。例如, 如果有人想在 XML 中而不是 INI 中配置, 他们就可以创建基于 xml 的实现, 例如,`com.foo.bar.shiro.XmlWebEnvironment`。

##### Custom Configuration Locations

除了上面提到的默认找到的位置，如果你希望把配置放在其他的位置，你可以在 web.xml 中用 context-param 特定位置：

```xml
<context-param>
    <param-name>shiroConfigLocations</param-name>
    <param-value>YOUR_RESOURCE_LOCATION_HERE</param-value>
</context-param>
```

默认地，param-value 通过 ServletContext.getSource 方法定义的规则解析。例如，`/WEB-INF/some/path/shiro.ini`. 但你也可以通过使用一个由 Shiro ResourceUtils class 支持的属性源前缀，特定文件系统、classpath 或 URL 位置，例如：

  * file:/home/foobar/myapp/shiro.ini
  * classpath:com/foo/bar/shiro.ini
  * url:http://confighost.mycompany.com/myapp/shiro.ini

#### Shiro 1.1 and earlier

在 1.1 及以前的 web applications，最简单的方式使 Shiro 生效是定义 IniShiroFilter 并特定一个 filter-mapping:

```xml
<filter>
    <filter-name>ShiroFilter</filter-name>
    <filter-class>org.apache.shiro.web.servlet.IniShiroFilter</filter-class>
</filter>

...

<!-- 确保任何你想要到 Shiro 的请求被过滤.捕获所有请求. 通常这个 filter mapping 被定义在第一个 (在其他所有的之前) 来确保 Shiro 工作在过滤链的子序列过滤器: -->
<filter-mapping>
    <filter-name>ShiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>FORWARD</dispatcher>
    <dispatcher>INCLUDE</dispatcher>
    <dispatcher>ERROR</dispatcher>
</filter-mapping>
```

这个定义期待你的 INI 配置在 classpath 根目录的 shiro.ini 文件中（例如：classpath:shiro.ini）。

##### Custom Path

```xml
<filter>
    <filter-name>ShiroFilter</filter-name>
    <filter-class>org.apache.shiro.web.servlet.IniShiroFilter</filter-class>
    <init-param>
        <param-name>configPath</param-name>
        <param-value>/WEB-INF/anotherFile.ini</param-value>
    </init-param>
</filter>
```

不合格的（schemeless 或无前缀） configPath 值被假定为 ServletContext 资源路径，可通过由 ServletContext. getResource 方法定义的规则解析。

> ServletContext 资源路径 - 1.2+
>
> ServletContext 资源路径在 Shiro 1.2 和以后可获得的，在 1.1 及以前，所有的 configPath 定义必须特定一个前缀。

例如：

```xml
<init-param>
    <param-name>configPath</param-name>
    <param-value>url:http://configHost/myApp/shiro.ini</param-value>
</init-param>
```

##### Inline Config

最后，有可能根本不是用 INI 文件将你的 INI 配置嵌入在内联 web.xml 中，使用 `config init-param` 而不是 configPath:

```xml
<filter>
    <filter-name>ShiroFilter</filter-name>
    <filter-class>org.apache.shiro.web.servlet.IniShiroFilter</filter-class>
    <init-param><param-name>config</param-name><param-value>

    # INI Config Here

    </param-value></init-param>
</filter>
```

内嵌配置通常适用于小型或简单的应用程序, 但通常更方便地将它放在一个专用的 Shiro.ini 文件中, 原因如下:

  * 您可能会编辑安全配置, 并且不希望将修订控件 'noise' 添加到 web.xml 文件
  * 您可能希望将安全配置与 web.xml 配置的其余部分分开
  * 您的安全设置可能会变得很大, 并且您希望保持 web.xml 的精简和易于阅读
  * 你有一个复杂的构建系统, 在这里可能需要相同的在多个地方被引用

这取决于你，使用什么对你的项目有意义。

### Web INI configuration

另外的标准 [main], [users], 和 [roles]  部分已经在 Configuration 章节描述了，你可以在你的 shiro.ini 文件中另外指定一个特定的 web [urls] 部分。

```ini
# [main], [users] and [roles] above here
...
[urls]
...
```

`[urls]` 部分允许你做在任何不存在 web 架构中，我们也曾见过的：在你的应用程序中，为匹配的 URL 路径定义点对点过滤链。

这比你正常在 `web.xml` 中定义过滤器链的方式更灵活，更强大和简洁: 即使你从来没有用过 Shiro 提供的其它功能和只是这一个，单个它就值得使用。

#### [urls]

在 `[urls]` 部分每行的格式如下：

```
_URL_Ant_Path_Expression_ = _Path_Specific_Filter_Chain_
```

例如：

```
...
[urls]

/index.html = anon
/user/create = anon
/user/** = authc
/admin/** = authc, roles[administrator]
/rest/** = authc, rest
/remoting/rpc/** = authc, perms["remote:invoke"]
```

接下来，我们将实际介绍这些行的意思。

##### URL Path Expressions

等号左边的 Token 是一个 Ant 样式路径表达式，与你的 web 应用程序的上下文根路径相关。

例如，让我们告诉你接下来的 [urls] 行:

```
/account/** = ssl, authc
```

这行陈述了，任何到我的应用程序路径 `/account` 或它的子路径（`/account/foo`, `/account/bar/baz`, 等）的请求，将触发 "ssl, authc" 过滤链。我们将在下面涉及过滤链。

记住，所有路径表达式与你的应用程序上下文根路径有关。这意味着，如果你某天部署应用程序到 `www.somehost.com/myapp`, 并且之后部署到 `www.anotherhost.com`(没有 "myapp" 子路径)，这个匹配模式仍然有效。所有的路径与 HttpServletRequest.getContextPath() 的值有关。

> 顺序重要！
>
> URL 路径表达式根据传入请求的定义顺序进行计算, 而第一个匹配项将获胜。例如，假设有如下链定义：
```
/account/** = ssl, authc
/account/signup = anon
```
> 如果一个传入的请求为了到达 `/account/signup/index.html`（所有的匿名用户“anon”可以访问），它将永远不会被处理！原因是，`/account/**` 模式首先匹配了传入的字符串，并且短路了所有剩余的定义。
>
> 总是记住基于一个 **第一条获胜** 的策略定义你的过滤链。

##### Filter Chain Definitions

等号右边的 Token 是以逗号分隔的过滤器列表，用于执行与该路径匹配的请求。它必须与下面的格式匹配：

```
filter1[optional_config1], filter2[optional_config2], ..., filterN[optional_configN]
```

where:

  * filterN 是定义在 `[main]` 部分的一个过滤器 bean；
  * `[optional_configN]` 是一个可选的字符串，它对特定的路径有特定筛选（每一个过滤器，特定路径配置！）。如果过滤器不需要给 URL 路径特定配置，你可以丢弃括号，所以 `filterN[]` 就变成 `filterN`.

因为过滤器 tokens 定义为链（也称列表），所以记住顺序重要性！按你希望流经链的顺序定义逗号分隔列表。

最后，每个过滤器都可以自由地处理它想要的响应，如果它的必要条件不被满足（例如执行重定向、用 HTTP 错误码响应、直接渲染，等）。否则它将允许请求继续通过链到最终的目的视图。

> Tip
>
> 能够对路径特定配置做出反应（例如，一个过滤器 token 的 `[optional_configN]` 部分），是 Shiro 过滤器有的唯一的功能。
>
> 如果你想创建自己的 `javax.servlet.Filter` 实现，也可以这样做，确保你的过滤器子类 `[org.apache.shiro.web.filter.PathMatchingFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/PathMatchingFilter.html)`.

###### Avaliable Filters

在 `[main]` 部分定义了可用于过滤器链定义的过滤器“池”。在 `[main]` 部分分配给他们的名称是要在过滤器链定义中使用的名称。例如：

```
[main]
...
myFilter = com.company.web.some.FilterImplementation
myFilter.property1 = value1
...

[urls]
...
/some/path/** = myFilter
```

## Default Filters

当运行一个 web-app，Shiro 将创建一些有用的默认 `Filter` 实例，并让它们在 `[main]` 部分自动可获得。你可以在 `[main]` 中配置它们，正如你想要的其他 bean, 并在你的链定义中引用它们。例如：

```
[main]
...
# 注意我们如何不为 FormAuthenticationFilter 定义类 ('authc') - 它已经实例化并可获得:
authc.loginUrl = /login.jsp
...

[urls]
...
# 确保 end-user 被身份认证. 如果没有, 重定向到上面的 'authc.loginUrl', 并且在成功身份认证之后, 重定向他们回到尝试看的原始账户页面:
/account/** = authc
...
```

默认自动可获得的过滤器实例由 [DefaultFilter enum](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/mgt/DefaultFilter.html) 定义，并且枚举的 `name` 域是配置可获得的名字。它们如下：

|Filter Name |Class |Feature |
|:---|:---:|:---|
|anon | [org.apache.shiro.web.filter.authc.AnonymousFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/AnonymousFilter.html)|允许立即访问路径, 而不执行任何类型的安全检查。通常用于排除策略。
|authc |[org.apache.shiro.web.filter.authc.FormAuthenticationFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/FormAuthenticationFilter.html)|要求请求用户对请求进行身份验证以继续, 如果没有, 则强制用户通过将其重定向到您配置的 loginUrl 进行登录。
|authcBasic |[org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/BasicHttpAuthenticationFilter.html)|要求请求的用户被身份认证以继续，如果不是，则要求用户通过HTTP基本协议特定的挑战进行登录。登录成功后，他们可以继续使用请求的资源/ URL。
|logout |[org.apache.shiro.web.filter.authc.LogoutFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/LogoutFilter.html)|一旦接收到请求，将立即注销当前正在执行的 Subject, 然后将它们重定向到配置的 redirectUrl。
|noSessionCreation |[org.apache.shiro.web.filter.session.NoSessionCreationFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/session/NoSessionCreationFilter.html)|一个 PathMatchingFilter 将禁止在请求期间创建新的会话。这是一个有用的过滤器，可放置在任何可能导致 REST，SOAP 或其他服务调用无意参与会话的过滤器链的前端。
|perms |[org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/PermissionsAuthorizationFilter.html)|如果当前用户具有由映射值指定的权限，则允许访问;如果用户没有指定所有权限，则拒绝访问。
|port |[org.apache.shiro.web.filter.authz.PortFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/PortFilter.html)|要求请求位于特定端口上，如果不是，则重定向到该端口上的相同URL。
|rest |[org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/HttpMethodPermissionFilter.html)|将 HTTP 请求的方法 (如 GET、 POST 等) 转换为相应动作(谓词), 并使用该谓词构造一个将被检查以确定访问的权限。
|roles |[org.apache.shiro.web.filter.authz.RolesAuthorizationFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/RolesAuthorizationFilter.html)|如果当前用户具有由映射值指定的角色, 则允许访问, 如果用户没有指定的所有角色, 则拒绝访问。
|ssl |[org.apache.shiro.web.filter.authz.SslFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/SslFilter.html)|需要通过 SSL 请求的过滤器，如果在配置的服务器端口收到请求并且 `request.isSecure()` 为 true，则访问允许。如果任一条件为 false，过滤器链不会继续。
|user |[org.apache.shiro.web.filter.authc.UserFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/UserFilter.html)|如果访问者是已知用户 (定义为具有已知主体), 则允许对资源进行存取。这意味着任何被身份验证或通过 '记住我' 功能进行记忆的用户都将被允许从该筛选器访问。

## Enabling and Disabling Filters

与任何过滤器链定义机制 (`web.xml`、Shiro's INI 等) 一样, 只要将过滤器包含在过滤器链定义中, 就可以启用它, 并通过从链定义中移除来禁用它。

但是, 在 Shiro 1.2 中添加的一个新功能是能够在不从过滤器链中删除过滤器的情况下启用或禁用过滤。如果启用 (默认设置), 则将按预期方式筛选请求。如果禁用, 则筛选器将允许请求立即传递到 `FilterChain` 中的下一个元素。您可以根据配置属性触发过滤器的启用状态, 也可以根据每个请求触发它。

这是一个强大的概念, 因为根据某些要求启用或禁用过滤器通常更方便, 而不是更改静态筛选器链定义, 这将是永久性的和不灵活的。

Shiro 通过它的 `[OncePerRequestFilter](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/servlet/OncePerRequestFilter.html)` 抽象父类完成这个。所有 Shiro 的现成的过滤器实现子类这一个, 因此能够不从过滤链中删除它们且被启用或禁用。如果您还需要此功能, 则可以为自己的过滤器实现子类。

### General Enabling/Disabling

OncePerRequestFilter(及其所有子类)支持在所有请求以及基于每个请求的基础上启用/禁用。

对所有请求启用或禁用筛选器的一般做法是将其 `enabled` 属性设置为 true 或 false。默认设置为 `true`, 因为如果在链中配置了大多数筛选器, 则本质上需要执行。

例如，在 shiro.ini:

```ini
[main]
...
# configure Shiro's default 'ssl' filter to be disabled while testing:
ssl.enabled = false

[urls]
...
/some/path = ssl, authc
/another/path = ssl, roles[admin]
...
```

这个例子展示了可能有许多 URL 路径都要求必须通过 SSL 连接保护请求。在开发中设置 SSL 会让人感到沮丧和费时。在开发时, 可以禁用 ssl 筛选器。在部署到生产中时, 您可以使用一个配置属性启用它-这比手动更改所有 URL 路径或维护两个 Shiro 配置更容易。

### Request-specific Enabling/Disabling

`OncePerRequestFilter` 实际上确定筛选器启用还是禁用是基于其 `isEnabled(request, response)` 方法的。

此方法默认情况下返回 `enabled` 属性的值, 这用于一般地启用/禁用上述所有请求。如果要根据请求特定条件启用或禁用筛选, 可以重写 `OncePerRequestFilter` `isEnabled(request, response)` 方法以执行更具体的检查。

### Path-specific Enabling/Disabling

Shiro 的 PathMatchingFilter(`OncePerRequestFilter` 的一个子类) 可以对基于特定、被过滤的路径配置作出反应。这意味着除了传入的请求和响应之外, 还可以根据路径和指定路径配置启用或禁用过滤器。

如果需要能够对匹配路径和路径特定的配置做出反应, 以确定是否启用或禁用了筛选, 而不是重写 `OncePerRequestFilter` `isEnabled(request, response)` 方法, 则可以重写 `PathMatchingFilter` `isEnabled(request, response, path, pathConfig)` 方法。

## Session Management

### Sevlet Container Sessions

在 web 环境，Shiro 的默认会话管理器 `SessionManager` 实现是 `ServletContainerSessionManager`. 这个非常简单的实现委托所有会话管理职责（包括绘画集群，如果 Servlet 支持的话）到运行时 Servlet 容器。它本质上是 Shiro 的会话 API 到 Servlet 容器的桥梁，除此以外别无它用。

使用这个默认的好处是，与现有的 Servlet 容器会话配置（超时，任何特定于容器的集群机制等）一起使用的应用程序将按预期工作。

这个默认的缺点是你被绑定到 Servlet 容器的特定会话行为。例如，如果你想集群会话，但你使用的 Jetty 做测试，Tomcat 用于生产环境，那么你容器特定的配置将不可移植。

#### Servlet Container Session Timeout

如果使用默认的 Servlet 容器支持，你在 web 应用程序的 web.xml 文件中按预期配置会话超时.例如：

```XML
<session-config>
  <!-- web.xml expects the session timeout in minutes: -->
  <session-timeout>30</session-timeout>
</session-config>
```

### 本机会话（Native Sessions）

如果你想你的会话配置设置和集群可以在 Servlet 容器中（例如测试中的 Jetty，但生产环境中的 Tomcat 或 JBoss）移植, 或者你想控制特定会话/集群功能，你可以启用 Shiro 的本地会话管理。

这里的“Native”意味着 Shiro 自己的企业会话管理实现将用于支持所有 `Subject` 和 `HttpServletRequest` 会话并完全绕过 Servlet 容器。但请放心 - Shiro 直接实现了 Servlet 规范的相关部分，所以任何现有 web/http 相关代码都能按预期工作，并从不需要知道 Shiro 透明地管理会话。

#### DefaultWebSessionManager

为你的 web 应用程序启动本机会话管理，你将需要配置一个本地 web-capable 会话管理器来覆盖默认的基于 Servlet 容器的会话管理器。你可以通过在 Shiro 的 `SecurityManager` 上配置一个 `DefaultWebSessionManager` 实例。例如，在 `shiro.ini`:

**shiro.ini native web session management**

```ini
[main]
...
sessionManager = org.apache.shiro.web.session.mgt.DefaultWebSessionManager
# configure properties (like session timeout) here if desired

# Use the configured native session manager:
securityManager.sessionManager = $sessionManager
```

一旦声明，你可以配置带有本地会话选项（如会话超时和集群配置正如在 [Session Management](https://shiro.apache.org/session-management.html) 部分描述的） `DefaultWebSessionManager` 实例。

##### 本机会话超时(Native Session Timeout)

在配置 `DefaultWebSessionManager` 实例后，会话超时如在 Session Management 中那样被配置：[Session Timeout](https://shiro.apache.org/session-management.html#SessionManagement-sessionTimeout)

##### Session Cookie

`DefaultWebSessionManager` 支持两种 web 特定配置属性：

  * `sessionIdCookieEnabled`(a boolean)
  * `sessionIdCookie`, a Cookie instance

> Cookie as a template
>
> `sessionIdCookie` 属性本质上是一个模板 - 你配置的 `Cookie` 实例属性，和这个模板将用于在运行时使用一个适当的会话 ID 值设置实际的 HTTP `Cookie` 头。

###### Session Cookie Configuration

DefaultWebSessionManager 的 `sessionIdCookie` 默认实例是一个 `SimpleCookie`。这个简单的实现允许为你想在一个 HTTP Cookie 上配置所有相关属性的 Javabeans 风格的属性配置。

例如，你可以设置 Cookie 域：

```ini
[main]
...
securityManager.sessionManager.sessionIdCookie.domain = foo.com
```

其他属性参阅 [SimpleCookie JavaDoc](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/servlet/SimpleCookie.html).

Cookie 默认的名字是 `JSESSIONID`，与 Servlet 规范相符合。此外，Shiro 的 Cookie 支持 `HttpOnly` 标记。`sessionIdCookie` 为了额外的安全，通过默认设置 `HttpOnly` 到 `true`。

> 注意
>
> 即使在Servlet 2.4和2.5环境中，Shiro的`Cookie`概念也支持该`HttpOnly`标志（而Servlet API仅在2.6或更高版本中本机支持）。

###### Disabling the Session Cookie

如果你不想使用 session cookies, 你可以通过设置 `sessionIdCookieEnabled` 属性为 false 来禁用它们。例如：

**Disabling native session cookies**

```INI
[main]
...
securityManager.sessionManager.sessionIdCookieEnabled = false
```

## Remember Me Services

如果 `authenticationToken` 实现 `org.apache.shiro.authc.RememberMeAuthenticationToken` 接口，Shiro 将执行“RememberMe”服务。这个接口制定一个方法：

```java
boolean isRemember();
```

如果这个方法返回 `true`, Shiro 将记住跨会话的终端用户的认证身份。

> UsernamePasswordToken and RememberMe
>
> 常用的 `UsernamePasswordToken` 已经实现了 `RememberMeAuthenticationToken` 接口并支持 RememberMe 登录。

### 程序支持（Programmatic Support）

要以编程的方式使用 RememberMe，你可以在一个支持这个配置的类上设置值为 `true`。例如，使用标准的 `UsernamePasswordToken`:

```java
UsernamePasswordToken token = new UsernamePasswordToken(username, password);
token.setRememberMe(true);
SecurityUtils.getSubject().login(token);
```

### 基于表单的登录（Form-based Login）

对于 Web 应用程序，`authc` 过滤器默认是通过 `FormAuthenticationFilter`. 这个支持读 "rememberMe" 布尔值作为一个表单/请求参数。默认情况下，它期待请求参数被命名为“rememberMe”。这有一个支持这个的 Shiro 配置例子：

```ini
[main]
authc.loginUrl = /login.jsp

[urls]

# your login form page here:
login.jsp = authc
```

在你的网页中，有一个复选框名为“rememberMe”:

```html
<form ...>

   Username: <input type="text" name="username"/> <br/>
   Password: <input type="password" name="password"/>
    ...
   <input type="checkbox" name="rememberMe" value="true"/>Remember Me?
   ...
</form>
```

默认情况下，`FormAuthenticationFilter` 将寻找名为 `username`，`password`和`rememberMe`的请求参数. 如果这些不同于在你表单中使用的表单字段名称，则需要在 `FormAuthenticationFilter` 上配置名字。例如，在 `shiro.ini`:

```INI
[main]
...
authc.loginUrl = /whatever.jsp
authc.usernameParam = somethingOtherThanUsername
authc.passwordParam = somethingOtherThanPassword
authc.rememberMeParam = somethingOtherThanRememberMe
...
```

### Cookie configuration

你可以配置如何 rememberMe cookie 功能，通过设置默认 {{RememberMeManager}}s 各种 cookie属性。例如，在 shiro.ini:

```INI
[main]
...

securityManager.rememberMeManager.cookie.name = foo
securityManager.rememberMeManager.cookie.maxAge = blah
...
```

配置属性查看 `[CookieRememberMeManager](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/mgt/CookieRememberMeManager.html)` 和支持 `[SimpleCookie](https://shiro.apache.org/static/current/apidocs/src-html/org/apache/shiro/web/servlet/SimpleCookie.html)` JavaDoc.

### Custom RememberMeManager

应该注意的是，如果默认基于 Cookie 的 `RememberMeManager` 实现没有满足你的要求，你可以插入任何你喜欢的到 `securityManager` 像你会配置任何其他的对象引用：

```INI
[main]
...
rememberMeManager = com.my.impl.RememberMeManager
securityManager.rememberMeManager = $rememberMeManager
```

## JSP / GSP Tag Library

Apache Shiro 提供一个 Subject感知的 JSP/GSP 标签库，允许你基于当前 Subject 的状态控制你的 JSP, JSTL或 GSP 页面输出。这对基于当前查看网页状态的用户身份和授权状态对视图个性化非常有用。

### Tag Library Configuration

标签库描述符（TLD）文件捆绑在 `shiro-web.jar` 中 `META-INF/shiro.tld` 的文件。要使用任何标记，请将以下行添加到 JSP 页面的顶部（或者定义页面指令的任何位置）：

```html
<%@ taglib prefix="shiro" uri="http://shiro.apache.org/tags" %>
```

我们已经使用 `Shiro` 前缀来表示 Shiro 标签库命名空间，但你可以分配你喜欢的任何名字。现在，我们将介绍每个标签并展示它怎么使用来渲染页面。

### The guest tag

`guest` 标签只有当前 Subject 被视为“客人”的情况下，它才会显示其包装的内容。客人是任何没有身份认证的 `Subject`。也就是说，我们不知道用户是谁，因为它们没有登录并且以前网站访问没有被记住（来自 RememberMe 服务）。

例如：

```html
<shiro:guest>
    Hi there!  Please <a href="login.jsp">Login</a> or <a href="signup.jsp">Signup</a> today!
</shiro:guest>
```

guest 标签是 user 标签相反的逻辑。

### The user tag

`user` 只有在当前`Subject`被视为“用户”的情况下，标签才会显示其包装的内容。在这种情况下，“用户”被定义为具有已知身份的`Subject`，无论是来自成功的身份验证还是来自“RememberMe”服务。请注意，这个标签在语义上与 [authenticated](https://shiro.apache.org/web.html#Web-authenticatedtag) 标签不同，它比这个标签更具限制性。

例如：

```html
<shiro:user>
    Welcome back John!  Not John? Click <a href="login.jsp">here<a> to login.
</shiro:user>
```

 user 标签是 guest 标签相反的逻辑。

### The authenticated tag

仅当当前用户在它们当前会话期间成功身份验证，才显示主体内容。它比“用户”标签更具限制性。它在逻辑上与“notAuthenticated”标签相反。

仅当当前 `Subject` 在它们当前会话期间成功身份验证，`authenticated` 标签将显示它包裹的内容。这是比 user 更有限制性的标签，用于保证敏感工作流中的身份。

例如：

```html
<shiro:authenticated>
    <a href="updateAccount.jsp">Update your contact information</a>.
</shiro:authenticated>
```

`authenticated` 标签逻辑相反的是 `notAuthenticated` 标签。

### The notAuthenticated tag

如果当前 `Subject` 在当前会话期间还没有成功地身份认证，`notAuthenticated` 标签将显示它包裹的内容。

例如：

```html
<shiro:notAuthenticated>
    Please <a href="login.jsp">login</a> in order to update your credit card information.
</shiro:notAuthenticated>
```

`notAuthenticated` 标签逻辑相反的是 `authenticated` 标签。

### The principal tag

`principal` 标签将输出 Subject 的 `principal`(识别属性) 或主要的属性。

没有任何标签属性，这个标签将渲染 principal 的 `toString()` 值。例如（假设 principal 是一个字符串用户名）：

```html
Hello, <shiro:principal/>, how are you today?
```

这个（大部分）等价于以下内容：

```html
Hello, <%= SecurityUtils.getSubject().getPrincipal().toString() %>, how are you today?
```

#### Typed principal

`principal` 标签假设默认情况下，principal 打印的是 `subject.getPrincipal()` 的值。但是，如果你想打印出一个不是主要 principal 的值，而是 Subject [principal collection](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/subject/Subject.html#getPrincipals--)中另外的一个，你可以通过类型获取 principal 并打印出值。

例如，打印 Subject 用户ID(不是用户名)，假定在 principal collection 中的 ID:

```html
User ID: <principal type="java.lang.Integer"/>
```

这个（大部分）等价于以下内容：

```html
User ID: <%= SecurityUtils.getSubject().getPrincipals().oneByType(Integer.class).toString() %>
```

#### Principal property

但是，如果 principal(上面的默认主要 principal 或 Typed principal) 是一个复杂的对象并不是一个简单字符串，而你想引用这个 principal 上的一个属性呢？你可以使用 `property` 属性来指示要读取的属性的名称（必须可通过 JavaBeans 兼容的 getter 方法访问）。例如（假设主要的 principal 是一个 User 对象）：

```html
Hello, <shiro:principal property="firstName"/>, how are you today?
```

这个（大部分）等价于以下内容：

```html
Hello, <%= SecurityUtils.getSubject().getPrincipal().getFirstName().toString() %>, how are you today?
```

或者，结合 type 属性：

```html
Hello, <shiro:principal type="com.foo.User" property="firstName"/>, how are you today?
```

这在很大程度上相当于以下内容：

```html
Hello, <%= SecurityUtils.getSubject().getPrincipals().oneByType(com.foo.User.class).getFirstName().toString() %>, how are you today?
```

### The hasRole tag

仅当当前 `Subject` 被分配到指定角色，`hasRole` 标签将显示它包裹的内容。

例如：

```html
<shiro:hasRole name="administrator">
    <a href="admin.jsp">Administer the system</a>
</shiro:hasRole>
```

`hasRole` 标签逻辑相反的是 `lacksRole` 标签。

### The lacksRole tag

仅当当前 `Subject` **未被** 分配到指定角色，`hasRole` 标签将显示它包裹的内容。

例如：

```html
<shiro:lacksRole name="administrator">
    Sorry, you are not allowed to administer the system.
</shiro:lacksRole>
```

`lacksRole` 标签逻辑相反的是 `hasRole` 标签。

### The hasAnyRole tag

如果当前 `Subject` 被分配从一个逗号分隔的角色名列表中任意指定的角色，`hasAnyRole` 标签将显示它包裹的内容。

例如：

```html
<shiro:hasAnyRoles name="developer, project manager, administrator">
    You are either a developer, project manager, or administrator.
</shiro:hasAnyRoles>
```

`hasAnyRole` 标签目前没有一个逻辑相反的标签。

### The hasPermission tag

仅当当前 `Subject` 有（暗示）指定的权限，`hasPermission` 标签将显示它包裹的内容。也就是说，用户具有指定的能力。

例如：

```html
<shiro:hasPermission name="user:create">
    <a href="createUser.jsp">Create a new User</a>
</shiro:hasPermission>
```

`hasPermission` 标签逻辑相反的是 `lacksPermission` 标签。

### The lacksPermission tag

仅当当前 `Subject` **没有** 指定的权限，`hasPermission` 标签将显示它包裹的内容。也就是说，用户 **不具有** 指定的能力。

```html
<shiro:lacksPermission name="user:delete">
    Sorry, you are not allowed to delete user accounts.
</shiro:lacksPermission>
```

`lacksPermission` 标签逻辑相反的是 `hasPermission` 标签。
