---
title: 一致性Hash算法
date: 2021-07-23 14:30:40
excerpt: 一致性Hash算法原理和实现
tags:
- 分布式
- 一致性Hash
categories:
- 分布式
---

### Hash算法使用场景

---
Hash算法在分布式领域中有广泛的应用，比如分布式集群架构Redis、Hadoop、ElasticSearch，Mysql分库分表，Nginx负载均衡等
+ 请求负载均衡(比如nginx的ip_hash策略)
+ 分布式存储
  比如根据key做hash取余数，然后请求不同的redis节点

### 普通Hash算法

---
普通Hash算法存在这样的问题，即如果其中一个节点宕机了，那么hash的结果就不对了，
如果后台服务器很多台，客户端也有很多，那么影响是很大的，缩容和扩容都会存在这样的问题。
**大量用户的请求会被路由到其他的目标服务器处理**，用户在原来服务器中的会话都会丢失。
缓存命中率低。


### 一致性Hash算法

---
一致性Hash算法，保证无论是扩容还是缩容，都只影响一小部分的客户端，并且保证一定的缓存命中率（很多请求还是走的原来的服务节点）。
可以增加虚拟节点解决`Hash 环数据倾斜的问题`，理论上虚拟节点越多分布越均匀


### 实现一致性Hash算法

---

这里使用`TreeMap`实现Hash环，hash算法使用默认的`hashCode()`，如果为了hash均匀，可以采用其他的Hash算法比如
`CRC32_HASH、FNV1_32_HASH、KETAMA_HASH`


```java
public class ConsistentHashWithVirtual {
    public static void main(String[] args) {
        // 定义服务器ip
        String[] tomcatServers = new String[]{"123.111.0.0", "123.101.3.1", "111.20.35.2", "123.98.26.3"};
        // hash环用有序Map来模拟
        final TreeMap<Integer, String> treeMap = new TreeMap<>();
        // 定义针对每个真实服务器虚拟出来几个节点
        int virtualCount = 3;

        for (String tomcatServer : tomcatServers) {
            // 求出每一个ip的hash值，对应到hash环上，存储hash值与ip的对应关系
            // 直接使用hashCode可能会造成分布不均匀的情况, 可以使用 FNVI_32_HASH 算法
            // 使用 FNVI_32_HASH 算法计算 Hash 值，在服务器增加后，缓存的命中率为 78% 左右
            int hash = Math.abs(tomcatServer.hashCode());
            treeMap.put(hash, tomcatServer);

            // 增加虚拟节点
            for (int i = 0; i < virtualCount; i++) {
                // 123.111.0.0#0,123.111.0.0#1,123.111.0.0#2
                int virtualHash = Math.abs((tomcatServer + "#" + i).hashCode());
                treeMap.put(virtualHash, tomcatServer);
            }
        }

        // 定义客户端IP
        String[] clients = new String[]{"10.78.12.3","113.25.63.1","126.12.3.8"};

        for(String client : clients) {
            int hash = Math.abs(client.hashCode());
            // 向右寻找第一个 key
            Map.Entry<Integer, String> subEntry = treeMap.ceilingEntry(hash);
            // 设置成一个环，如果超过尾部，则取第一个点
            subEntry = subEntry == null ? treeMap.firstEntry() : subEntry;
            System.out.println("==========>>>>客户端：" + client + " 被路由到服务器：" + subEntry.getValue());
        }
    }
}
```

### Nginx配置一致性Hash算法

---
`ngx_http_upstream_consistent_hash` 模块是一个负载均衡器，使用一个内部一致性hash算法来选择
合适的后端节点。

该模块可以根据配置参数采取不同的方式将请求均匀映射到后端机器
+ consistent_hash $remote_addr:可以根据客户端ip映射
+ consistent_hash $request_uri:根据客户端请求的uri映射
+ consistent_hash $args:根据客户端携带的参数进行映

```text
upstream server {
    consistence_hash $$request_uri;
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}
```
