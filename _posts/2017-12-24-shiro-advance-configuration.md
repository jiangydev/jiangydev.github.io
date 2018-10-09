---
layout:     post
title:      "Shiro 配置"
subtitle:   ""
date:       2017-12-24
author:     "jiangydev"
header-img: "img/post-bg-shiro.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Shiro
---

[TOC]

# Apache Shiro Configuration

Shiro 被设计在任何环境中工作，从简单的命令行应用程序到最大的企业集群应用程序。由于这种环境的多样性, 有许多适合配置的配置机制。本节涉及仅由 Shiro 核心支持的配置机制。

> Many Configuration Options
>
> Shiro 的 SecurityManager 实现和所有支持组件都是兼容 JavaBeans 的。这允许 Shiro 实际上以任何配置格式被配置，例如常见的 Java, XML(Spring, JBoss, Guice, etc), YAML, JSON, Groovy Builder markup 等。

## Programmatic Configuration

创建 SecurityManager 并使其可用于应用程序的绝对最简单的方法是创建一个 `org.apache.shiro.mgt.DefaultSecurityManager`, 并在代码中对其进行连线。例如:

```Java
Realm realm = // 实例化或获取一个 Realm 实例. 我们将以后讨论 Realms.
SecurityManager securityManager = new DefaultSecurityManager(realm);

//使 SecurityManager 实例通过静态 memory 对整个应用程序可获得:
SecurityUtils.setSecurityManager(securityManager);
```

Surprisingly, 仅在 3 行代码之后，你现在有一个适合许多应用程序的功能性的 Shiro 环境。

### SecurityManager Object Graph

