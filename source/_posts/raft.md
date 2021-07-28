---
title: raft算法
date: 2021-07-28 14:14:01
excerpt: raft算法介绍
tags:
- 分布式
- raft
categories: 后端
---

### raft算法简介

raft算法是Paxos算法的工程实现，主要特点就是通过较为简单的算法实现分布式系统的数据一致性和高可用，
Raft通过选举一个领导人，然后给予他全部的管理复制日志的责任来实现一致性。
Raft算法中任何服务器都可以扮演下面的角色之一:
1. 领导者(leader): 处理客户端交互，日志复制等动作，一般一次只有一个领导者
2. 候选者(candidate): 候选者就是在选举过程中提名自己的实体，一旦选举成功，则成为领导者
3. 跟随者(follower): 类似选民，完全被动的角色，这样的服务器等待被通知投票

他们之间的身份变化如下图
![raft算法角色变化](/img/raft-election.png)

### leader选举

1. 初始状态下集群中的所有节点都处于 `follower` 状态

![初始状态](/img/raft-election1.png)

2. 某一时刻，其中的一个 `follower` 由于没有收到 `leader` 的 `heartbeat` 率先发生 `election timeout` 进而发起选举

![某一时刻发起选取](/img/raft-election2.png)

3. 只要集群中超过半数的节点接受投票，`candidate` 节点将成为即切换 `leader` 状态

![超过半数选取为新leader](/img/raft-election3.png)

4. 成为 `leader` 节点之后，`leader` 将定时向 `follower` 节点同步日志并发送 `heartbeat`

![同步日志并发送心跳](/img/raft-election4.png)

### 日志复制

raft协议的 `log replication`  有点类似 2PC ，但是不同的是raft只要求大多数节点的回复即可，raft保证的是最终一致性
leader会不断尝试给follower发log entries，直到所有节点的log entries都相同。

**Raft 协议要求投票只能投给拥有最新数据的节点**, 保证了在同步log的时候leader挂掉，重新选举leader的时候，数据不丢失

![Raft日志复制过程.png](/img/raft-replication.png)

主要步骤如下:
1. 客户端的每一个请求都包含被复制状态机执行的指令。 
2. leader把这个指令作为一条新的日志条目添加到日志中，然后并行发起 RPC 给其他的服务器，让他们复制这 条信息。
3. 跟随者响应ACK,如果 follower 宕机或者运行缓慢或者丢包，leader会不断的重试，直到所有的 follower 最终 都复制了所有的日志条目。 
4. 通知所有的Follower提交日志，同时领导人提交这条日志到自己的状态机中，并返回给客户端。
5. 如果committed状态后client未收到leader响应，则client会重新发送请求，需要做幂等保证一致性。

### 脑裂情况

raft也可以保证在脑裂情况保证数据的一致性

![脑裂情况图示](/img/brain.png)

### 参考资料

[raft动画](http://thesecretlivesofdata.com/raft/)

[raft复制日志各状态分析](https://www.cnblogs.com/mindwind/p/5231986.html)
