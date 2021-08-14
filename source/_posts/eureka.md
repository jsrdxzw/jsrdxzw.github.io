---
title: eureka源码分析
excerpt: eureka源码解读和分析
date: 2021-08-13 14:17:03
tags:
- spring cloud
- eureka
categories: 后端
---

## eureka server
eureka 分为服务器和客户端，这里先分析服务器端的主要代码

### 自动装配

Spring Cloud 的`eureka`利用了 Spring Boot 提供的自动装配功能

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```

Spring启动的时候加载`EurekaServerAutoConfiguration`

```java
@Configuration(proxyBeanMethods = false)
// 实例化并加入Spring容器
@Import(EurekaServerInitializerConfiguration.class)
// 条件注入的前提条件
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class, InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {
}
```

注入的条件是来自EurekaServerMarkerConfiguration的 Marker, 看一下这个类

```java
@Configuration(proxyBeanMethods = false)
public class EurekaServerMarkerConfiguration {

	@Bean
	public Marker eurekaServerMarkerBean() {
		return new Marker();
	}

	class Marker {

	}

}
```

可以发现需要EurekaServerMarkerConfiguration去初始化这个Marker，
而EurekaServerMarkerConfiguration的初始化可以从下面的EnableEurekaServer来完成

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 实例化并加入Spring容器，这样Marker也能实例化了
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}
```

只有添加了`@EnableEurekaServer`注解，才会有后面的动作，这是成为一个eureka server的前提

### 主要配置类和初始化
```java
// 初始化仪表盘, eureka.dashboard.enabled=false 则可以关闭
@Bean
@ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
public EurekaController eurekaController() {
    return new EurekaController(this.applicationInfoManager);
}
// 集群模式下使用的注册器，eureka集群下每个节点是对等的
@Bean
public PeerAwareInstanceRegistry peerAwareInstanceRegistry(ServerCodecs serverCodecs) {
    this.eurekaClient.getApplications(); // force initialization
    return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig, serverCodecs, this.eurekaClient,
    this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
    this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
}
// 封装对等节点的相关操作
@Bean
@ConditionalOnMissingBean
public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry, ServerCodecs serverCodecs,
    ReplicationClientAdditionalFilters replicationClientAdditionalFilters) {
    return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig, this.eurekaClientConfig, serverCodecs,
    this.applicationInfoManager, replicationClientAdditionalFilters);
}
@Bean
@ConditionalOnMissingBean
public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs, PeerAwareInstanceRegistry registry,
    PeerEurekaNodes peerEurekaNodes) {
    return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs, registry, peerEurekaNodes,
    this.applicationInfoManager);
}

// 启动使用类
@Bean
public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
    EurekaServerContext serverContext) {
    return new EurekaServerBootstrap(this.applicationInfoManager, this.eurekaClientConfig, this.eurekaServerConfig,
    registry, serverContext);
}
// 使用jersey框架来提供rest服务
@Bean
public FilterRegistrationBean<?> jerseyFilterRegistration(javax.ws.rs.core.Application eurekaJerseyApp) {
    FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<Filter>();
    bean.setFilter(new ServletContainer(eurekaJerseyApp));
    bean.setOrder(Ordered.LOWEST_PRECEDENCE);
    bean.setUrlPatterns(Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));

    return bean;
}
```
继续看`PeerEurekaNodes`的start方法
```java
public class PeerEurekaNodes {
    public void start() {
        taskExecutor = Executors.newSingleThreadScheduledExecutor(
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                        thread.setDaemon(true);
                        return thread;
                    }
                }
        );
        try {
            // 更新对等的节点信息，在DefaultEurekaServerContext中调用
            updatePeerEurekaNodes(resolvePeerUrls());
            Runnable peersUpdateTask = new Runnable() {
                @Override
                public void run() {
                    try {
                        updatePeerEurekaNodes(resolvePeerUrls());
                    } catch (Throwable e) {
                        logger.error("Cannot update the replica Nodes", e);
                    }

                }
            };
            taskExecutor.scheduleWithFixedDelay(
                    peersUpdateTask,
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    TimeUnit.MILLISECONDS
            );
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
        for (PeerEurekaNode node : peerEurekaNodes) {
            logger.info("Replica node URL:  {}", node.getServiceUrl());
        }
    }
}
```

`DefaultEurekaServerContext`是eureka server的默认上下文，这边就会获取其他的服务节点信息

```java
public class DefaultEurekaServerContext {
    @PostConstruct
    @Override
    public void initialize() {
        logger.info("Initializing ...");
        peerEurekaNodes.start();
        try {
            // 初始化
            registry.init(peerEurekaNodes);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        logger.info("Initialized");
    }
}
```

重点看eureka的初始化过程，在eureka server启动过程中，继承了`smartLifecycle`，这样可以在Bean初始化完做一些事情，
在syncUp方法中，会获取其他eureka server的注册信息，并保存到本地的register中，结构是 `ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>`

