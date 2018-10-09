---
layout:     post
title:      "Java 数组、集合的使用问题"
subtitle:   ""
date:       2017-10-09
author:     "jiangydev"
header-img: "img/post-bg-java.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Java
---

[TOC]

## 集合的使用问题

#### Arrays.asList()异常 UnsupportOperationException
1. 问题分析
Arrays.asList() 转换后的集合是AbstractList类型，只是转换接口，仍是数组，不能使用其修改集合相关的方法，因此不能对其add/remove/clear，否则抛出异常UnsupportOperationException.

2. 解决方案
```java
List<String> list = new ArrayList<>(Arrays.asList(new String[]{"1", "4", "4"}));
```

#### HashSet 与 ArrayList
1. HashSet 判断、删除和添加元素等操作依据的是被操作元素所在的类的`hashCode()`和`equals()`这两个方法
   - HashSet 在存储元素时，先查看两个对象的哈希值是否一样的。
     - 如果哈希值是一样，再调用元素对象的equals方法，查看两个对象的内容是否一样。
     - 如果 equlas 比较的值一样，说明两个对象相同，不能添加到 HashSet 对应的集合中。
     - 如果 equlas 比较的值不一样，说明两个对象不同，可以添加到 HashSet 对应的集合中
   - 如果哈希值是不一样，直接认为两个对象不一样，equals 方法不起作用。

2. ArrayList 做同等的操作，依据的仅仅 equals() 方法

