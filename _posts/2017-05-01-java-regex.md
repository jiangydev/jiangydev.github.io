---
layout:     post
title:      "Java 正则表达式"
subtitle:   "常见操作，三种量词（贪婪、懒惰、强占），表达式样例"
date:       2017-05-01
author:     "jiangydev"
header-img: "img/post-bg-java.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Java
    - Regex
---

[TOC]

## 正则常见操作

### 1 匹配

```java
// String 的 matches方法
// 匹配手机号
x.matches("1[358][0-9]{9}");
// "1[358]\\d{9}"
```

### 2 切割

```java
String的split方法
//切空格
x.split(" +")
//切小数点
x.split("\\.")
//切重复元素
// 组的概念：((A)(B(C)))
x.split("(.)\\1+")
```

### 3 替换

```java
String的replaceAll()
//重复元素替换为一个
x.replaceAll("(.)\\1+","$1")
//电话号码隐藏中间四位
x.replaceAll("(\\d{3})\\d{4}(\\d{4})","$1****$2")
```

### 4 获取

```java
str = "";
regex = "\\b[a-z]{3}\\b";
Pattern p = Pattern.compile(regex);
Matcher m = p.matcher(str);
while(m.find()) {
    m.group();
}
```

### 5 实例 - IP地址排序

```java
String ip_str = "192.168.1.1   127.0.0.1  5.5.5.5  102.131.2.98"
// 1. 补零
ip_str = ip_str.replaceAll("(\\d+)","00$1")
// 2. 取三位
ip_str = ip_str.replaceAll("0*(\\d{3})","$1")
// 3. 切
String[] ips = ip_str.split(" +");
// 4. 排序
TreeSet<String> ts = new TreeSet<>();
for (String ip : ips) {
    ts.add();
}
// 5. 输出
for (String ip : ts) {
    System.out.println(ip.replaceAll("0*(\\d+)","$1"));
}
```



## Java 正则表达式中的三种量词

Java 中的正则位于`java.util.regex`包中，这个包中只有一个 MatchResult 接口和 Matcher、Pattern 两个类。

正则中的数量词有 `Greedy(贪婪)`、`Reluctant(懒惰)`和 `Possessive(强占)`三种。

### Greedy 数量词

`X?`：X，一次或一次也没有

`X*`：X，零次或多次

`X+`：X，一次或多次

`X{n}`：X，恰好 n 次

`X{n,}`：X，至少 n 次

`X{n,m}`：X，至少 n 次，但是不超过 m 次

Greedy是最常用的，它的匹配方式是先把整个字符串吞下，然后匹配整个字符串，如果不匹配，就从右端吐出一个字符，再进行匹配，直到找到匹配或把整个字符串吐完为止。

```java
Matcher m=Pattern.compile("a.*b")  
                   .matcher("a====b=========b=====");  
while(m.find()){  
    System.out.println(m.group());  
}
// 输出:
// a====b=========b  
```


因为总是从最大 匹配开始匹配，故称贪婪。

### Reluctant 数量词

`X??`: X，一次或一次也没有

`X*?`: X，零次或多次

`X+?`: X，一次或多次

`X{n}?`: X，恰好 n 次

`X{n,}?`: X，至少 n 次

`X{n,m}?`: X，至少 n 次，但是不超过 m 次

Reluctant正好和Greedy相反，它先从最小匹配开始，先从左端吞入一个字符，然后进行匹配，若不匹配就再吞入一个字符，直到找到匹配或将整个字符串吞入为止。

```java
Matcher m=Pattern.compile("a.*?b").matcher("a====b=========b=====");  
while(m.find()){  
    System.out.println(m.group());  
}
// 输出:
// a====b
```
因为总是从最小匹配开始，故称懒惰。

### Possessive 数量词

`X?+`: X，一次或一次也没有

`X*+`: X，零次或多次

`X++`: X，一次或多次

`X{n}+`: X，恰好 n 次

`X{n,}+`: X，至少 n 次

`X{n,m}+`: X，至少 n 次，但是不超过 m 次

Possessive和Greedy的匹配方式一样，先把整个字符串吞下，然后匹配整个字符串，如果匹配，就认为匹配，如果不匹配，就认为整个字符串不匹配，它不会从右端吐出一个字符串再进行匹配，只进行一次

```java
Matcher m=Pattern.compile("a.*+b").matcher("a====b=========b=====");  
while(m.find()){  
    System.out.println(m.group());  
}  
// 输出：
```

因为贪婪但并不聪明，故称强占。



## 正则表达式样例

```java
 /**
 * 正则表达式：验证用户名
 */
public static final String REGEX_USERNAME = "^[a-zA-Z]\\w{5,17}$";

/**
 * 正则表达式：验证密码
 */
public static final String REGEX_PASSWORD = "^[a-zA-Z0-9]{6,16}$";

/**
 * 正则表达式：验证手机号
 */
public static final String REGEX_MOBILE = "^((13[0-9])|(15[^4,\\D])|(18[0,5-9]))\\d{8}$";

/**
 * 正则表达式：验证邮箱
 */
public static final String REGEX_EMAIL = "^([a-z0-9A-Z]+[-|\\.]?)+[a-z0-9A-Z]@([a-z0-9A-Z]+(-[a-z0-9A-Z]+)?\\.)+[a-zA-Z]{2,}$";

/**
 * 正则表达式：验证汉字
 */
public static final String REGEX_CHINESE = "^[\u4e00-\u9fa5],{0,}$";

/**
 * 正则表达式：验证身份证
 */
public static final String REGEX_ID_CARD = "(^\\d{18}$)|(^\\d{15}$)";

/**
 * 正则表达式：验证URL
 */
public static final String REGEX_URL = "http(s)?://([\\w-]+\\.)+[\\w-]+(/[\\w- ./?%&=]*)?";

/**
 * 正则表达式：验证IP地址
 */
public static final String REGEX_IP_ADDR = "(25[0-5]|2[0-4]\\d|[0-1]\\d{2}|[1-9]?\\d)";
```