正如在 [架构](https://shiro.apache.org/architecture.html) 章节讨论的，Shiro 的 SecurityManager 实现本质上是一个嵌套特定安全组件的模块化对象图表。因为它们也兼容 JavaBeans，你可以调用任何嵌套组件的 getter 和 setter 方法来配置 SecurityManager 和它的内部对象图表。

例如，你想配置 SecurityManager 实例来使用一个自定义 SessionDAO 自定义 Session Management, 你可以直接用嵌套 SessionManager 的 setSessionDAO 方法设置 SessionDAO ：

```java
DefaultSecurityManager securityManager = new DefaultSecurityManager(realm);

SessionDAO sessionDAO = new CustomSessionDAO();

((DefaultSessionManager)securityManager.getSessionManager()).setSessionDAO(sessionDAO);
```

使用直接方法调用, 你可以配置 SecurityManager 的对象图的任何部分。

但是, 就像编程定制一样简单, 它并不代表大多数真实世界应用程序的理想配置。编程配置可能不适合您的应用程序有几个原因:

  * 它需要您了解并实例化一个直接实现。如果你不需要了解具体的实现以及在哪里可以找到它们, 那就更好了。

  * 由于 Java 的类型安全性质, 您需要将通过 get* 方法获得的对象强制转换为特定的实现。这么多的转换是丑陋的, 冗长的, 紧密结合到实现类。

  * SecurityUtils.setSecurityManager 方法调用使实例化的 SecurityManager 实例成为 VM 静态单例, 这在许多应用程序中很好, 但如果在同一 JVM 上运行多个启用 Shiro 的应用程序, 则会导致问题。如果实例是单个应用程序, 而不是静态内存引用, 则可能会更好。

  * 它要求您在每次要进行一个 Shiro 配置更改时重新编译应用程序。

然而, 即使有了这些警告, 直接编程操作方法仍然可以在内存受限的环境 (如智能电话应用程序) 中很有价值。如果应用程序没有在内存受限的环境中运行, 您将发现基于文本的配置更易于使用和读取。

## INI Configuration

大多数应用程序都是从基于文本的配置中受益的，这些配置可以独立于源代码进行修改，甚至使那些不熟悉 Shiro APIs 的人更容易理解。

为了确保基于文本的通用配置机制能在具有最小的第三方依赖关系的所有环境中工作，Shiro 支持 INI 格式来构建 SecurityManager 对象图表及其支持的组件。INI 易于阅读、配置，而且安装简单，适用于大多数应用。

### Creating a SecurityManager from INI

这有两个如何基于 INI 配置构建一个 SecurityManager 的例子。

#### SecurityManager from an INI resource

我们可以从一个 INI 资源路径创建 SecurityManager 实例。当前缀带有 file:, classpath: 或 url: 时，资源分别从文件系统、classpath或 URLs 获取。这个例子使用一个 Factory，从 classpath 根下摄取 shiro.ini 文件，并返回 SecurityManager 实例：

```java
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.util.Factory;
import org.apache.shiro.mgt.SecurityManager;
import org.apache.shiro.config.IniSecurityManagerFactory;

...

Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro.ini");
SecurityManager securityManager = factory.getInstance();
SecurityUtils.setSecurityManager(securityManager);

```

#### SecurityManager from an INI instance

如果期望通过 `org.apache.shiro.config.Ini` 类，INI 配置也能被编程结构化。Ini 类与 JDK 中 `java.util.Properties` 类功能相似，但 但另外还支持按节名进行分段。例如：

```java
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.util.Factory;
import org.apache.shiro.mgt.SecurityManager;
import org.apache.shiro.config.Ini;
import org.apache.shiro.config.IniSecurityManagerFactory;

...

Ini ini = new Ini();
//populate the Ini instance as necessary
...
Factory<SecurityManager> factory = new IniSecurityManagerFactory(ini);
SecurityManager securityManager = factory.getInstance();
SecurityUtils.setSecurityManager(securityManager);
```

现在，我们知道如何从 INI 配置中构造一个 SecurityManager，让我们看一下怎样准确定义一个 Shiro INI 配置。


### INI Sections

INI 是由唯一命名部分组织的，由 键/值对 组成的一个基本文本配置。键仅在每个部分中是唯一的, 而不是整个配置 (与 JDK [Properties](http://java.sun.com/javase/6/docs/api/java/util/Properties.html) 不同)。但是, 每个部分可以像单个属性定义一样被查看。

注释行可以开始用一个 Octothorpe (# - 又名 "hash", "pound" 或 "number" 符号) 或分号 (";").

这里是一个 section 的例子，用来理解:

```ini
# =======================
# Shiro INI configuration
# =======================

[main]
# Objects and their properties are defined here, Such as the securityManager, Realms and anything else needed to build the SecurityManager
# 对象和他们的属性在这被定义，例如 SecurityManager, Realms 和任何需要被用来构建 SecurityManager 的。

[users]
# The 'users' section is for simple deployments
# when you only need a small number of statically-defined
# set of User accounts.

[roles]
# The 'roles' section is for simple deployments
# when you only need a small number of statically-defined
# roles.

[urls]
# The 'urls' section is used for url-based security
# in web applications. We'll discuss this section in the
# Web documentation
```

#### [mian]

`[main]` 部分是你配置应用程序 SecurityManager 实例和它的依赖（例如，Realms）的地方。

我们仅可以使用 名/值对，用 INI 配置对象实例（像 SecurityManager 或任何它的依赖）听起来像是一件困难的事。但通过一点规则和对象图表的理解，你会发现你能做很多。Shiro 使用这些假设来实现一个简单而又相当简洁的配置机制。

我们经常把这种方法称为 "poor man's" 依赖注入，虽然没有充分发展的（full-blown） Spring/Guice/JBoss XML 文件那样强大, 但你会发现它在没有太多复杂性的情况下完成了很多工作。当然, 其他的配置机制也可用, 但他们不需要使用 Shiro。

只是为了激起你的食欲, 这里是一个有效的 `[main]` 配置的例子。我们将在下面详细介绍它, 但你可能会发现, 仅凭直觉, 您就已经完全理解了这一点:

```ini
[main]
sha256Matcher = org.apache.shiro.authc.credential.Sha256CredentialsMatcher

myRealm = com.company.security.shiro.DatabaseRealm
myRealm.connectionTimeout = 30000
myRealm.username = jsmith
myRealm.password = secret
myRealm.credentialsMatcher = $sha256Matcher

securityManager.sessionManager.globalSessionTimeout = 1800000
```

##### Defining an object

考虑一下接下来的 `[main]` 片段：

```ini
[main]
myRealm = com.company.shiro.realm.MyRealm
...
```

这行实例化一个 `com.company.shiro.realm.MyRealm` 类型的新对象实例，并使该对象在 myRealm 名称下可用以进一步引用和配置。

如果对象实例化实现了 `org.apache.shiro.util.Nameable` 接口, 则将在具有 name 值的对象 (本例中为 myRealm) 上调用 Nameable.setName 方法。

##### Setting object properties

###### Primitive Values

简单的原始属性仅通过使用相等符号分配：

```ini
myRealm.connectionTimeout = 30000
myRealm.username = jsmith
```

这些配置行转换成方法调用为：

```java
myRealm.setConnectionTimeout(30000);
myRealm.setUsername("jsmith");
```

它假定所有对象都是与 Java Beans 对象兼容的 POJOs.

当设置这些属性时，Shiro 默认使用 Apache Commons BeanUtils 来做所有繁重的事。因此, 尽管 INI 值是文本, 但 BeanUtils 知道如何将字符串值转换为正确的原始类型, 然后调用相应的 JavaBeans setter 方法。

###### Reference Values

如果你需要设置的值不是原始的，是另一个对象呢？Well, 你可以使用一个 `$` 符号来引用之前定义的实例，例如：

```ini
sha256Matcher = org.apache.shiro.authc.credential.Sha256CredentialsMatcher
...
myRealm.credentialsMatcher = $sha256Matcher
```

这只是定位由名称 sha256Matcher 定义的对象，然后使用 BeanUtils 在 Realm 实例上设置该对象（通过调用 `myRealm.setCredentialsMatcher(sha256Matcher)` 方法）。

###### Nested properties

在 INI 行的等号左边使用，你可以遍历一个对象图以获取你想设置的最终 对象/属性。例如，配置行如下：

```ini
securityManager.sessionManager.globalSessionTimeout = 1800000
```

被 BeanUtils 转换成接下来的逻辑：

```java
securityManager.getSessionManager().setGlobalSessionTimeout(1800000);
```

图表遍历尽可能必要的深：`object.property1.property2....propertyN.value = blah`

> BeanUtils Property Support
>
> 任何属性分配操作由 BeanUtils.setProperty 方法支持，工作在 Shiro 的 [main] 部分，包含 set/list/map 元素分配。详细信息可以阅读 [Apache Commons BeanUtils Website](http://commons.apache.org/proper/commons-beanutils/).

###### Byte Array Values

因为原字节数组不能天然的在文本格式中特定，我们必须使用字节数组的文本编码。值可以特定为一个 Base64 编码字符串（默认）或者一个 Hex 编码字符串。默认是 Base64，因为 Base64 编码要求更少的实际文本来呈现值 - 它有一个更大的编码字母表，意味着你的 tokens 更短（一个更漂亮的文本配置）。另外，Hex 编码不区分大小写。

```ini
# 'cipherKey' 属性是一个字节数组. 默认的, 所有字节数组属性的文本值期待是 Base64 编码:

securityManager.rememberMeManager.cipherKey = kPH+bIxk5D2deZiIxcaaaA==
...
```

但是，如果你更喜欢用 Hex 编码，你必须使用 `0x` 前缀：

```ini
securityManager.rememberMeManager.cipherKey = 0x3707344A4093822299F31D008
```

###### Collection properties

Lists, Sets 和 Maps 能被设置成任何其他的属性 - 直接或作为嵌套属性。对 sets 和 lists 而言，只要指定一个以逗号分隔的值或对象应用集。某些 SessionListeners 如下：

```ini
sessionListener1 = com.company.my.SessionListenerImplementation
...
sessionListener2 = com.company.my.other.SessionListenerImplementation
...
securityManager.sessionManager.sessionListeners = $sessionListener1, $sessionListener2
```

对 Maps 而言，你指定一个逗号分隔键值对列表，每一个键值对被一个分号分隔：

```ini
object1 = com.company.some.Class
object2 = com.company.another.Class
...
anObject = some.class.with.a.Map.property

anObject.mapProperty = key1:$object1, key2:$object2
```

在上面的例子中，对象通过 `$object1` 应用，在 map 中是在字符串键 `key1` 下的，例如：`map.get("key1")` 返回 `$object1`。你也可以使用其他的对象作为键：

```ini
anObject.map = $objectKey1:$objectValue1, $objectKey2:$objectValue2
```

##### Considerations

###### Order Matters

上面的 INI 格式和约定非常方便且易于理解，但它不如其他 文本/基于XML 的配置机制强大。在使用上述机制时，最重要的就是了解顺序的重要性!

> Be Careful
>
> 每个对象实例化和每个值分配按它们在 `[main]` 部分出现的顺序执行。这些行为最终转换为 JavaBeans getter/setter 方法调用，因此这些方法按相同的顺序调用！在编写配置时要牢记这一点。

###### Overriding Instances

任何对象能通过一个之后再配置中新定义的实例重写。所以，第二个 myRealm 定义将覆盖第一个的例子如下：

```ini
myRealm = com.company.security.MyRealm
...
myRealm = com.company.security.DatabaseRealm
```

这个结果是 myRealm 为 `com.company.security.DatabaseRealm` 实例，前面的实例不再被使用（垃圾回收）。

###### Default SecurityManager

你可能意识到上面完成的例子，SecurityManager 实例没有定义，并且我们跳过只设置一个嵌套的属性：

```ini
myRealm = ...

securityManager.sessionManager.globalSessionTimeout = 1800000
```

这是因为 securityManager 实例是特殊的一个，它已经为你初始化好并且准备使用，所以你不需要知道指定 SecurityManager 实现类来实例化。

当然，如果你实在想指定你自己的实现，你只要在上面的 "Overriding Instances" 部分定义你的实现：

```ini
...
securityManager = com.company.security.shiro.MyCustomSecurityManager
...
```

这很少需要 - Shiro 的 SecurityManager 实现非常自定义化，并且有必要就可以被配置。你可能想问你自己（或用户列表）是否真的需要。

#### [users]

`[users]` 部分允许你定义一个用户账号的静态集合。这在一个非常小数量用户账号或用户账号不需要在运行时动态创建时，非常有用。例子如下：

```ini
[users]
admin = secret
lonestarr = vespa, goodguy, schwartz
darkhelmet = ludicrousspeed, badguy, schwartz
```

> Automatic IniRealm
>
> 只定义非空的 [users] 和 [roles] 部分将自动触发创建一个 `org.apache.shiro.realm.text.IniRealm` 实例并使它在 [main] 部分以名字 `iniRealm` 有用。你可以配置它，像任何其它上面描述的对象一样。

##### Line Format

在 `[users]` 部分，每一行必须遵循接下来的格式：

```ini
username = password, roleName1, roleName2, …, roleNameN
```

  * 等号左边的值是 用户名；
  * 等号右边第一个值是用户的密码，一个密码是必须的；
  * 在密码后，任何逗号分隔的值是分配给用户的角色名，角色名可选。

##### Encrypting Passwords

如果你不希望 `[users]` 部分的密码是纯文本格式，则可以使用你喜欢的哈希算法（MD5，Sha1，Sha256等）对它们加密，并使用结果字符串作为密码值。默认情况下，密码字符串应为 Hex 编码，但可以配置 Base64 编码。

> Easy Secure Passwords
>
> 为节约时间并使用最佳测试，你可能想要使用 Shiro 的 [Command Line Hasher](https://shiro.apache.org/command-line-hasher.html), 它将哈希密码以及任何其他类型的资源。它特别便于对 INI [users] 密码加密。

一旦你指定了哈希文本密码的值，你不得不告诉 Shiro 这些是加密的。通过在 `[main]` 部分中配置隐式创建的 iniRealm, 可以使用与你指定的哈希算法相对应的适当的 CredentialsMatcher 实现:

```ini
[main]
...
sha256Matcher = org.apache.shiro.authc.credential.Sha256CredentialsMatcher
...
iniRealm.credentialsMatcher = $sha256Matcher
...

[users]
# user1 = sha256-hashed-hex-encoded password, role1, role2, ...
user1 = 2bb80d537b1da3e38bd30361aa855686bde0eacd7162fef6a25fe97bf527a25b, role1, role2, ...
```

你可以像任何其他对象一样在 `CredentialsMatcher` 上配置任何属性，以反映哈希策略。例如，指定是否使用 salting，或要执行多少哈希迭代。请参阅 `org.apache.shiro.authc.credential.HashedCredentialsMatcher` JavaDoc 以更好的理解哈希策略，以及它们是否对你有用。

例如，如果你的用户密码字符串是 Base64 编码的，而不是默认的 Hex，你可以指定：

```ini
[main]
...
# true = hex, false = base64:
sha256Matcher.storedCredentialsHexEncoded = false
```

#### [roles]

`[roles]` 部分允许你关联权限和定义在 `[users]` 部分的角色。相同的，在小数量角色，角色在运行时不需要动态创建的环境下，是有用的。这有一个例子：

```ini
[roles]
# 'admin' role has all permissions, indicated by the wildcard '*'
admin = *
# The 'schwartz' role can do anything (*) with any lightsaber:
schwartz = lightsaber:*
# The 'goodguy' role is allowed to 'drive' (action) the winnebago (type) with
# license plate 'eagle5' (instance specific id)
goodguy = winnebago:drive:eagle5
```

##### Line Format

在 `[roles]` 部分的每一行必须定义一个角色到权限的键值映射，接下来的格式：

```
rolename = permissionDefinition1, permissionDefinition2, …, permissionDefinitionN
```

权限定义是一个随意的字符串，但大多数人将想用字符串遵循 [`org.apache.shiro.authz.permission.WildcardPermission`](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/authz/permission/WildcardPermission.html) 格式以易于使用和灵活性。参阅 Permissions 文档获取更多权限的信息。

> Internal commas
>
> 记住，如果一个独立的的权限定义需要内部逗号分隔（例如，`printer:5thFloor:print,info`），你将需要用双引号包围着这个定义，避免解析出错：`"printer:5thFloor:print,info"`.


> Roles without Permissions
>
> 如果你有角色不需要权限关联，你不需要在 [roles] 部分列出它们。如果它也不存在，只要在 [users] 部分定义角色名就足够创建角色。

#### [urls]

这部分在 [web](https://shiro.apache.org/web.html) 章节。
