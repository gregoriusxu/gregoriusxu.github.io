---
layout: post
title: 全链路监控框架SkyWalking源代码解读
subtitle: 了解SkyWalking架构设计
author: gregorius
header-img: img/post-bg-universe.jpg
catalog: true
tags: [设计]
---

### 全链路监控，想说爱你并不容易

当系统上线后你没有这样的经历？

- 客户反馈系统慢，你确束手无策
- 系统报出了异常，你确分析不出源头在哪里
- 系统可能面临突发流量，你确不知道是否能支撑得住
- 问题只在正式重现，测试环境缺无法复现
- 每次出现问题无法预警，问题出现问题后事后检讨，而没有事前预警
  ......

如果你有上面的困扰，说明你的系统缺少可观测性，缺少一个全链路监控系统。系统越复杂，我们发现出现问题时越来越无法驾驭，各种问题层出不穷，问题越来越多，为了掌握系统的运行状态，确保系统正常对外提供服务，需要一些手段去监控系统，以了解系统行为，分析系统的性能，或在系统出现故障时，能发现问题、记录问题并发出告警，从而帮助运维人员发现问题、定位问题。也可以根据监控数据发现系统瓶颈，提前感知故障，预判系统负载能力等。

那么我们如何去做？早在2010年谷歌发表了一篇论文《Dapper, a Large-Scale Distributed Systems Tracing Infrastructure》介绍了分布式追踪的概念，之后很多互联网公司都开始根据这篇论文打造自己的分布式链路追踪系统。前面提到的 APM 系统的核心技术就是分布式链路追踪。

为了追踪错综复杂的调用关系，我们需要一些手段去辅助完成，最简单的方式就是画图，如下图的场景

![Cgq2xl5fMbeAIKO2AABvAar2e5E122](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureCgq2xl5fMbeAIKO2AABvAar2e5E122.png)

客户端请求经过负载均衡，负载均衡提供了三种服务

- RPC
- Web服务
- 资源管理
上面的方式虽然直观，但是随着系统的复杂性增加导致我们很清楚的查看调用关系，所以有没有更好的表达方式？

![CgpOIF5fMbeACGYWAABqbqP7vns698](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureCgpOIF5fMbeACGYWAABqbqP7vns698.png)

上面的方式就是 OpenTracing 规范推荐给我们的方案。

其实Tracing只是全链路很小的一部分，我们来看看全貌

![CgpOIF5fMbeAVBbiAAKBOtXrJgg411](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureCgpOIF5fMbeAVBbiAAKBOtXrJgg411.png)

- Logging 就是记录系统行为的离散事件，例如，服务在处理某个请求时打印的错误日志，我们可以将这些日志信息记录到 ElasticSearch 或是其他存储中，然后通过 Kibana 或是其他工具来分析这些日志了解服务的行为和状态。大多数情况下，日志记录的数据很分散，并且相互独立，比如错误日志、请求处理过程中关键步骤的日志等等。

- Metrics 是系统在一段时间内某一方面的某个度量，例如，电商系统在一分钟内的请求次数。我们常见的监控系统中记录的数据都属于这个范畴，例如 Promethus、Open-Falcon 等，这些监控系统最终给运维人员展示的是一张张二维的折线图。Metrics 是可以聚合的，例如，为电商系统中每个 HTTP 接口添加一个计数器，计算每个接口的 QPS，之后我们就可以通过简单的加和计算得到系统的总负载情况。

- Tracing 即我们常说的分布式链路追踪。在微服务架构系统中一个请求会经过很多服务处理，调用链路会非常长，要确定中间哪个服务出现异常是非常麻烦的一件事。通过分布式链路追踪，运维人员就可以构建一个请求的视图，这个视图上展示了一个请求从进入系统开始到返回响应的整个流程。这样，就可以从中了解到所有服务的异常情况、网络调用，以及系统的性能瓶颈等。

更复杂的是我们要分析日志之间的血缘关系，然后根据大数据进行计算和预警，辅助我们判断可能要发生的系统故障事件。

