---
title: session一致性
excerpt: session在分布式环境下的一致性问题和解决方案
date: 2021-07-26 18:44:16
tags:
- 分布式
- session
categories: 分布式
---

### session在分布式环境下的问题

Session在集群情况下，可能会发生不一致的问题，即Session不能在多机器共享

![Session不一致问题](/img/session.png)

### 解决方案

#### 1. 使用Nginx的`ip_hash` 保证同一个ip请求同一个实例

优点:
+ 配置简单，不入侵应用，不需要额外修改代码

缺点:
+ 服务器重启，Session会丢失，因为Session本质上保存在服务器内存中
+ 单点负载，故障风险

#### 2. 使用Redis统一保存Session

优点:
+ 能适应各种负载均衡策略 
+ 服务器重启或者宕机不会造成Session丢失 
+ 扩展能力强
+ 适合大集群数量使用

使用`Spring Session`可以非常方便的实现Redis管理Session
1. 引入jar包
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.session</groupId>
  <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

2. 配置redis
```yaml
spring:
  redis:
    database: 0
    host: 127.0.0.1
    port: 6379
```

3. 添加注解
```java
@EnableRedisHttpSession
@SpringBootApplication
public class SpringApplication {
    
}
```

4. 源码分析

```java
// 其实就是自定义了一个filter
public class SessionRepositoryFilter<S extends Session> extends OncePerRequestFilter {
    
    // 请求中只会调用一次
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);

        SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryRequestWrapper(
                request, response, this.servletContext);
        SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryResponseWrapper(
                wrappedRequest, response);

        try {
            filterChain.doFilter(wrappedRequest, wrappedResponse);
        }
        finally {
            // 这里保存到redis
            wrappedRequest.commitSession();
        }
    }
}
```

关注 `SessionRepositoryRequestWrapper.getSession()` 方法

```java
// 创建了一个session，但是还没有保存到Redis
S session = SessionRepositoryFilter.this.sessionRepository.createSession();
// wrappedRequest.commitSession() 里面的方法
private void commitSession() {
    HttpSessionWrapper wrappedSession = getCurrentSession();
        if (wrappedSession == null) {
            if (isInvalidateClientSession()) {
                SessionRepositoryFilter.this.httpSessionIdResolver.expireSession(this,
                this.response);
            }
        }
        else {
            S session = wrappedSession.getSession();
            clearRequestedSessionCache();
            // 在里面保存
            SessionRepositoryFilter.this.sessionRepository.save(session);
            String sessionId = session.getId();
        if (!isRequestedSessionIdValid()
            || !sessionId.equals(getRequestedSessionId())) {
                SessionRepositoryFilter.this.httpSessionIdResolver.setSessionId(this,
                this.response, sessionId);
            }
        }
}
```

```java
// 核心保存方法
private void saveDelta() {
    if (this.delta.isEmpty()) {
        return;
    }
    String key = getSessionKey(getId());
    RedisSessionRepository.this.sessionRedisOperations.opsForHash().putAll(key, new HashMap<>(this.delta));
    RedisSessionRepository.this.sessionRedisOperations.expireAt(key,
            Date.from(Instant.ofEpochMilli(getLastAccessedTime().toEpochMilli())
                    .plusSeconds(getMaxInactiveInterval().getSeconds())));
    this.delta.clear();
}
```
