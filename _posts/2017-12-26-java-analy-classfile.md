---
layout:     post
title:      "Java Class 文件内容分析"
subtitle:   ""
date:       2017-12-26
author:     "jiangydev"
header-img: "img/post-bg-java.jpg"
tags:
    - Java
---

# Java Class 文件内容解析

参考文献: [1. JVM系列文章(三):Class文件内容解析,  yxwkaifa](https://www.cnblogs.com/yxwkf/p/5222589.html)

一个Java类文件是一个（使用 .class 文件扩展名）Java字节码文件，可以在Java虚拟机（JVM）上执行。Java类文件是由 Java 编译器从包含 Java 类的 Java 编程语言源文件（.java文件）生成的。如果一个源文件有多个类，每个类都被编译成一个单独的类文件。

JVM可用于多种平台，并且在一个平台上编译的类文件将在另一个平台的JVM上执行。这使得Java应用程序与平台无关。[摘自 Wikipedia](https://en.wikipedia.org/wiki/Java_class_file)

因此，无论何种语言，编译成 JVM 识别的 class 文件即可执行。class 文件以另一种方式重新描述了我们在源文件中要表达的意思，相当于一个接口，学习 class 文件对深入理解 Java 语言和 JVM 有很大帮助。

## 格式概述

Class文件是一组以8位字节为基础单位的二进制流。各个数据项严格按顺序排列，不论什么分隔符。

Class文件格式采用一种类似于C语言结构体的伪结构来存储数据。这样的伪结构仅仅有两种数据类型：无符号数和表。

    `无符号数`：是基本数据类型。以u1、u2、u4、u8分别代表1个字节、2个字节、4个字节、8个字节的无符号数，能够用来描写叙述数字、索引引用、数量值或者依照UTF-8编码构成的字符串值。
    
    `表`：由多个无符号数或者其它表作为数据项构成的复合数据类型。全部表都习惯性地以 `_info` 结尾。整个Class文件本质上就是一张表，例如以下所看到的：
    
    |类型 |名称 |数量
    |:---:|:---:|:---:|
    |u4 |magic |1
    |u2 |minor_version |1
    |u2 |major_version |1
    |u2 |constant_pool_count |1
    |cp_info |constant_pool |constant_pool_count-1
    |u2 |access_flags |1
    |u2 |this_class |1
    |u2 |super_class |1
    |u2 |interfaces_count |1
    |u2 |interfaces |interfaces_count
    |u2 |fields_count |1
    |field_info |fields |fields_count
    |u2 |methods_count |1
    |method_info |methods |methods_count
    |u2 |attributes_count |1
    |attribute_info |attributes |attributes_count


## class 文件编译

以 `ClassTest` project 为例。

### Eclipse

测试环境：

Eclipse Java EE IDE for Web Developers, Oxygen Release (4.7.0), 20170620-1800

JRE: 1.8.0_112-release-736-b21 amd64

#### 初识项目结构

项目目录结构如下

  ```
  ClassTest
  ├─.settings
  ├─bin
  │  └─xyz
  │      └─jiangjiangy
  │          └─test
  └─src
      └─xyz
          └─jiangjiang
              └─test
  ```

由 `ClassTest/.classpath` 文件中属性指定 path。如 `classpathentry.output = bin` 属性指定输出文件夹为 `bin`，如下：

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <classpath>
    <!-- 指定编译 JDK 版本 -->
  	<classpathentry kind="con" path="org.eclipse.jdt.launching.JRE_CONTAINER/org.eclipse.jdt.internal.debug.ui.launcher.StandardVMType/JavaSE-1.8"/>
    <!-- 指定 src 目录 -->
  	<classpathentry kind="src" path="src"/>
    <!-- 指定 output 目录 -->
  	<classpathentry kind="output" path="bin"/>
  </classpath>
  ```

#### 编写 java 源文件

  ```java
  package xyz.jiangjiangy.test;
  /**
   * @Author jiangydev.
   * @Discription
   * @Time Created in 15:55 2017/12/26.
   * @Modified By
   */
  public class Test {

      private int m;

      public int getM(){
          return m + 1;
      }

  }
  ```

幸运的是，Eclipse 默认自动编译，在菜单栏有 `project` -> `bulid automatically` 选项，如果勾选，则帮我们自动编译写好的 java 源文件为 class 文件，class 文件的输出目录为 `ClassTest/bin/xyz/jiangjiangy/test`。你将会看到一个名为 `Test.class` 的文件。

  ![Eclipse 下 Test.java 编译后的字节码文件](/img/in-post/java/Eclipse-TestJavaToClass.png)

### IntelliJ IDEA

测试环境：

IntelliJ IDEA 2017.1.3, Build #IU-171.4424.56, For educational use only.

JRE: 1.8.0_112-release-736-b21 amd64

#### 初识项目结构

项目目录结构如下

  ```
  ├─.idea
  │  ├─dictionaries
  │  └─inspectionProfiles
  ├─out                             # 该目录当有 java 文件编译后才会生成。
  │  └─production
  │      └─ClassTest
  │          └─xyz
  │              └─jiangjiangy
  │                  └─test
  └─src
      └─xyz
          └─jiangjiangy
              └─test
  ```

由 `ClassTest/.idea/workspace.xml` 文件中属性指定 path，这个值具体由 IDEA 决定，修改文件后不生效。

#### 编写 java 源文件

Test.java 内容与 Eclipse 中的 `Test.java` 一致.

与 Eclipse 不同，IntelliJ IDEA 并不会默认开启自动编译，如果要开启，可以按照以下步骤：`Settings` -> `Build, Execution, Deployment` -> `Compiler` -> `Build project automatically`。这里为了测试，暂时勾选这一选项。

  ![IntelliJ IDEA 下 Test.java 编译后的字节码文件](/img/in-post/java/Eclipse-TestJavaToClass.png)

### class 文件输出目录

class 文件输出配置由编译工具决定，这说明可以自定义输出目录，而且可能同一款 IDE 的不同版本，其值也会不同。对于一般 project，默认情况下，Eclipse 输出在 `$WORKSPACE_DIR$/$PROJECT_DIR$/bin`，IDEA 输出在 `$WORKSPACE_DIR$/$PROJECT_DIR$/out/production/$PROJECT_DIR$`。

另外，若项目由 Maven 管理，输出在 `$WORKSPACE_DIR$/$PROJECT_DIR$/target/classes`。若为 Maven + web-app，输出在 `$WORKSPACE_DIR$/$PROJECT_DIR$/target/$PROJECT_DIR$/WEB-INF/classes`。

  > 注意
  >
  >
  > 在以后的开发中，可能会使用 Maven 管理项目，如果部署项目出现找不到文件的错误，这时最好先检查对应的部署目录下是否包含了这些文件，具体的情况需要具体分析，这里不涉及排错的解析。

## class 文件详解

### magic(魔数)

每一个 class 文件的头4个字节称为 magic，它唯一的作用是确定这个文件是否为一个能被虚拟机接受的 Class 文件。

非常多文件存储标准中都使用魔数来进行身份识别，如图片格式gif、jpeg等。使用魔数而不是拓展名来进行识别主要是基于安全方面的考虑，因为文件拓展格式能够任意修改。

Class 文件的魔数为：`0xCAFEBABE`（咖啡宝贝？）

### 版本

第5，6个字节是次版本（Minor Version），第7，8个字节是主版本（Major Version）。

高版本号的 JDK 能够向下兼容曾经版本号的 Class 文件，可是无法执行以后版本号的 Class 文件，即使文件格式并未发生变化，虚拟机也必须拒绝执行超过其版本号的Class文件。