常见的APM系统有

- CAT： 由国内美团点评开源的，基于 Java 语言开发，目前提供 Java、C/C++、Node.js、Python、Go 等语言的客户端，监控数据会全量统计。国内很多公司在用，例如美团点评、携程、拼多多等。CAT 需要开发人员手动在应用程序中埋点，对代码侵入性比较强。

- Zipkin： 由 Twitter 公司开发并开源，Java 语言实现。侵入性相对于 CAT 要低一点，需要对web.xml 等相关配置文件进行修改，但依然对系统有一定的侵入性。Zipkin 可以轻松与 Spring Cloud 进行集成，也是 Spring Cloud 推荐的 APM 系统。

- Pinpoint： 韩国团队开源的 APM 产品，运用了字节码增强技术，只需要在启动时添加启动参数即可实现 APM 功能，对代码无侵入。目前支持 Java 和 PHP 语言，底层采用 HBase 来存储数据，探针收集的数据粒度非常细，但性能损耗较大，因其出现的时间较长，完成度也很高，文档也较为丰富，应用的公司较多。

- SkyWalking： 国人开源的产品，2019 年 4 月 17 日 SkyWalking 从 Apache 基金会的孵化器毕业成为顶级项目。目前 SkyWalking 支持 Java、.Net、Node.js 等探针，数据存储支持MySQL、ElasticSearch等。 SkyWalking 与 Pinpoint 相同，Java 探针采用字节码增强技术实现，对业务代码无侵入。探针采集数据粒度相较于 Pinpoint 来说略粗，但性能表现优秀。目前，SkyWalking 增长势头强劲，社区活跃，中文文档齐全，没有语言障碍，支持多语言探针，这些都是 SkyWalking 的优势所在，还有就是 SkyWalking 支持很多框架，包括很多国产框架，例如，Dubbo、gRPC、SOFARPC 等等，也有很多开发者正在不断向社区提供更多插件以支持更多组件无缝接入 SkyWalking。

还有很多不开源的 APM 系统，例如，淘宝鹰眼、Google Dapper 等等，不再展开介绍了。

在选择一款APM系统对系统的侵入是我们需要首要考虑的问题，其次是性能问题，所以SkyWalking是我们的最常见选择。

那么SkyWalking能为我们做什么？

- 服务、服务实例、端点指标分析。

- 服务拓扑图分析

- 服务、服务实例和端点（Endpoint）SLA 分析

- 慢查询检测

- 告警

SkyWalking的整体架构包括三部分

- Agent（探针）：Agent 运行在各个服务实例中，负责采集服务实例的 Trace 、Metrics 等数据，然后通过 gRPC 方式上报给 SkyWalking 后端。

- OAP：SkyWalking 的后端服务，其主要责任有两个。

  - 一个是负责接收 Agent 上报上来的 Trace、Metrics 等数据，交给 Analysis Core （涉及 SkyWalking OAP 中的多个模块）进行流式分析，最终将分析得到的结果写入持久化存储中。SkyWalking 可以使用 ElasticSearch、H2、MySQL 等作为其持久化存储，一般线上使用 ElasticSearch 集群作为其后端存储。

  - 另一个是负责响应 SkyWalking UI 界面发送来的查询请求，将前面持久化的数据查询出来，组成正确的响应结果返回给 UI 界面进行展示。

- UI 界面：SkyWalking 前后端进行分离，该 UI 界面负责将用户的查询操作封装为 GraphQL 请求提交给 OAP 后端触发后续的查询操作，待拿到查询结果之后会在前端负责展示。

### SkyWalking 源代码解读

