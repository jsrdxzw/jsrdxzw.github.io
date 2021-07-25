---
title: 分布式id
date: 2021-07-25 13:49:53
excerpt: 介绍全局分布式id的主要原理和实现
tags: 分布式
categories: 后端
---

在数据库集群模式中，往往不能使用自增id，这时候可以在业务端生成全局唯一id。主要的方法有三种

### UUID
直接使用jdk提供的UUID，优点是简单，缺点是非自增，无规律

### snowflake算法

---

![雪花算法](/img/snowflake.png)

雪花算法主要结构如下
+ 1 位，不用。二进制中最高位为 1 的都是负数，但是我们生成的 id 一般都使用整数，所以这个最高位固定是`0`
+ 41 位，用来记录时间戳（毫秒）, 理论上可以支持69年左右
+ 10 位，用来记录集群id和工作机器id
  - 可以部署在 `2^10 = 1024` 个节点，包括 5 位 datacenterId 和 5 位 workerId
+ 12 位序列号，用来记录同毫秒内产生的不同 id, 一毫秒同一个机器可以产生4096个id

SnowFlake 可以保证：



同一台服务器所有生成的 id 按时间趋势递增

整个分布式系统内不会产生重复 id（因为有 datacenterId 和 workerId 来做区分）

`依赖机器时钟，需要做时钟同步`
```shell
ntpdate -u ntp.api.bz
```
主要的算法实现
```java
public synchronized long nextId() {
    long timestamp = timeGen();
    // 获取当前时间戳如果小于上次时间戳，则表示时间戳获取出现异常
    if (timestamp < lastTimestamp) {
        System.err.printf("clock is moving backwards.  Rejecting requests until %d.", lastTimestamp);
        throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds",
                lastTimestamp - timestamp));
    }
    // 获取当前时间戳如果等于上次时间戳
    // 说明：还处在同一毫秒内，则在序列号加1；否则序列号赋值为0，从0开始。
    if (lastTimestamp == timestamp) {
        sequence = (sequence + 1) & sequenceMask;
        // sequence=0表示已经打到了4096，超过了边界，此时需要等待直到下一毫秒
        if (sequence == 0) {
            timestamp = tilNextMillis(lastTimestamp);
        }
    } else {
        sequence = 0;
    }
    //将上次时间戳值刷新
    lastTimestamp = timestamp;

    /**
     * 返回结果：
     * (timestamp - twepoch) << timestampLeftShift) 表示将时间戳减去初始时间戳，再左移相应位数
     * (datacenterId << datacenterIdShift) 表示将数据id左移相应位数
     * (workerId << workerIdShift) 表示将工作id左移相应位数
     * | 是按位或运算符，例如：x | y，只有当x，y都为0的时候结果才为0，其它情况结果都为1。
     * 因为个部分只有相应位上的值有意义，其它位上都是0，所以将各部分的值进行 | 运算就能得到最终拼接好的id
     */
    return ((timestamp - twepoch) << timestampLeftShift) |
            (datacenterId << datacenterIdShift) |
            (workerId << workerIdShift) |
            sequence;
}
```

具体的源码可以参考这里 [雪花算法源码](https://github.com/jsrdxzw/improve-java/blob/master/id_worker/src/main/java/globalid/IdWorker.java)

### 借助Redis的Incr命令获取全局唯⼀ID

---

Redis可以保证原子性提供全局id，也是一个推荐的做法

```java
Jedis jedis = new Jedis("127.0.0.1", 6379);
    try {
        long id = jedis.incr("id");
        System.out.println("从redis中获取的分布式id为：" + id);
    } finally {
        if (null != jedis) {
            jedis.close();
        } 
    }
```