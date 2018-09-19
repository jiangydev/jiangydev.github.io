---
layout:     post
title:      "Dubbo 负载均衡源码 - 轮询"
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

## RounndRobinLoadBalance(轮询)

一般的轮询，类似服务器遍历，即不考虑服务器性能的`取模轮循`，要求各个服务器有相同的性能，否则存在慢的提供者请求累积的问题，例如：第二台服务器很慢，但仍然可以接受调用请求，久而久之，第二台服务器会产生累积，无法及时处理。

### 解决方案

基于权限轮询服务器。

### 引用 Alibaba 开源项目 `Dubbo` 源码

```java
import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.common.utils.AtomicPositiveInteger;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class RoundRobinLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "roundrobin";

    // 创建一个线程安全的、同时发生的(并发的)HashMap，与 HashTable 类似，但与 HashMap 不同，不允许将 null 用作键或值
    private final ConcurrentMap<String, AtomicPositiveInteger> sequences = new ConcurrentHashMap<String, AtomicPositiveInteger>();

    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        int length = invokers.size(); // invoker数量
        int maxWeight = 0; // 最大权重值
        int minWeight = Integer.MAX_VALUE; // 最小权重值
        final LinkedHashMap<Invoker<T>, IntegerWrapper> invokerToWeightMap = new LinkedHashMap<Invoker<T>, IntegerWrapper>();
        int weightSum = 0; // 权重值和
        // 遍历invoker，记录权重最大值和最小值；如果权重值大于0，将invoker及其权重值包装类放入map集合，并累加权重值。
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            maxWeight = Math.max(maxWeight, weight); // 选择最大权重值
            minWeight = Math.min(minWeight, weight); // 选择最小权重值
            if (weight > 0) {
                invokerToWeightMap.put(invokers.get(i), new IntegerWrapper(weight));
                weightSum += weight;
            }
        }
        //
        AtomicPositiveInteger sequence = sequences.get(key);
        if (sequence == null) {
            // 如果指定键已经不再与某个值相关联，则将它与给定值关联
            sequences.putIfAbsent(key, new AtomicPositiveInteger());
            sequence = sequences.get(key);
        }
        int currentSequence = sequence.getAndIncrement();
        // 基于权重值的轮询
        if (maxWeight > 0 && minWeight < maxWeight) {
            int mod = currentSequence % weightSum;
            for (int i = 0; i < maxWeight; i++) {
                // 遍历 invoker/权重 集合，寻找符合的invoker
                for (Map.Entry<Invoker<T>, IntegerWrapper> each : invokerToWeightMap.entrySet()) {
                    final Invoker<T> k = each.getKey();
                    final IntegerWrapper v = each.getValue();
                    if (mod == 0 && v.getValue() > 0) {
                        return k;
                    }
                    // 如果服务器权重已经减到0，则不再减
                    if (v.getValue() > 0) {
                        v.decrement();
                        mod--;
                    }
                }
            }
        }
        // 取模轮询
        return invokers.get(currentSequence % length);
    }

    private static final class IntegerWrapper {
        private int value;

        public IntegerWrapper(int value) {
            this.value = value;
        }

        public int getValue() {
            return value;
        }

        public void setValue(int value) {
            this.value = value;
        }

        public void decrement() {
            this.value--;
        }
    }

}
```

### 举例

1. 基于权重值的轮询算法

   假设有服务器集合（服务器名：其权重值）：A(S0:1, S1:5, S2:5, S3:1, S4:3)，权重值不同，使用基于权重值的轮询算法。计算权重值总和（15），初始化一个原子性整数的序列并构建服务器与其权重值的映射集合，只要遍历一次映射集合。若当前所在的序列值为20，与15取模得5，开始遍历映射集合，因为 mod = 5, 服务器集合改变为A(S0:0, S1:5, S2:5, S3:1, S4:3)，mod 减一为 4；直到遍历到 mod = 0，此时服务器集合改变为A(S0:0, S1:1, S2:5, S3:1, S4:3)，会选择 S1 服务器。

   注意，需要维护这个序列，否则会造成每次请求来的时候初始化序列，始终调用 S0 服务器。

   ```java
   private final ConcurrentMap<String, AtomicPositiveInteger> sequences = new ConcurrentHashMap<String, AtomicPositiveInteger>();
   ```

   ```
   final 变量赋值不可变（引用不可变，对象内容可变），初始化块。
   ```

   


  2. 一般取模轮询算法

     假设有服务器集合（服务器名：其权重值）：A(S0:1, S1:1, S2:1, S3:1, S4:1)，权重值相同。invoker数量为5，若当前所在序列值为20，取服务器S0。

### 优化

遍历服务器时，可以添加动态删除无效的服务器方法。