SkyWalking使用java agent技术实现，对于agent如何无侵入实现日志记录和监控可以看我之前写得[这篇文章](https://www.jianshu.com/p/b49680cae5f1),了解了基础原理之后我们需要搭建调试环境来学习源代码，可以参考[这篇](https://www.jianshu.com/p/2bc101cce583)进行环境搭建，最后搭建好的项目结构如下图

![skywalkdingdebug](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureskywalkdingdebug.PNG)

上面apm模块为SkyWalking服务端，java-agent为客户端，另外一个是演示项目。

#### 体验微内核之美

可以说SkyWalking的服务端和客户端都是采用微内核设计，客户端的入口是SkyWalkingAgent

##### 客户端代码解读

它其实是一个java agent的入口类，需要实现一个静态的premain方法，如下

``` java

 /**
     * Main entrance. Use byte-buddy transform to enhance all classes, which define in plugins.
     */
    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
        final PluginFinder pluginFinder;
        try {
            SnifferConfigInitializer.initializeCoreConfig(agentArgs);
        } catch (Exception e) {
            // try to resolve a new logger, and use the new logger to write the error log here
            LogManager.getLogger(SkyWalkingAgent.class)
                    .error(e, "SkyWalking agent initialized failure. Shutting down.");
            return;
        } finally {
            // refresh logger again after initialization finishes
            LOGGER = LogManager.getLogger(SkyWalkingAgent.class);
        }

        try {
            pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
        } catch (AgentPackageNotFoundException ape) {
            LOGGER.error(ape, "Locate agent.jar failure. Shutting down.");
            return;
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent initialized failure. Shutting down.");
            return;
        }

        final ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));

        AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy).ignore(
                nameStartsWith("net.bytebuddy.")
                        .or(nameStartsWith("org.slf4j."))
                        .or(nameStartsWith("org.groovy."))
                        .or(nameContains("javassist"))
                        .or(nameContains(".asm."))
                        .or(nameContains(".reflectasm."))
                        .or(nameStartsWith("sun.reflect"))
                        .or(allSkyWalkingAgentExcludeToolkit())
                        .or(ElementMatchers.isSynthetic()));

        JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();
        try {
            agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent inject bootstrap instrumentation failure. Shutting down.");
            return;
        }

        try {
            agentBuilder = JDK9ModuleExporter.openReadEdge(instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            LOGGER.error(e, "SkyWalking agent open read edge in JDK 9+ failure. Shutting down.");
            return;
        }

        if (Config.Agent.IS_CACHE_ENHANCED_CLASS) {
            try {
                agentBuilder = agentBuilder.with(new CacheableTransformerDecorator(Config.Agent.CLASS_CACHE_MODE));
                LOGGER.info("SkyWalking agent class cache [{}] activated.", Config.Agent.CLASS_CACHE_MODE);
            } catch (Exception e) {
                LOGGER.error(e, "SkyWalking agent can't active class cache.");
            }
        }

        agentBuilder.type(pluginFinder.buildMatch())
                    .transform(new Transformer(pluginFinder))
                    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
                    .with(new RedefinitionListener())
                    .with(new Listener())
                    .installOn(instrumentation);

        PluginFinder.pluginInitCompleted();

        try {
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            LOGGER.error(e, "Skywalking agent boot failure.");
        }

        Runtime.getRuntime()
                .addShutdownHook(new Thread(ServiceManager.INSTANCE::shutdown, "skywalking service shutdown thread"));
    }
```

上面代码做了以下几件事件

- SnifferConfigInitializer.initializeCoreConfig 用来加载客户端的配置，这个类里面有个全局静态Map，后面其它地方可以通用它来使用配置
- 初始化PluginFinder，这个是用来精确匹配需要加载的类
- 使用BootstrapInstrumentBoost.inject来动态加载应用程序依赖类，使应用程序无感知，其内部使用的是框架自己扩展的class loader加载类 AgentClassLoader
- ServiceManager.INSTANCE.boot() 最后遍历加载服务类，并启动服务
  
我们来看一下ServiceManager.INSTANCE.boot()方法

``` java

    public void boot() {
        bootedServices = loadAllServices();

        prepare();
        startup();
        onComplete();
    }
```

其实代码很清晰

- loadAllServices 加载所有服务
- prepare做一些服务启动前的准备工作，其实是调用服务实现类的prepare方法
- startup 是调用服务类的boot方法来启动服务
- onComlete是在服务启动后给服务做一些回调

loadAllServices使用的是JDK SPI加载机制来加载服务类，资源文件在resource目录下面，加载的服务有

![service](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureservice.PNG)

我们来看一下最简单的JVMService的实现

![JVMService](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureJVMService.PNG)

可以发现JVMService其实是一个实现了Runnable接口的实现类，它的boot方法其实只是开启了两个定时调度的线程池，真正做事情的是run方法，那么run方法如何调用的呢？

``` java

@Override
    public void boot() throws Throwable {
        collectMetricFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("JVMService-produce"))
                                       .scheduleAtFixedRate(new RunnableWithExceptionProtection(
                                           this,
                                           new RunnableWithExceptionProtection.CallbackWhenException() {
                                               @Override
                                               public void handle(Throwable t) {
                                                   LOGGER.error("JVMService produces metrics failure.", t);
                                               }
                                           }
                                       ), 0, 1, TimeUnit.SECONDS);
        sendMetricFuture = Executors.newSingleThreadScheduledExecutor(
            new DefaultNamedThreadFactory("JVMService-consume"))
                                    .scheduleAtFixedRate(new RunnableWithExceptionProtection(
                                        sender,
                                        new RunnableWithExceptionProtection.CallbackWhenException() {
                                            @Override
                                            public void handle(Throwable t) {
                                                LOGGER.error("JVMService consumes and upload failure.", t);
                                            }
                                        }
                                    ), 0, 1, TimeUnit.SECONDS);
    }
```

我们发现线程池真正调用的是RunnableWithExceptionProtection，传入的第一个参数是this,所以我们猜测就是这个方法调用的run方法，结果确实如此

``` java

public class RunnableWithExceptionProtection implements Runnable {
    private Runnable run;
    private CallbackWhenException callback;

    public RunnableWithExceptionProtection(Runnable run, CallbackWhenException callback) {
        this.run = run;
        this.callback = callback;
    }

    @Override
    public void run() {
        try {
            run.run();
        } catch (Throwable t) {
            callback.handle(t);
        }
    }

    public interface CallbackWhenException {
        void handle(Throwable t);
    }
}
```

##### 服务端代码解读

服务端的启动类是OAPServerStartUp，调用的是OAPServerBootstrap.start方法

``` java

/**
 * Starter core. Load the core configuration file, and initialize the startup sequence through {@link ModuleManager}.
 */
@Slf4j
public class OAPServerBootstrap {
    public static void start() {
        String mode = System.getProperty("mode");
        RunningMode.setMode(mode);

        ApplicationConfigLoader configLoader = new ApplicationConfigLoader();
        ModuleManager manager = new ModuleManager();
        try {
            ApplicationConfiguration applicationConfiguration = configLoader.load();
            manager.init(applicationConfiguration);

            manager.find(TelemetryModule.NAME)
                   .provider()
                   .getService(MetricsCreator.class)
                   .createGauge("uptime", "oap server start up time", MetricsTag.EMPTY_KEY, MetricsTag.EMPTY_VALUE)
                   // Set uptime to second
                   .setValue(System.currentTimeMillis() / 1000d);

            log.info("Version of OAP: {}", Version.CURRENT);

            if (RunningMode.isInitMode()) {
                log.info("OAP starts up in init mode successfully, exit now...");
                System.exit(0);
            }
        } catch (Throwable t) {
            log.error(t.getMessage(), t);
            System.exit(1);
        }
    }
}
```

上面的代码其实做了三件事情

- 使用ApplicationConfigLoader加载服务端配置
- 将请求委托给ModuleManager处理
- 使用MetricsCreator上传服务启动数据

那么ModuleManager又是什么？

OAP 使用 ModuleManager（组件管理器）管理多个 Module（组件），一个 Module 可以对应多个 ModuleProvider（组件服务提供者），ModuleProvider 是 Module 底层真正的实现。
在 OAP 服务启动时，一个 Module 只能选择使用一个 ModuleProvider 对外提供服务。一个 ModuleProvider 可能支撑了一个非常复杂的大功能，在一个 ModuleProvider 中，可以包含多个 Service ，一个 Service 实现了一个 ModuleProvider 中的一部分功能，通过将多个 Service 进行组装集成，可以得到 ModuleProvider 的完整功能。

![OAP](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureOAP.png)

ModuleManager就是微内核的启动管理类，ModuleProvider是对Service进行划分管理，真正的服务是Service,我们来看一下ModuleManager的init方法

``` java

    /**
     * Init the given modules
     */
    public void init(
        ApplicationConfiguration applicationConfiguration) throws ModuleNotFoundException, ProviderNotFoundException, ServiceNotProvidedException, CycleDependencyException, ModuleConfigException, ModuleStartException {
        String[] moduleNames = applicationConfiguration.moduleList();
        ServiceLoader<ModuleDefine> moduleServiceLoader = ServiceLoader.load(ModuleDefine.class);
        ServiceLoader<ModuleProvider> moduleProviderLoader = ServiceLoader.load(ModuleProvider.class);

        HashSet<String> moduleSet = new HashSet<>(Arrays.asList(moduleNames));
        for (ModuleDefine module : moduleServiceLoader) {
            if (moduleSet.contains(module.name())) {
                module.prepare(this, applicationConfiguration.getModuleConfiguration(module.name()), moduleProviderLoader);
                loadedModules.put(module.name(), module);
                moduleSet.remove(module.name());
            }
        }
        // Finish prepare stage
        isInPrepareStage = false;

        if (moduleSet.size() > 0) {
            throw new ModuleNotFoundException(moduleSet.toString() + " missing.");
        }

        BootstrapFlow bootstrapFlow = new BootstrapFlow(loadedModules);

        bootstrapFlow.start(this);
        bootstrapFlow.notifyAfterCompleted();
    }
```

使用模块管理的好处是便于服务管理，同一类服务可以同时启动，服务与服务之间可以使用线程隔离，上面的方法其实只做了两件事情

- 使用SPI加载根据配置加载模块ModuleProvider和Service，将ModuleDefine加到集合里面
- 调用ModuleDefine的prepare方法，其实它本质是调用ModuleProvider的prepare方法
- 将模块委托给BootstrapFlow进行加载启动

我们可以在server-core项目的resource目录下找到定义的模块

``` text
org.apache.skywalking.oap.server.core.storage.StorageModule
org.apache.skywalking.oap.server.core.cluster.ClusterModule
org.apache.skywalking.oap.server.core.CoreModule
org.apache.skywalking.oap.server.core.query.QueryModule
org.apache.skywalking.oap.server.core.alarm.AlarmModule
org.apache.skywalking.oap.server.core.exporter.ExporterModule
```

以及ModuleProvider,其实只有一个

``` text
org.apache.skywalking.oap.server.core.CoreModuleProvider
```

其实是CoreModuleProvider是OAP的真正启动类，这个类异常复杂加载了服务端所有需要注册和启动的服务，这个模块只是核心加载模块，其它模块通过loadedProvider属性持有这个核心加载模块的引用

最后我们来看一下BootstrapFlow的start方法

``` java

    @SuppressWarnings("unchecked")
    void start(
        ModuleManager moduleManager) throws ModuleNotFoundException, ServiceNotProvidedException, ModuleStartException {
        for (ModuleProvider provider : startupSequence) {
            log.info("start the provider {} in {} module.", provider.name(), provider.getModuleName());
            provider.requiredCheck(provider.getModule().services());

            provider.start();
        }
    }
```

其实调用的是就是ModuleProvider的start方法，最后我们看一下coreModuleProvider全貌

![coreModuleProvider](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/picturecoreModuleProvider.PNG)

其中prepare和start方法我们刚才都有提到，其中start方法启动的是RemoteClientManager这个service类，其它方面限于篇幅，有兴趣的读者可以自行研究。
