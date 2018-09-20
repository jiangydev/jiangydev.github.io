---
layout:     post
title:      "Shiro 中的权限"
subtitle:   ""
date:       2017-12-23
author:     "jiangydev"
header-img: "img/post-bg-shiro.jpg"
tags:
    - Shiro
---

## Understanding Permissions in Apache Shiro

Shiro 将权限定义为一个定义了显式行为或操作的语句。它是应用程序中原始功能的声明，仅此而已。权限是安全策略中最低级别的构造，它们明确定义了应用程序可以执行什么。

它们根本不描述“谁”有能力执行操作。

例子如下：

  * 打开一个文件
  * 看 "/user/list" 网页
  * 打印文档
  * 删除用户

定义“谁”（user）被允许做某事（permissions）在某些方式上是一次给用户分配权限的练习。这总是由应用程序的数据模型完成，并在不同的应用程序中可能相差很大。

例如，可以将权限分组到角色中，该角色可以与一个或多个用户对象关联。或者某些应用程序可以有一组用户，并且可以为一个组分配角色，这可传递关联意味着组中的所有用户都被隐式授予角色中的权限。

在如何授予用户权限方面有许多变体，应用程序根据程序的要求确定如何对其建模。

### Wildcard Permissions(通配符权限)

上面的例子都是有效的权限语句，但是解释这些自然语言字符串并确定是否允许用户执行该行为将是非常困难的。

因此，为了使 easy-to-process 仍然可读的权限语句，Shiro 提供了强有力、直观的权限语法，称之为 Wildcard Permissions.

#### 简单用法

假设你希望保护对公司打印机的访问，以便某些人可以打印到特定的打印机，而其他用户可以查询当前队列中的作业。

一个极其简单的方式是授予用户 "queryPrinter" 权限。然后你能检查确保是否用户有查询打印机的权限，通过调用

```java
subject.isPermitted("queryPrinter")
```

这（大多数）等同于 `subject.isPermitted( new WildcardPermission("queryPrinter") )`，以后会更多。

这个简单权限字符串对简单应用程序可以，但它要求你有权限像 “printPrinter”, “queryPrinter”, “managePrinter”等。你也可以使用通配符(giving this permission construct its name)授予用户 `*` 权限，这意味着他在整个应用程序中拥有所有的权限。

但使用这个方法就没办法只说一个用户有“所有打印权限”。因此，通配符权限支持多级别权限。

#### 多个部分（Multiple Parts）

通配符权限支持多级别或部分的概念。例如，你能通过授予用户权限重建之前简单例子：

```
printer:query
```

在这个例子中，冒号是一个特殊字符被用来分隔权限字符串中的下一个部分。

例子中，第一部分是正在操作的域（printer），第二部分是正在执行的操作（query）。上面例子的另外一个被改成：

```
printer:print
printer:manage
```

没有限制使用 parts 的数量，所以在你的应用程序中，它取决于你想使用的。

##### 多值（Multiple Values）

每一部分能包含多个值。所以不要授权用户“printer:print”和“printer:query”权限，你可以简单的授予他们一个：

```
printer:print,query
```

这样他们有 print 和 query 的能力。既然他们被授予这些操作，你可以检查是否用户有能力查询打印机，通过以下语句：(将会返回 true)

```java
subject.isPermitted("printer:query")
```

##### 所有值（All Values）

如果你想在一个特定部分授予用户所有值，怎么办？同样，基于通配符，我们可以做到这一点，这样比手动列出所有值更方便。如果打印机域有3个可能的操作：

```
printer:query,print,manage
```

简单的变为：

```
printer:*
```

然后，任何权限检查将会返回 true。在这种方式下使用通配符比明确列出操作更好，如果你以后增加一个新的操作到应用程序，不需要更新那部分使用通配符的权限。

最后，还可以在通配符权限字符串的任何部分中使用通配符。例如, 如果要在所有域 (而不仅仅是打印机) 上授予用户 '查看' 操作, 可以授予此项:

```
*:view
```

然后，任何权限检查 (如：foo:view) 将返回 true.

#### 实例级访问控制（Instance-Level Access Control）

通配符权限的另一个常见用法是对实例级访问控制列表进行建模。在这种情况下，你使用三部分-第一部分是域，第二部分是操作，第三部分是正在执行的实例。

例子：

```
printer:query:lp7200
printer:print:epsoncolor
```

第一行定义了使用 ID lp7200 查询 printer 的行为。第二行定义了使用 ID epsoncolor 打印到打印机的行为。如果想用户授予这些权限，他们可以对特定的实例执行特定的行为。然后，你可以在代码中执行检查：