```java
public class PeerAwareInstanceRegistryImpl {
    public int syncUp() {
        // Copy entire entry from neighboring DS node
        int count = 0;
        // 重试，循环获取其他server的注册信息
        for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
            if (i > 0) {
                try {
                    Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
                } catch (InterruptedException e) {
                    logger.warn("Interrupted during registry transfer..");
                    break;
                }
            }
            // 获取其他的server注册信息
            Applications apps = eurekaClient.getApplications();
            for (Application app : apps.getRegisteredApplications()) {
                for (InstanceInfo instance : app.getInstances()) {
                    try {
                        if (isRegisterable(instance)) {
                            // 注册到自己的map中，作为本地缓存
                            register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                            count++;
                        }
                    } catch (Throwable t) {
                        logger.error("During DS init copy", t);
                    }
                }
            }
        }
        return count;
    }
}
```

```java
public class PeerAwareInstanceRegistryImpl {
    @Override
    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
        this.expectedNumberOfClientsSendingRenews = count;
        updateRenewsPerMinThreshold();
        logger.info("Got {} instances from neighboring DS node", count);
        logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
        this.startupTime = System.currentTimeMillis();
        if (count > 0) {
            this.peerInstancesTransferEmptyOnStartup = false;
        }
        DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
        boolean isAws = Name.Amazon == selfName;
        if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
            logger.info("Priming AWS connections for all replicas..");
            primeAwsReplicas(applicationInfoManager);
        }
        logger.info("Changing status to UP");
        // 实例状态改为UP
        applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
        // 开启定时任务，每60秒进行服务失败剔除
        super.postInit();
    }
}
```

### 服务注册过程

1. 首先通过Jersey的接口监听注册请求

```java
public class ApplicationResource {
    // 供客户端进行服务注册的接口
    @POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info,
                                @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
        // ...
        // 在这边注册
        registry.register(info, "true".equals(isReplication));
        return Response.status(204).build();  // 204 to be backwards compatible
    }
}
```

2. 调用`register`方法注册到本地缓存

```java
public class PeerAwareInstanceRegistryImpl {
    public void register(final InstanceInfo info, final boolean isReplication) {
        // 服务失效间隔【检查间隔】默认90s
        int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            // 配置文件为准
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }
        super.register(info, leaseDuration, isReplication);
        // 当前server把该注册信息同步到其他节点
        replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
    }
}
```

`super.register`主要是注册节点到本地缓存的逻辑

```java
public class AbstractInstanceRegistry {
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        read.lock();
        try {
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // ...
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            // 设置实例到本地缓存
            gMap.put(registrant.getId(), lease);
            // ...
        } finally {
            read.unlock();
        }
    }
}
```

3. 把该注册信息同步到其他节点

```java
public class A {
    private void replicateToPeers(Action action, String appName, String id,
                                  InstanceInfo info /* optional */,
                                  InstanceStatus newStatus /* optional */, boolean isReplication) {
        Stopwatch tracer = action.getTimer().start();
        try {
            if (isReplication) {
                numberOfReplicationsLastMin.increment();
            }
            // If it is a replication already, do not replicate again as this will create a poison replication
            if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
                return;
            }

            for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
                // If the url represents this host, do not replicate to yourself.
                // 判断是否是当前节点，如果是则不用同步了【之前初始化已经同步过了】
                if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                    continue;
                }
                // 对等节点动作
                replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
            }
        } finally {
            tracer.stop();
        }
    }
    
    private void replicateInstanceActionsToPeers(Action action, String appName,
                                                 String id, InstanceInfo info, InstanceStatus newStatus,
                                                 PeerEurekaNode node) {
        try {
            InstanceInfo infoFromRegistry;
            CurrentRequestVersion.set(Version.V2);
            switch (action) {
                case Cancel:
                    node.cancel(appName, id);
                    break;
                    // 心跳
                case Heartbeat:
                    InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                    infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                    node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                    break;
                    // 注册
                case Register:
                    node.register(info);
                    break;
                case StatusUpdate:
                    infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                    node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                    break;
                case DeleteStatusOverride:
                    infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                    node.deleteStatusOverride(appName, id, infoFromRegistry);
                    break;
            }
        } catch (Throwable t) {
            logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
        } finally {
            CurrentRequestVersion.remove();
        }
    }
}
```

4. 心跳【续约】接口

```java
public class InstanceResource {
    // 续约接口
    public Response renewLease() {
        boolean isFromReplicaNode = "true".equals(isReplication);
        boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);
    }
}
```
这边调用了registry.renew方法
```java
public class PeerAwareInstanceRegistryImpl {
    public boolean renew(final String appName, final String id, final boolean isReplication) {
        // super.renew 里面会调用 --> lastUpdateTimestamp = System.currentTimeMillis() + duration;
        if (super.renew(appName, id, isReplication)) {
            // Action.Heartbeat表示心跳，再一次调用了这个方法
            // renew同步到其他节点
            replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
            return true;
        }
        return false;
    }
}
```

## eureka client
eureka client会注册到server端，并且发送心跳给服务器端。
client在启动的时候需要
1. 读取配置文件
2. 注册到server
3. 开启定时任务发送心跳，**主动拉取**server的注册信息刷新到本地的缓存【eureka不会主动推送】

