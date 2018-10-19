---
layout:     post
title:      "Dubbo 负载均衡源码 - 随机"
subtitle:   ""
date:       2018-01-04
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Load Balance
    - Dubbo
    - Algorithm
    - 源码学习
---

[TOC]

## RandomLoadBalance(随机)

随机算法有两种，一种根据服务器数量随机分配，这种方式不考虑服务器差异，可能会导致请求累计、服务器资源分配不当造成浪费等问题；另一种方式，根据服务器权重值分配，相对更优。

### 引用 Alibaba 开源项目 `Dubbo` 源码

```java
import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;

import java.util.List;
import java.util.Random;

public class RandomLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "random";

    private final Random random = new Random();

    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // invoker数量
        int totalWeight = 0; // 权重值总和
        boolean sameWeight = true; // 是否每一个invoker有相同的权重?
        // 遍历invoker，计算权重值总和，并判断是否权重值都相同
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            totalWeight += weight; // Sum
            if (sameWeight && i > 0
                    && weight != getWeight(invokers.get(i - 1), invocation)) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // 如果不是每个invoker有相同权重并且至少有一个invoker的权重大于0，选择基于权重值的随机算法

            // 从权重总和值的范围内取随机数，即偏移量
            int offset = random.nextInt(totalWeight);
            // 返回基于这个随机数的invoker
            for (int i = 0; i < length; i++) {
                offset -= getWeight(invokers.get(i), invocation);
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // 如果所有的invoker权重值相同或权重值总和为0, 返回随机invoker
        return invokers.get(random.nextInt(length));
    }

}
```

### 举例

1. 基于权重的随机

   假设有服务器集合（服务器名：其权重值）：A(S0:1, S1:5, S2:5, S3:1, S4:3)，权重值不同，使用基于权重值的随机算法，先计算权重总和（15），在 0-14 中产生随机整数，若随机数为 10，由 (10-1-5-5 = -1 < 0) 可知，随机数命中的服务器为 S2。

2. 均等随机

   假设有服务器集合（服务器名：其权重值）：A(S0:1, S1:1, S2:1, S3:1, S4:1)，权重值相同，使用均等随机算法，已知服务器数总和（5），在 0-4 中产生随机整数，若随机数为 4，则命中的服务器为 S4。