```
if ( SecurityUtils.getSubject().isPermitted("printer:query:lp7200") {
    // Return the current jobs on printer lp7200 }
}
```

这是一个非常有力的方式表达权限，但同样，不得不为所有打印机定义多个实例ID，不能很好扩展，尤其是在新打印机添加到系统时。你可以改用通配符：

```
printer:print:*
```

可扩展，因为它也覆盖了任何新的打印机。你甚至可以允许在所有的打印机上所有的操作：

```
printer:*:*
```

或在单个打印机上的所有操作：

```
printer:*:lp7200
```

或甚至特定操作：

```
printer:query,print:lp7200
```

##### 缺少部分

有关权限分配的最后一件事要注意: 缺少的部分意味着用户可以访问与该部件对应的所有值。话句话说，`printer:print` 等价于 `printer:print:*`, `printer` 等价于 `printer:*:*`。但是，`printer:lp7200` 不等价于 `printer:*:lp7200`.

### 权限检查

尽管权限分配使用通配符结构相当为了方便和可扩展，但运行时权限检查应该始终基于可能的最具体的权限字符串的。

例如，如果用户有一个 UI 并且想打印文档到 lp7200 打印机，你应该检查用户是否允许这么做：

```java
if ( SecurityUtils.getSubject().isPermitted("printer:print:lp7200") ) {
    //print the document to the lp7200 printer }
}
```

这个检查非常具体而明确的表明当时用户尝试要做的。

但接下来的，对于运行时检查更不理想了：

```java
if ( SecurityUtils.getSubject().isPermitted("printer:print") ) {
    //print the document }
}
```

为什么？因为第二个例子说“你必须有能力打印到任何打印机，以便执行以下代码块”。因此，这是一次不正确的检查。如果当前用户没有能力打印到任何打印机，但他们确实有能力打印到 lp7200 和 epsoncolor。然后，上面第二个例子将永远不会允许他打印到 lp7200，即使他们已经被授予该功能。

因此，经验法则是在执行权限检查时，使用最具体的权限字符串。当然，如果允许用户打印到任何打印机（可疑，但可能），那么上面第二块代码可能是应用程序中其他位置的有效检查。你的应用程序将决定什么是有意义的检查，但总的来说，越具体越好。

### 隐含，不相等（Implication, not Equality）

为什么运行时权限检查应该尽可能具体，但权限分配可以更通用一些呢？这是因为权限检查由隐含逻辑（而不是相等检查）进行评估。

就是说，如果用户分配到 `user:*` 权限，这暗示用户可以执行 `user:view` 操作。字符串 `user:*` 清晰地不等价于 `user:view`，但前者意味着后者。`user:*` 描述了由 `user:view` 定义的功能的超集。

为了支持隐含规则，所有权限被转化为实现 `org.apache.shiro.authz.Permission` 接口的对象实例。这是为了使隐含逻辑可以在运行时执行，并且隐含逻辑通常比简单的字符串相等检查更复杂。本文档中描述的所有通配符行为实际上通过 `org.apache.shiro.authz.permission.WildcardPermission` 类实现成为可能的。下面是一些更多的通配符权限字符串, 它们通过隐含表示访问:

  `user:*` 隐含删除用户：`user:delete`;

  `user:*:12345` 隐含更新 ID 12345 的用户账号：`user:update:12345`

  `printer` 隐含打印到任何打印机: `printer:print`

### 性能注意事项（Performance Considerations）

权限检查比简单相等比较更复杂，所以运行时隐含逻辑必须为每个分配的权限执行。当使用权限字符串像上面展示的一个，你正在使用 Shiro 默认的 `WildcardPermission` 执行必要的隐含逻辑。

Shiro 对 Realm 实现 的默认行为，为每个权限检查（例如调用 `subject.isPermitted`），分配给该用户所有权限（在他们的组、角色，或直接分配给他们）需要对隐含的单独检查。Shiro "short circuits" 这个过程通过在第一次成功的检查后立即返回来提高性能，但这不是一枚银子弹。

当使用适当的 `CacheManager` 时，在内存中缓存用户、角色和权限，这通常很快，这是 Shiro 对 Realm 实现 的支持。只需要知道，使用此默认行为，当分配给用户或他们的角色、组的数量增加时，执行检查的时间必然会增加。

如果 Realm 实现器 有更有效的方式检查权限和执行这个隐含逻辑，尤其如果基于应用程序的数据模型，他们应该实现 Realm isPermitted* 方法实现类作为一部分。默认的 Realm/WildcardPermission 支持存在覆盖 80-90% 的使用案例，但对有大量权限存储和在运行时检查的应用程序来说，可能不是最好的解决方案。
