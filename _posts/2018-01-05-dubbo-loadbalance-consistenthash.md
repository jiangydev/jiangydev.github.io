---
layout:     post
title:      "Dubbo 负载均衡源码 - 一致性哈希"
subtitle:   ""
date:       2018-01-05
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
tags:
    - Load Balance
    - Dubbo
    - Algorithm
    - 源码学习
---

## ConsistentHashLoadBalance(一致性哈希)

### 参考资料

1. [Dubbo负载均衡：一致性Hash的实现分析](http://blog.csdn.net/revivedsun/article/details/71022871)
2. [一致性哈希算法](http://blog.csdn.net/cywosp/article/details/23397179)



### 引用 Alibaba 开源项目 `Dubbo` 源码

```java
import com.alibaba.dubbo.common.Constants;
import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;

import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.List;
import java.util.SortedMap;
import java.util.TreeMap;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class ConsistentHashLoadBalance extends AbstractLoadBalance {

    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = new ConcurrentHashMap<String, ConsistentHashSelector<?>>();

    @SuppressWarnings("unchecked")
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        // identityHashCode() 返回给定对象的哈希码，该代码与默认的方法 hashCode() 返回的代码一样，无论给定对象的类是否重写 hashCode()。null 引用的哈希码为 0。
        int identityHashCode = System.identityHashCode(invokers);
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        if (selector == null || selector.identityHashCode != identityHashCode) {
            selectors.put(key, new ConsistentHashSelector<T>(invokers, invocation.getMethodName(), identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
        return selector.select(invocation);
    }

    private static final class ConsistentHashSelector<T> {

        // 虚拟节点
        private final TreeMap<Long, Invoker<T>> virtualInvokers;

        // 副本数量
        private final int replicaNumber;

        private final int identityHashCode;

        // 参数索引数组
        private final int[] argumentIndex;

        ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
            // 创建 TreeMap 保存节点
            this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
            // 生成调用节点HashCode
            this.identityHashCode = identityHashCode;
            // 获取Url
            URL url = invokers.get(0).getUrl();
            // 获取配置的副本节点数（如：<dubbo:parameter key="hash.nodes" value="320" />），如果没有配置，则默认使用160
            this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);
            // 获取需要进行hash的参数索引数组，如果没有设置，则默认对第一个参数进行hash（如：<dubbo:parameter key="hash.arguments" value="0,1" />）
            String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));
            argumentIndex = new int[index.length];
            for (int i = 0; i < index.length; i++) {
                argumentIndex[i] = Integer.parseInt(index[i]);
            }
            // 创建虚拟节点
            // 对每个 invoker 生成 replicNumber 个虚拟节点，存放在 TreeMap 中
            for (Invoker<T> invoker : invokers) {
                String address = invoker.getUrl().getAddress();
                for (int i = 0; i < replicaNumber / 4; i++) {
                    // 根据md5算法为每4个结点生成一个消息摘要，摘要长为16字节128位。
                    byte[] digest = md5(address + i);
                    // 随后将128位分为4部分，0-31,32-63,64-95,95-128，并生成4个32位数，存于long中，long的高32位都为0
                    // 并作为虚拟结点的key。
                    for (int h = 0; h < 4; h++) {
                        long m = hash(digest, h);
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }

        // 选择节点
        public Invoker<T> select(Invocation invocation) {
            // 根据调用参数来生成Key
            String key = toKey(invocation.getArguments());
            // 根据这个参数生成消息摘要
            byte[] digest = md5(key);
            // 调用hash(digest, 0), 将消息摘要转换为hashCode, 这里仅取0-31位来生成hashCode
            // 调用selectForKey方法选择节点
            return selectForKey(hash(digest, 0));
        }

        private String toKey(Object[] args) {
            StringBuilder buf = new StringBuilder();
            // 由于hash.arguments没有进行配置，因为只取方法的第一个参数作为key
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }

        // 根据hashCode选择节点
        private Invoker<T> selectForKey(long hash) {
            Invoker<T> invoker;
            Long key = hash;
            // 若hashCode直接与某个虚拟节点的key一样，则直接返回该节点
            if (!virtualInvokers.containsKey(key)) {
                // 若不一致，找到一个最小上界的key所对应的节点
                SortedMap<Long, Invoker<T>> tailMap = virtualInvokers.tailMap(key);
                // 使用TreeMap的firstKey方法，来选择最小上界
                if (tailMap.isEmpty()) {
                    key = virtualInvokers.firstKey();
                } else {
                    key = tailMap.firstKey();
                }
            }
            invoker = virtualInvokers.get(key);
            return invoker;
        }

        // 用来生成hashCode, 该函数将生成的结果转换为long类，这是因为生成的结果是一个32位数，若用int保存可能会产生负数。而一致性hash生成的逻辑环其hashCode的范围是在 0 - MAX_VALUE之间。因此为正整数，所以这里要强制转换为long类型，避免出现负数。
        private long hash(byte[] digest, int number) {
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                    | (digest[number * 4] & 0xFF))
                    & 0xFFFFFFFFL;
        }

        private byte[] md5(String value) {
            MessageDigest md5;
            try {
                // 返回实现指定摘要算法的 MessageDigest 对象。
                md5 = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            // 重置摘要以供再次使用。
            md5.reset();
            byte[] bytes;
            try {
                bytes = value.getBytes("UTF-8");
            } catch (UnsupportedEncodingException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            // 使用指定的 bytes 数组更新摘要。
            md5.update(bytes);
            // 通过执行诸如填充之类的最终操作完成哈希计算。在调用此方法之后，摘要被重置
            return md5.digest();
        }

    }

}
```
