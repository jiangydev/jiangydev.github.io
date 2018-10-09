---
layout:     post
title:      "Shiro 命令行哈希器"
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

# Command Line Hasher

Shiro 1.2 及之后的版本提供一个命令行工程，能够哈希字符串和资源（文件、URLs、classpath 入口）几乎任意类型。为了使用它，你必须安装了 Java 虚拟机，并且 `java` 命令能在你的 `$PATH` 环境变量中访问。

## Usage

确保你能访问 `shiro-tools-hasher-_version_-cli.jar` 文件。你可以在buildroot /tools/hasher/target 目录中生成的源代码中找到它，或通过 Maven 下载。

一旦你能访问到 jar, 你可以运行下面的命令行：

```cmd
$ java -jar shiro-tools-hasher-X.X.X-cli.jar
```

这会为标准的（MD5, SHA1）和更多复杂密码哈希的场景打印出所有可获得的选项。

## Common Scenarios

请阅读上面命令行打印出的指令。它会提供一个详尽的指令列表，按你的需求帮助你使用 Hasher。但是，为方便起见，下面我们也提供一些快速使用/场景参考。

## shiro.ini User Passwords

最好使用户密码在 shiro.ini 的 [users] 部分安全。为了增加一个新的用户帐户行，使用上面的命令行带 `-p`(或 `--password`) 选项：

```
$ java -jar shiro-tools-hasher-X.X.X-cli.jar -p
```

它将会让你输入密码并确认：

```
Password to hash:
Password to hash (confirm):
```

当这个命令行执行，它会打印出  securely-salted-iterated-and-hashed 密码，例如：

```
$shiro1$SHA-256$500000$eWpVX2tGX7WCP2J+jMCNqw==$it/NRclMOHrfOvhAEFZ0mxIZRdbcfqIBdwdwdDXW2dM=
```

拿着这个值并在用户定义的行(跟着任何可选的角色)作为密码替换：

```ini
[users]
...
user1 = $shiro1$SHA-256$500000$eWpVX2tGX7WCP2J+jMCNqw==$it/NRclMOHrfOvhAEFZ0mxIZRdbcfqIBdwdwdDXW2dM=
```

你还需要确保隐含的 iniRealm 使用 CredentialsMatcher，知道如何执行安全哈希密码比较。所以在 [main] 部分配置它：

```ini
[main]
...
passwordMatcher = org.apache.shiro.authc.credential.PasswordMatcher
iniRealm.credentialsMatcher = $passwordMatcher
...
```

## MD5 checksum

尽管你可以用 JVM 上支持的任何算法执行任何哈希，默认的哈希算法是 MD5，对文件校验很常见。只要使用 `-r`(或 `--resource`) 选项指明接下来的值是资源位置：

```
$ java -jar shiro-tools-hasher-X.X.X-cli.jar -r RESOURCE_PATH
```

默认的 `RESOURCE_PATH` 期望是一个文件路径，但你可能通过 `classpath:` 或 `url:` 前缀指定 classpath 或 URL 资源. 例子如下：

```
<command> -r fileInCurrentDirectory.txt
<command> -r ../../relativePathFile.xml
<command> -r ~/documents/myfile.pdf
<command> -r /usr/local/logs/absolutePathFile.log
<command> -r url:http://foo.com/page.html
<command> -r classpath:/WEB-INF/lib/something.jar
```