### 自动装配和注册

1. 自动装配初始化配置类，读取配置文件

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
// eureka.client.enabled = false, 则不会成为客户端
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@ConditionalOnDiscoveryEnabled
@AutoConfigureBefore({ CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
// 配置完毕AutoConfigureAfter再装配EurekaClientAutoConfiguration
@AutoConfigureAfter(name = { "org.springframework.cloud.netflix.eureka.config.DiscoveryClientOptionalArgsConfiguration",
		"org.springframework.cloud.autoconfigure.RefreshAutoConfiguration",
		"org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration",
		"org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration" })
public class EurekaClientAutoConfiguration {
    // 读取配置信息
    @Bean
    @ConditionalOnMissingBean(value = EurekaInstanceConfig.class, search = SearchStrategy.CURRENT)
    public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils,
                                                             ManagementMetadataProvider managementMetadataProvider) {
        String hostname = getProperty("eureka.instance.hostname");
        boolean preferIpAddress = Boolean.parseBoolean(getProperty("eureka.instance.prefer-ip-address"));
        String ipAddress = getProperty("eureka.instance.ip-address");
        boolean isSecurePortEnabled = Boolean.parseBoolean(getProperty("eureka.instance.secure-port-enabled"));

        String serverContextPath = env.getProperty("server.servlet.context-path", "/");
        int serverPort = Integer.parseInt(env.getProperty("server.port", env.getProperty("port", "8080")));

        Integer managementPort = env.getProperty("management.server.port", Integer.class);

        // ...
    }
}
```

2. 在DiscoveryClient的初始化中，获取server的注册信息

```java
public class DiscoveryClient {
    @Inject
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
        // 判断要不要注册
        if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
            
        }
        // 判断要不要获取注册信息
        if (clientConfig.shouldFetchRegistry()) {
            boolean primaryFetchRegistryResult = fetchRegistry(false);
        }
    }
    
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // If the delta is disabled or if it is the first time, get all
            // applications
            // 获取本地缓存
            Applications applications = getApplications();

            // 全量获取
            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
                logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
                logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
                logger.info("Application is null : {}", (applications == null));
                logger.info("Registered Applications size is zero : {}",
                        (applications.getRegisteredApplications().size() == 0));
                logger.info("Application version is -1: {}", (applications.getVersion() == -1));
                getAndStoreFullRegistry();
            } else {
                // 增量获取
                getAndUpdateDelta(applications);
            }
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } catch (Throwable e) {
            logger.info(PREFIX + "{} - was unable to refresh its cache! This periodic background refresh will be retried in {} seconds. status = {} stacktrace = {}",
                    appPathIdentifier, clientConfig.getRegistryFetchIntervalSeconds(), e.getMessage(), ExceptionUtils.getStackTrace(e));
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        // Notify about cache refresh before updating the instance remote status
        onCacheRefreshed();

        // Update remote status based on refreshed data held in the cache
        updateInstanceRemoteStatus();

        // registry was fetched successfully, so return true
        return true;
    }
    
}
```

在全量获取的时候，使用了jersey框架获取了applications信息，然后使用了乐观锁设置到本地缓存

```java
 private void getAndStoreFullRegistry() throws Throwable {
    // AtomicLong 作为乐观锁
    long currentUpdateGeneration = fetchRegistryGeneration.get();

    logger.info("Getting all instance registry info from the eureka server");

    Applications apps = null;
    EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
        // 请求服务器获取
            ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
            : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        apps = httpResponse.getEntity();
    }
    logger.info("The response status is {}", httpResponse.getStatusCode());

    if (apps == null) {
        logger.error("The application is null for some reason. Not storing this information");
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        // 洗牌随机算法设置缓存
        localRegionApps.set(this.filterAndShuffle(apps));
        logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
    } else {
        logger.warn("Not updating applications as another thread is updating it already");
    }
}
```

3. 注册自己

```java
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
        throw e;
    }
    if (logger.isInfoEnabled()) {
        logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
    }
    return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
}
```

4. 定时器获取注册信息和发送心跳

回到`DiscoveryClient`的构造方法，调用完fetchRegistry方法之后会执行`initScheduledTasks`方法，作为定时器的初始化，
定时任务基本和之前初始化获取注册信息`instance`一样

```java
void refreshRegistry() {
    // 重新获取注册信息，和之前的
    boolean success = fetchRegistry(remoteRegionsModified); 
}

boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        // 找不到则被eureka service剔除
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            // 重新注册自己
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```

5. 服务主动下线会调用清理定时任务和server下架操作

```java
void unregister() {
    // It can be null if shouldRegisterWithEureka == false
    if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
        try {
            logger.info("Unregistering ...");
            EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
            logger.info(PREFIX + "{} - deregister  status: {}", appPathIdentifier, httpResponse.getStatusCode());
        } catch (Exception e) {
            logger.error(PREFIX + "{} - de-registration failed{}", appPathIdentifier, e.getMessage(), e);
        }
    }
}
```
