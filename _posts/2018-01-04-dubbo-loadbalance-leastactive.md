---
layout:     post
title:      "Dubbo 负载均衡源码 - 最少活跃数"
subtitle:   ""
date:       2018-01-04
author:     "jiangydev"
header-/img: "/img/post-bg-cloud.jpg"
tags:
    - Load Balance
    - Dubbo
    - Algorithm
    - 源码学习
---

## LeastActiveLoadBalance(最少活跃数)

根据统计收到请求时各提供者已有的请求数，在此基础上再考虑服务器权重值，使请求处理慢、性能相对不佳的提供者收到更少的请求。

### 引用 Alibaba 开源项目 `Dubbo` 源码

```java
import com.alibaba.dubbo.common.Constants;
import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.RpcStatus;

import java.util.List;
import java.util.Random;

public class LeastActiveLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "leastactive";

    private final Random random = new Random();

    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // invoker数量
        int leastActive = -1; // invoker最少活跃的值
        int leastCount = 0; // 有相同最少活跃值的invoker数量 (leastActive)
        int[] leastIndexs = new int[length]; // 有相同最少活跃值的invoker索引 (leastActive)
        int totalWeight = 0; // 权重值总和
        int firstWeight = 0; // 初始化值, 用来比较
        boolean sameWeight = true; // 是否每个invoker有相同的权重值?
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive(); // 激活数量
            int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT); // 权重
            if (leastActive == -1 || active < leastActive) { // 重新开始, 当找到有一个更小最少活跃值的invoker时
                leastActive = active; // 记录当前最少活跃值
                leastCount = 1; // 重设 leastCount, 基于当前 leastCount 重新计数
                leastIndexs[0] = i; // Reset
                totalWeight = weight; // Reset
                firstWeight = weight; // 记录第一个invoker权重
                sameWeight = true; // Reset, every invoker has the same weight value?
            } else if (active == leastActive) { // 如果当前invoker的活跃值等于 leaseActive, then accumulating.
                leastIndexs[leastCount++] = i; // Record index number of this invoker
                totalWeight += weight; // Add this invoker's weight to totalWeight.
                // If every invoker has the same weight?
                if (sameWeight && i > 0
                        && weight != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // assert(leastCount > 0)
        if (leastCount == 1) {
            // 如果准确得到仅有一个invoker有最少活跃值, 直接返回这个invoker.
            return invokers.get(leastIndexs[0]);
        }
        if (!sameWeight && totalWeight > 0) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            int offsetWeight = random.nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexs[i];
                offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
                if (offsetWeight <= 0)
                    return invokers.get(leastIndex);
            }
        }
        // 如果所有的invoker有相同的权重值或 totalWeight=0, return evenly.
        return invokers.get(leastIndexs[random.nextInt(leastCount)]);
    }
}
```

### 举例

假设有服务器集合（服务器名：其权重值：活跃值）：A(S0:1:2, S1:5:4, S2:5:3, S3:1:2, S4:3:2)，权重值不同，通过遍历invokers，找出最小活跃值服务器集合：B(S0:1:2, S3:1:2, S4:3:2)，活跃值均为2，然后通过 RandomLoadBalance（基于权重的随机负载均衡算法）得到最终的invoker（S0，S3的概率为1/5，S3的概率为3/5）。
