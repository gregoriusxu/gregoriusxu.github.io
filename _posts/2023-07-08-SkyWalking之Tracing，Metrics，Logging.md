---
layout: post
title: SkyWalking之Tracing，Metrics，Logging
subtitle: 从服务端解读Tracing，Metrics，Logging模块源代码
author: gregorius
header-img: img/post-bg-universe.jpg
catalog: true
tags: [设计]
---

### 从Server端谈Tracing上报

从上篇文章我们知道，Tracing数据是使用TraceSegmentServiceClient通过GRPC发送到服务端，那么服务端是如何接收请求处理的呢?

#### 从Server说起

前面我们说到服务端基于微内核架构初始化了各种Module，每个Module又有很多Service，服务启动时会找到Module进行初始化，初始化模块时先初始化依赖模块再初始化非依赖模块，模块的具体承载对象是ModuleProvider，其中CoreModule顾名思意是核心模块，其对象的Provider是CoreModuleProvider，在之前的文章我们已经介绍过CoreModuleProvider初始化过程，其中其实就包括了Server的初始化过程，其实在Server框架中Server有两种

- GPRCServer 用于接收 SkyWalking Agent 发送的 gRPC 请求。正如前面课时介绍的那样， SkyWalking 6.x 中的 Trace 上报、JVM 监控上报、服务以及服务实例注册请求、心跳请求都是通过 gRPC 请求实现的。
- HTTPServer 用于接收 SkyWalking Agent 以及用户的 Http 请求。

在 GRPCServer 实现中首先会根据配置初始化 NettyServerBuilder 对象（其中会指定服务监听的地址和端口，以及每个连接上的并发请求数等参数），然后创建并启动 io.grpc.Server 接收gRPC 请求。gRPC 的 Java 实现底层是依靠 Netty 实现的，Netty 是一个高性能的开源网络库，由于 Java 本身的 NIO API 使用起来比较麻烦，而 Netty 底层封装了 Java NIO 并对外提供了简单易用的 API，所以很多开源软件的网络模块都是使用 Netty 实现的。

NettyServerBuilder借助于ServerImplBuilder 构建ServerImpl,ServerImpl通过NettyClientTransportServersBuilder构建Server，最终构造出来的Server就是io.grpc.netty.NettyServer，在CoreModuleProvider初始化GRPCServer的代码如下：

``` java

 if (moduleConfig.isGRPCSslEnabled()) {
        grpcServer = new GRPCServer(moduleConfig.getGRPCHost(), moduleConfig.getGRPCPort(),
        moduleConfig.getGRPCSslCertChainPath(),
        moduleConfig.getGRPCSslKeyPath(),
        null
        );
} else {
    grpcServer = new GRPCServer(moduleConfig.getGRPCHost(), moduleConfig.getGRPCPort());
}
```

我们纵观一下CoreModuleProvider的源代码，发现还有几处：

``` java
prepare方法

this.registerServiceImplementation(GRPCHandlerRegister.class, new GRPCHandlerRegisterImpl(grpcServer));


start 方法

grpcServer.addHandler(new RemoteServiceHandler(getManager()));
grpcServer.addHandler(new HealthCheckServiceHandler());

```

GRPCHandlerRegisterImpl 用于实现gRPC的扩展性，和Serverlet的Filter类似，用于权限验证或限流等横向扩展功能的接口注册。

这里简单介绍一下上面start方法里面的两个 GRPCHandler 的功能：

- HealthCheckServiceHandler：在 Cluster 模块中提供了支持多种第三方服务发展组件的实现，前文介绍的 cluster-zookeeper-plugin 模块使用的 curator-x-discovery 扩展库底层是依赖 Zookeeper 的 Watcher 来监听一个 OAP 服务实例是否可用。但是有的第三方服务发现组件（例如 Consul）会依靠定期健康检查（Health Check）来检查一个 OAP 服务实例是否可用，此时 OAP 就需要保留一个接口来处理健康检查请求，这就是 HealthCheckServiceHandler 的核心功能。对 Consul 感兴趣的同学，可以参考下面两篇文档：
<https://www.consul.io/docs/agent/checks.html>
<https://github.com/grpc/grpc/blob/master/doc/health-checking.md>

- RemoteServiceHandler：OAP 集群中各个 OAP 实例节点之间通信的接口，在后面会详细介绍该 GRPCHandler 的实现以及通信方式。

其实真正的上报Tracing数据的处理是在SharingServerModule模块提供的SharingServerModuleProvider处理的，SkyWalking OAP 需要接收外部请求的地方还是挺多的，例如 Agent 上报的监控数据、 SkyWalking Rocketbot 的查询请求、OAP 集群中节点之间的相互通信，等等。除了 CoreModuleProvider 中会启动 Server 组件之外，sharing-server-plugin 模块中也可以启动单独的一套 Server 实例，并监听独立的端口，这套 Server 实例主要用来接收 Agent 的注册请求、上报的 JVM 监控数据以及 Trace 数据。

SharingServerModuleProvider初始化GRPCServer的过程和CoreModuleProvider类似，唯一的区别是SharingServerModuleProvider可以复用CoreModuleProvider创建的GRPCServer。

#### trace-receiver-plugin

trace-receiver-plugin其实是真正处理上报数据的项目，负责的模块是TraceModule，Provider是TraceModuleProvider，TraceModule依赖了5个模块，其中两个分别是CoreModule和SharingServerModule，首先我们看一下start方法的代码：

``` java

@Override
public void start() {
        GRPCHandlerRegister grpcHandlerRegister = getManager().find(SharingServerModule.NAME)
            .provider()
            .getService(GRPCHandlerRegister.class);
        HTTPHandlerRegister httpHandlerRegister = getManager().find(SharingServerModule.NAME)
            .provider()
            .getService(HTTPHandlerRegister.class);

        TraceSegmentReportServiceHandler traceSegmentReportServiceHandler = new TraceSegmentReportServiceHandler(getManager());
        grpcHandlerRegister.addHandler(traceSegmentReportServiceHandler);
        grpcHandlerRegister.addHandler(new TraceSegmentReportServiceHandlerCompat(traceSegmentReportServiceHandler));

        httpHandlerRegister.addHandler(new TraceSegmentReportHandler(getManager()),
        Collections.singletonList(HttpMethod.POST)
    );
}
```

可以看到start方法使用SharingServerModule的GRPCHandlerRegister服务注册了两个自己的处理器分别是TraceSegmentReportServiceHandler，TraceSegmentReportServiceHandlerCompat，TraceSegmentReportServiceHandler 负责接收 Agent 发送来的 SegmentObject 数据，并调用 SegmentParserServiceImpl.send() 方法进行解析，然后再将SegmentObject交给TraceAnalyzer.doAnalysis()方法处理。

``` java

public void doAnalysis(SegmentObject segmentObject) {
    if (segmentObject.getSpansList().size() == 0) {
            return;
    }

    createSpanListeners();

    notifySegmentListener(segmentObject);

    segmentObject.getSpansList().forEach(spanObject -> {
        if (spanObject.getSpanId() == 0) {
            notifyFirstListener(spanObject, segmentObject);
        }

        if (SpanType.Exit.equals(spanObject.getSpanType())) {
            notifyExitListener(spanObject, segmentObject);
        } else if (SpanType.Entry.equals(spanObject.getSpanType())) {
            notifyEntryListener(spanObject, segmentObject);
        } else if (SpanType.Local.equals(spanObject.getSpanType())) {
            notifyLocalListener(spanObject, segmentObject);
        } else {
            log.error("span type value was unexpected, span type name: {}", spanObject.getSpanType()
                                                                                          .name());
        }
    });

    notifyListenerToBuild();
}
```

上面的其实分为三步

- 首先注册监听
- 将数据将给第一步注册的监听处理
- 最后调用AnalysisListener将数据将给聚合流程处理

##### 注册监听

``` java

private void createSpanListeners() {
    listenerManager.getSpanListenerFactories()
        .forEach(
                spanListenerFactory -> analysisListeners.add(
                    spanListenerFactory.create(moduleManager, config)));
}
```

所有监听接口通过SegmentParserListenerManager进行管理，由刚才的代码我们看到监听分为6种

- FirstAnalysisListener
- ExitAnalysisListener
- EntryAnalysisListener
- LocalAnalysisListener
- SegmentListener
- AnalysisListener

刚才我们说到SharingServerModule依赖于5个模块，其中一个就是AnalyzerModule的AnalyzerModuleProvider，那么AnalyzerModuleProvider就是负责SegmentParserListenerManager对象创建和监听器初始化的入口，具体方法是listenerManager

``` java

private SegmentParserListenerManager listenerManager() {
    SegmentParserListenerManager listenerManager = new SegmentParserListenerManager();
    if (moduleConfig.isTraceAnalysis()) {
        listenerManager.add(new RPCAnalysisListener.Factory(getManager()));
        listenerManager.add(new EndpointDepFromCrossThreadAnalysisListener.Factory(getManager()));
        listenerManager.add(new NetworkAddressAliasMappingListener.Factory(getManager()));
    }
    listenerManager.add(new SegmentAnalysisListener.Factory(getManager(), moduleConfig));

    return listenerManager;
}
```

上面注册的监听实例囊括了我们刚才提到的6种监听接口，就是说在极端情况下TraceAnalyzer.doAnalysis()方法会使用上面所有的实例进行数据加工处理，其中RPCAnalysisListener，EndpointDepFromCrossThreadAnalysisListener，NetworkAddressAliasMappingListener为可选流程，通过moduleConfig.isTraceAnalysis()开关开启，限于篇幅，所以下面我们重点介绍SegmentAnalysisListener。

###### FirstAnalysisListener

FirstSpanListener 的 parseFirst() 方法处理 TraceSegment 中的第一个 Span，这里只有 SegmentSpanListener 实现了该方法，具体实现分为三步：

- 检测当前 TraceSegment 是否被成功采样。
- 将 segmentCoreInfo 中记录的 TraceSegment 数据拷贝到 segment 字段中。
- 将 endpointNameId 记录到 firstEndpointId 字段，通过前面的分析我们知道，endpointNameId 在 Spring MVC 里面对应的是 URL，在 Dubbo 里面对应的是 RPC 请求 path 与方法名称的拼接。

上面说的具体实现是在SegmentAnalysisListener的parseFirst方法

``` java

@Override
public void parseFirst(SpanObject span, SegmentObject segmentObject) {
        // 检测是否采样成功
        if (sampleStatus.equals(SAMPLE_STATUS.IGNORE)) {
            return;
        }

        if (StringUtil.isEmpty(serviceId)) {
            serviceName = namingControl.formatServiceName(segmentObject.getService());
            serviceId = IDManager.ServiceID.buildId(
                serviceName,
                true
            );
        }

        long timeBucket = TimeBucket.getRecordTimeBucket(startTimestamp);

        // 将 segmentCoreInfo 中记录的 TraceSegment 数据拷贝到 segment 字段中。
        segment.setSegmentId(segmentObject.getTraceSegmentId());
        segment.setServiceId(serviceId);
        segment.setServiceInstanceId(IDManager.ServiceInstanceID.buildId(
            serviceId,
            namingControl.formatInstanceName(segmentObject.getServiceInstance())
        ));
        segment.setLatency(duration);
        segment.setStartTime(startTimestamp);
        segment.setTimeBucket(timeBucket);
        segment.setIsError(BooleanUtils.booleanToValue(isError));
        segment.setDataBinary(segmentObject.toByteArray());

        // 将 endpointNameId 记录到 firstEndpointId 字段，通过前面的分析我们知道，/endpointNameId 在 Spring MVC 里面对应的是 URL，在 Dubbo 里面对应的是 RPC 请求 path 与方法名称的拼接。
        endpointName = namingControl.formatEndpointName(serviceName, span.getOperationName());
        endpointId = IDManager.EndpointID.buildId(
            serviceId,
            endpointName
        );
}
```

###### EntryAnalysisListener

EntrySpanListener 实现的 parseEntry() 方法对于 Entry 类型的 Span 进行处理。SegmentAnalysisListener.parseEntry() 方法只做了一件事，就是将 serviceName 记录到 endpointId 字段中：

``` java

@Override
public void parseEntry(SpanObject span, SegmentObject segmentObject) {
        if (StringUtil.isEmpty(serviceId)) {
            serviceName = namingControl.formatServiceName(segmentObject.getService());
            serviceId = IDManager.ServiceID.buildId(
                serviceName, true);
        }

        endpointName = namingControl.formatEndpointName(serviceName, span.getOperationName());
        endpointId = IDManager.EndpointID.buildId(
            serviceId,
            endpointName
        );
}
```

###### LocalAnalysisListener

主要是为了解决localspan和exitspan丢失元数据的问题，SegmentAnalysisListener并不会实现LocalAnalysisListener，但是RPCAnalysisListener和EndpointDepFromCrossThreadAnalysisListener确有实现，拿EndpointDepFromCrossThreadAnalysisListener为例，其主要的实现是将SegmentObject里面的SpanObject信息缓存到RPCTrafficSourceBuilder对象，然后在build环节交给聚合流程处理，限于篇幅，这里不详细介绍。

###### ExitAnalysisListener

同样SegmentAnalysisListener并不会实现ExitAnalysisListener，但是RPCAnalysisListener和EndpointDepFromCrossThreadAnalysisListener确有实现，以RPCAnalysisListener为例，其主要的实现是将将SegmentObject里面的SpanObject信息缓存到List\<RPCTrafficSourceBuilder> callingOutTraffic,List\<DatabaseSlowStatementBuilder> dbSlowStatementBuilders两个变量，然后在build环节交给聚合流程处理，限于篇幅，这里不详细介绍。

###### SegmentListener

主要实现在SegmentAnalysisListener的parseSegment方法，主要用来给segment对象设置TraceId，根据Span的起始时间设置抽样状态sampleStatus

###### AnalysisListener

SegmentListener继承了AnalysisListener，调用的是build方法

``` java

  @Override
public void build() {
    if (sampleStatus.equals(SAMPLE_STATUS.IGNORE)) {
        if (log.isDebugEnabled()) {
            log.debug("segment ignored, trace id: {}", segment.getTraceId());
            }
            return;
        }

    if (log.isDebugEnabled()) {
        log.debug("segment listener build, segment id: {}", segment.getSegmentId());
    }

    segment.setEndpointId(endpointId);

    sourceReceiver.receive(segment);
    addAutocompleteTags();
}
```

上面的方法其实就是做了一件事件，将segment对象的信息，交给sourceReceiver处理。

###### sourceReceiver

sourceReceiver来自于CoreModule，我们在CoreModuleProvider的构造方法可以发现其实它是一个SourceReceiverImpl实例，SourceReceiver 底层封装的 DispatcherManager  会根据 Segment 选择相应的 SourceDispatcher 实现 —— SegmentDispatcher 进行分发，DispatcherManager的scan方法会扫描所有SourceDispatcher接口实现类，并将其泛型类型的scope方法返回的内容即DefaultScopeDefine.SEGMENT作为KEY缓存到一个Map对象dispatcherMap,forward方法会根据传入实例的类型的scope方法从dispatcherMap拿到SourceDispatcher即SegmentDispatcher调用其dispatch处理。

``` java

public void forward(ISource source) {
        if (source == null) {
            return;
        }

        List<SourceDispatcher> dispatchers = dispatcherMap.get(source.scope());

        /**
         * Dispatcher is only generated by oal script analysis result.
         * So these will/could be possible, the given source doesn't have the dispatcher,
         * when the receiver is open, and oal script doesn't ask for analysis.
         */
        if (dispatchers != null) {
            source.prepare();
            for (SourceDispatcher dispatcher : dispatchers) {
                dispatcher.dispatch(source);
            }
        }
}
```

SegmentDispatcher.dispatch() 方法中会将 Segment 中的数据拷贝到 SegmentRecord 对象中。

SegmentRecord 继承了 StorageData 接口，与前面介绍的 RegisterSource 以及 Metrics 的实现类似，通过注解指明了 Trace 数据存储的 index 名称的前缀（最终写入的 index 是由该前缀以及 TimeBucket 后缀两部分共同构成）以及各个字段对应的 field 名称，如下所示：

``` java

// @Stream 注解的 name 属性指定了 index 的名称(index 前缀)，processor 指定了处理该类型数据的 StreamProcessor 实现

@SuperDataset
@Stream(name = SegmentRecord.INDEX_NAME, scopeId = DefaultScopeDefine.SEGMENT, builder = SegmentRecord.Builder.class, processor = RecordStreamProcessor.class)
public class SegmentRecord extends Record {

    public static final String INDEX_NAME = "segment";
    public static final String SEGMENT_ID = "segment_id";
    public static final String TRACE_ID = "trace_id";
    public static final String SERVICE_ID = "service_id";
    public static final String SERVICE_INSTANCE_ID = "service_instance_id";
    public static final String ENDPOINT_ID = "endpoint_id";
    public static final String START_TIME = "start_time";
    public static final String LATENCY = "latency";
    public static final String IS_ERROR = "is_error";
    public static final String DATA_BINARY = "data_binary";
    public static final String TAGS = "tags";

    @Setter
    @Getter
    @Column(columnName = SEGMENT_ID, length = 150)
    private String segmentId;
    @Setter
    @Getter
    @Column(columnName = TRACE_ID, length = 150)
    @BanyanDBGlobalIndex(extraFields = {})
    private String traceId;
    @Setter
    @Getter
    @Column(columnName = SERVICE_ID)
    @BanyanDBShardingKey(index = 0)
    private String serviceId;
    @Setter
    @Getter
    @Column(columnName = SERVICE_INSTANCE_ID)
    @BanyanDBShardingKey(index = 1)
    private String serviceInstanceId;
    @Setter
    @Getter
    @Column(columnName = ENDPOINT_ID)
    private String endpointId;
    @Setter
    @Getter
    @Column(columnName = START_TIME)
    private long startTime;
    @Setter
    @Getter
    @Column(columnName = LATENCY)
    private int latency;
    @Setter
    @Getter
    @Column(columnName = IS_ERROR)
    @BanyanDBShardingKey(index = 2)
    private int isError;
    @Setter
    @Getter
    @Column(columnName = DATA_BINARY, storageOnly = true)
    private byte[] dataBinary;
    @Setter
    @Getter
    @Column(columnName = TAGS, indexOnly = true)
    private List<String> tags;

    // ...其它代码
```

在 SegmentRecord 的父类 —— Record 中还定义了一个 timeBucket 字段（long 类型），对应的 field 名称是 "time_bucket"。

RecordStreamProcessor 的核心功能是为每个 Record 类型创建相应的 worker 链。在 RecordStreamProcessor 中，每个 Record 类型对应的 worker 链中只有一个worker 实例 —— RecordPersistentWorker。RecordPersistentWorker 负责 SegmentRecord 数据的持久化。

``` java
@Override
public void in(Record record) {
    try {
        InsertRequest insertRequest = recordDAO.prepareBatchInsert(model, record);
        batchDAO.insert(insertRequest);
    } catch (IOException e) {
        LOGGER.error(e.getMessage(), e);
    }
}
```

recordDAO由StorageDAO创建，StorageDAO由StorageModule初始化，StorageDAO对应的实例是 StorageEsDAO，创建的batchDAO为RecordEsDAO，batchDAO其实对应的是BatchProcessEsDAO，最后RecordPersistentWorker 生成的全部 IndexRequest 请求会交给全局唯一的 BatchProcessEsDAO 实例批量发送到 ES ，完成写入。

### Metrics

有了上面的基础知道，我们来研究一下聚合到底是怎么做的。上面我们提到在RPCAnalysisListener的parseExit方法会将Tracing上报数据缓存到dbSlowStatementBuilders集合，这个集合就是用来收集上报数据中存储的慢查询数据用的。收集到数据之后在build方法将数据交给sourceReceiver处理，sourceReceiver接收的其实是一个DatabaseSlowStatement对象，所以其对应的处理实现类是DatabaseStatementDispatcher，它的dispatch方法其实就是做了聚合的事情，因为对于慢查询来说，我们一般都会去关注前N个慢查询信息，

``` java

@Override
public void dispatch(DatabaseSlowStatement source) {
    TopNDatabaseStatement statement = new TopNDatabaseStatement();
    statement.setId(source.getId());
    statement.setServiceId(source.getDatabaseServiceId());
    statement.setLatency(source.getLatency());
    statement.setStatement(source.getStatement());
    statement.setTimeBucket(source.getTimeBucket());
    statement.setTraceId(source.getTraceId());

    TopNStreamProcessor.getInstance().in(statement);
}
```

TopNStreamProcessor 为每个 TopN 类型（其实只有 TopNDatabaseStatement）提供的 Worker 链中只有一个 Worker —— TopNWorker。与前文介绍的 MetricsPersistentWorker 以及 RecordPersistentWorker 类似，TopNWorker 也继承了 PersistenceWorker 抽象类，其结构如下图所示，TopNWorker 也是先将 TopNDatabaseStatement 暂存到 DataCarrier，然后由后台 Consumer 线程定期读取并调用 onWork() 方法进行处理。

在 TopNWorker.onWorker() 方法中会将 TopNDatabaseStatement 暂存到 LimitedSizeBufferedData 中进行排序。LimitedSizeBufferedData 使用双队列模式，继承了 BufferedData 中进行排序。LimitedSizeBufferedData 底层的队列实现是 HashMap嵌套LinkedList数组HashMap<String, LinkedList\<STORAGE_DATA>>，其 data 字段（Map 类型）中维护了每个存储服务的慢查询（即 TopNDatabaseStatement）列表，每个列表都是定长的（由 limitedSize 字段指定，默认 50），在调用 ReadWriteSafeCache.write() 方法写入的时候会按照 latency 从大到小排列，并只保留最多 50 个元素。

那么数据是如何写入的？

#### PersistenceTimer

在CoreModuleProvider的notifyAfterCompleted方法会启动一个PersistenceTimer实例并调用其start 方法PersistenceTimer.INSTANCE.start(getManager(), moduleConfig)处理PersistenceWorker的入库，在它的extractDataAndSave方法主要处理两类PersistenceWorker

- TopNStreamProcessor
- MetricsStreamProcessor

调用PersistenceWorker的buildBatchRequests拿到List\<PrepareRequest>之后再调用batchDAO进行入库操作，batchDAO就是IBatchDAO，我们之前有介绍，这里不再赘述。

#### MetricsStreamProcessor

MetricsStreamProcessor相比TopNStreamProcessor更为复杂的聚合，主要是解决度量数据的按时间维度的精度模糊聚合，举个例子，JVM数据一般按年月日时分秒进行记录，我们需要按小时或者分钟的曲线图，所以需要将低级单位的数据向高级单位进行聚合，这样不但可以节省存储，也可以提高查询性能，因为监控数据不太需要太精确，只需要看到一个发展的趋势就可以了，所以我们可以采用聚合的方式对数据进行模糊化处理，我们把这个过程叫做Downsampling。

MetricsStreamProcessor 中为每个 Metrics 类型维护了一个 Worker 链，

``` java

private Map<Class<? extends Metrics>, MetricsAggregateWorker> entryWorkers = new HashMap<>();
```

MetricsStreamProcessor 初始化 entryWorkers 集合的核心逻辑也是在 create() 方法中，具体方法如下：

``` java

public void create(ModuleDefineHolder moduleDefineHolder,
                       StreamDefinition stream,
                       Class<? extends Metrics> metricsClass) throws StorageException {
        final StorageBuilderFactory storageBuilderFactory = moduleDefineHolder.find(StorageModule.NAME)
        .provider()
        .getService(StorageBuilderFactory.class);
        final Class<? extends StorageBuilder> builder = storageBuilderFactory.builderOf(
            metricsClass, stream.getBuilder());

        StorageDAO storageDAO = moduleDefineHolder.find(StorageModule.NAME).provider().getService(StorageDAO.class);
        IMetricsDAO metricsDAO;
        try {
            metricsDAO = storageDAO.newMetricsDao(builder.getDeclaredConstructor().newInstance());
        } catch (InstantiationException | IllegalAccessException | NoSuchMethodException | InvocationTargetException e) {
            throw new UnexpectedException("Create " + stream.getBuilder().getSimpleName() + " metrics DAO failure.", e);
        }

        ModelCreator modelSetter = moduleDefineHolder.find(CoreModule.NAME).provider().getService(ModelCreator.class);
        DownSamplingConfigService configService = moduleDefineHolder.find(CoreModule.NAME)
        .provider()
        .getService(DownSamplingConfigService.class);

        MetricsPersistentWorker hourPersistentWorker = null;
        MetricsPersistentWorker dayPersistentWorker = null;

        MetricsTransWorker transWorker = null;

        final MetricsExtension metricsExtension = metricsClass.getAnnotation(MetricsExtension.class);
        /**
         * All metrics default are `supportDownSampling` and `insertAndUpdate`, unless it has explicit definition.
         */
        boolean supportDownSampling = true;
        boolean supportUpdate = true;
        boolean timeRelativeID = true;
        if (metricsExtension != null) {
            supportDownSampling = metricsExtension.supportDownSampling();
            supportUpdate = metricsExtension.supportUpdate();
            timeRelativeID = metricsExtension.timeRelativeID();
        }
        if (supportDownSampling) {
            if (configService.shouldToHour()) {
                Model model = modelSetter.add(
                    metricsClass, stream.getScopeId(), new Storage(stream.getName(), timeRelativeID, DownSampling.Hour),
                    false
                );
                hourPersistentWorker = downSamplingWorker(moduleDefineHolder, metricsDAO, model, supportUpdate);
            }
            if (configService.shouldToDay()) {
                Model model = modelSetter.add(
                    metricsClass, stream.getScopeId(), new Storage(stream.getName(), timeRelativeID, DownSampling.Day),
                    false
                );
                dayPersistentWorker = downSamplingWorker(moduleDefineHolder, metricsDAO, model, supportUpdate);
            }

            transWorker = new MetricsTransWorker(
                moduleDefineHolder, hourPersistentWorker, dayPersistentWorker);
        }

        Model model = modelSetter.add(
            metricsClass, stream.getScopeId(), new Storage(stream.getName(), timeRelativeID, DownSampling.Minute),
            false
        );
        MetricsPersistentWorker minutePersistentWorker = minutePersistentWorker(
            moduleDefineHolder, metricsDAO, model, transWorker, supportUpdate);

        String remoteReceiverWorkerName = stream.getName() + "_rec";
        IWorkerInstanceSetter workerInstanceSetter = moduleDefineHolder.find(CoreModule.NAME)
        .provider()
        .getService(IWorkerInstanceSetter.class);
        workerInstanceSetter.put(remoteReceiverWorkerName, minutePersistentWorker, metricsClass);

        MetricsRemoteWorker remoteWorker = new MetricsRemoteWorker(moduleDefineHolder, remoteReceiverWorkerName);
        MetricsAggregateWorker aggregateWorker = new MetricsAggregateWorker(
            moduleDefineHolder, remoteWorker, stream.getName(), l1FlushPeriod);

        entryWorkers.put(metricsClass, aggregateWorker);
}
```

其中 minutePersistentWorker 是一定会存在的，其他 DownSampling（Hour、Day、Month） 对应的 PersistentWorker 则会根据配置的创建并添加，在 CoreModuleProvider.prepare() 方法中有下面这行代码，会获取 Downsampling 配置并保存于 DownsamplingConfigService 对象中配置。后续创建上述 Worker 时，会从中获取配置的 DownSampling。

``` java

 this.registerServiceImplementation(
            DownSamplingConfigService.class, new DownSamplingConfigService(moduleConfig.getDownsampling()));
```

##### MetricsAggregateWorker

MetricsAggregateWorker为聚合入口的第一个Worker,其功能就是进行简单的聚合,MetricsAggregateWorker 在收到 Metrics 数据的时候，会先写到内部的 DataCarrier 中缓存，然后由 Consumer 线程（都属于名为 “METRICS_L1_AGGREGATION” 的 BulkConsumePool）消费并进行聚合，并将聚合结果写入到 MergeDataCache 中的 current 队列暂存。

同时，Consumer 会定期（默认1秒，通过 METRICS_L1_AGGREGATION_SEND_CYCLE 配置修改）触发 current 队列和 last 队列的切换，然后读取 last 队列中暂存的数据，并发送到下一个 Worker 中处理。

写入 DataCarrier 的逻辑在前面已经分析过了，这里不再赘述。下面深入分析两个点：

- Consumer 线程消费 DataCarrier 并聚合监控数据的相关实现。
- Consumer 线程定期清理 MergeDataCache 缓冲区并发送监控数据的相关实现。

Consumer 线程在消费 DataCarrier 数据的时候，首先会进行 Metrics 聚合（即相同 Metrics 合并成一个），然后写入 MergeDataCache 中，实现如下：

``` java

    /**
     * Dequeue consuming. According to {@link IConsumer#consume(List)}, this is a serial operation for every work
     * instance.
     *
     * @param metricsList from the queue.
     */
    private void onWork(List<Metrics> metricsList) {
        metricsList.forEach(metrics -> {
            aggregationCounter.inc();
            mergeDataCache.accept(metrics);
        });

        flush();
    }
```

将聚合后的 Metrics 写入 MergeDataCache 之后，Consumer 线程会每隔一秒将 MergeDataCache 中的数据发送到下一个 Worker 处理，相关实现如下：

``` java

    private void flush() {
        long currentTime = System.currentTimeMillis();
        if (currentTime - lastSendTime > l1FlushPeriod) {
            mergeDataCache.read().forEach(
                data -> {
                    if (log.isDebugEnabled()) {
                        log.debug(data.toString());
                    }
                    nextWorker.in(data);
                }
            );
            lastSendTime = currentTime;
        }
    }
```

##### MetricsRemoteWorker

MetricsAggregateWorker 指向的下一个 Worker 是 MetricsRemoteWorker ，底层是通过 RemoteSenderService 将监控数据发送到远端节点。

它提供了三种不同的发送策略：

- HashCode 策略：根据 Hash 值选择发送到目标 OAP 节点。MetricsRemoteWorker 默认使用该策略。
- Rolling 策略：轮训方式选择目标 OAP 节点。
- ForeverFirst 策略：始终选择第一个 OAP 节点作为目标节点。RegisterRemoteWorker 默认使用该策略。

``` java

    @Override
    public final void in(Metrics metrics) {
        try {
            remoteSender.send(remoteReceiverWorkerName, metrics, Selector.HashCode);
        } catch (Throwable e) {
            log.error(e.getMessage(), e);
        }
    }
```

为什么要发到其他 OAP 节点进行处理呢？

在 CoreModuleProvider 启动过程中，我们可以看到 OAP 节点的角色选择逻辑，如下所示：

``` java

if (Mixed.name().equalsIgnoreCase(moduleConfig.getRole()) || 

      Aggregator.name().equalsIgnoreCase(moduleConfig.getRole())) {

    RemoteInstance gRPCServerInstance = new RemoteInstance(

        new Address(moduleConfig.getGRPCHost(), 

            moduleConfig.getGRPCPort(), true));

    // 只有Mixed、Aggregator两种角色的OAP节点才会通过Cluster模块进行注册

    this.getManager().find(ClusterModule.NAME).provider()

       .getService(ClusterRegister.class)

           .registerRemote(gRPCServerInstance);

}

```

在 application.yml 配置文件中可以配置 Mixed、Receiver、Aggregator 三种角色：

- Receiver 节点：负责接收 Agent 请求并进行 L1 级别的聚合处理，后续的 L2 级别的聚合操作由其他两种类型的节点处理。
- Mixed 节点：负责接收 Agent 请求以及其他 OAP 节点 L1 聚合结果，进行 L1 级别和 L2 级别的聚合处理。
- Aggregator 节点和：负责接收其他 OAP节点的 L1 聚合结果，进行 L2 级别的聚合处理。
  
正是因为集群环境中不同的服务有不同的角色，不同步角色做的聚合是不一样的，所以进行请求转发处理。

##### MetricsTransWorker

MetricsRemoteWorker 之后的下一个 worker 是 MetricsTransWorker，其中有两个字段分别指向两个不同 Downsampling 粒度的 PersistenceWorker 对象，如下：

``` java

   private final MetricsPersistentWorker hourPersistenceWorker;
   private final MetricsPersistentWorker dayPersistenceWorker;

```

MetricsTransWorker.in() 方法会根据上述字段是否为空，将 Metrics 数据分别转发到不同的 PersistenceWorker 中进行处理：

``` java

public void in(Metrics metrics) {

    // 检测 Hour、Day、Month 对应的 PersistenceWorker 是否为空，若不为空，

    // 则将 Metrics 数据拷贝一份并调整时间窗口粒度，交到相应的 

    // PersistenceWorker 处理，这里省略了具体逻辑

    // 最后，直接转发给 minutePersistenceWorker 进行处理

    if (Objects.nonNull(minutePersistenceWorker)) { 

        aggregationMinCounter.inc();

        minutePersistenceWorker.in(metrics);

    }

}
```

##### MetricsPersistentWorker

这个之前已经描述过，这里不再赘述

### Logging

SkyWalking OAP 底层支持 ElasticSearch、H2、MySQL 等多种持久化存储，同时也支持读取其他分布式链路追踪系统的数据，例如，jaeger（Uber 开源的分布式跟踪系统）、zipkin（Twitter 开源的分布式跟踪系统）。

首先，OAP 存储了两种类型的数据：时间相关的数据和非时间相关的数据（与“时序”这个专有名词区分一下）。注册到 OAP 集群的 Service、ServiceInstance 以及同步的 EndpointName、NetworkAddress 都是非时间相关的数据，一个稳定的服务产生的这些数据是有限的，我们可以用几个固定的 ES 索引（或数据库表）来存储这些数据。

而像 JVM 监控等都属于时间相关的数据，它们的数据量会随时间流逝而线性增加。如果将这些时间相关的数据存储到几个固定的 ES 索引中，就会导致这些 ES 索引（或数据库表）非常大，这种显然是不能落地的。既然一个 ES 索引（或数据库表）存不下，一般会考虑切分，常见的切分方式按照时间窗口以及 DownSampling 进行切分。

#### Model 与 ES 索引

常见的 ORM 框架会将数据中的表结构映射成 Java 类，表中的一个行数据会映射成一个 Java Bean 对象，在 OAP 的存储抽象中也有类似的操作。SkyWalking 会将 [Metric + DownSampling] 对应的一系列 ES 索引与一个 Model 对象进行映射。

Model 对象记录了对应 ES 索引的核心信息：

- name（String类型）：Metric 名称，在 OAP 创建 ES 索引时会使用
- columns（List类型）：ES 索引 中的 Field 集合，一个 ModelColumn 对象记录了一个 Field 的名称、类型等信息。
- capableOfTimeSeries（boolean类型）：对应 ES 索引中存储的数据是否为时间相关的数据。
- downsampling（Downsampling类型）：如果是时间相关的数据，则需要指定其 Downsampling 单位，可选值有 Second、 Minute、 Hour、 Day、 Month。对于非时间相关的数据，则该字段值为 Downsampling.NONE。
- deleteHistory（boolean类型）：是否删除历史数据。
- scopeId（int类型）：对应指标的全局唯一 ID。
很明显，ES 索引的名称由三部分构成：Metric 名称、DownSampling、时间窗口（后面两部分只有时序数据才会有），而 ES 索引的别名由 Metric 名称和 DownSampling 构成。

在 CoreModuleProvider 启动过程中，会扫描 @Stream 注解，创建相应的 Model 对象，并由 StorageModels 统一管理（底层维护了 List 集合）。  @Stream 中会包含 Model 需要的信息即可。StorageModels 同时实现了 IModelManager、ModelCreator、ModelManipulator 三个 Service 接口,在 CoreModuleProvider 的 prepare() 方法中会将 StorageModels 作为上述三个 Service 接口的实现注册到 services 集合中。如果回看一下上面的代码你就会发现在org.MetricsStreamProcessor.create从CoreModuleProvider获取ModelCreator进行注册。

``` java
//...
ModelCreator modelSetter = moduleDefineHolder.find(CoreModule.NAME).provider().getService(ModelCreator.class);

//...
if (configService.shouldToHour()) {
    Model model = modelSetter.add(
                    metricsClass, stream.getScopeId(), new Storage(stream.getName(), timeRelativeID, DownSampling.Hour),
                    false
                );
    hourPersistentWorker = downSamplingWorker(moduleDefineHolder, metricsDAO, model, supportUpdate);
}
//...
```

prepare 方法先注册监听

``` java

annotationScan.registerListener(new StreamAnnotationListener(getManager()));

```

start 方法启动@Stream 注解的扫描

``` java
// ...
annotationScan.scan();

// ...
```

下面介绍一下Model的初始化过程，主要还是调用StorageModels的add方法进行注册，我们分析一下metricsClass来自于哪里

- AnalyzerModuleProvider 的start方法中会调用MeterProcessService的start方法
- MeterProcessService会初始化一个MetricConvert

``` java
      public void start(List<MeterConfig> configs) {
        final MeterSystem meterSystem = manager.find(CoreModule.NAME).provider().getService(MeterSystem.class);
        this.metricConverts = configs.stream().map(c -> new MetricConvert(c, meterSystem)).collect(Collectors.toList());
    }

```

- MetricConvert会调用Analyzer的build方法

``` java

  public MetricConvert(MetricRuleConfig rule, MeterSystem service) {
        Preconditions.checkState(!Strings.isNullOrEmpty(rule.getMetricPrefix()));
        this.analyzers = rule.getMetricsRules().stream().map(
            r -> Analyzer.build(
                formatMetricName(rule, r.getName()),
                rule.getFilter(),
                Strings.isNullOrEmpty(rule.getExpSuffix()) ?
                    r.getExp() : String.format("(%s).%s", r.getExp(), rule.getExpSuffix()),
                service
            )
        ).collect(toList());
    }
```

- Analyzer 会MeterSystem的create来初始化MetricsStreamProcessor，并调用MetricsStreamProcessor的create方法。

#### 初始化存储结构

很多 ORM 框架（例如，Hibernate ）提供了在应用启动时根据 Java Bean 初始化表结构的功能。SkyWalking OAP 也提供了类似的功能，主要在 ModelInstaller 中完成，核心在 whenCreating() 方法中：

``` java

@Override
public void whenCreating(Model model) throws StorageException {
        if (RunningMode.isNoInitMode()) {
            while (!isExists(model)) {
                try {
                    log.info(
                        "table: {} does not exist. OAP is running in 'no-init' mode, waiting... retry 3s later.",
                        model
                            .getName()
                    );
                    Thread.sleep(3000L);
                } catch (InterruptedException e) {
                    log.error(e.getMessage());
                }
            }
        } else {
            if (!isExists(model)) {
                log.info("table: {} does not exist", model.getName());
                createTable(model);
            }
        }
}
```

在StorageModels的add方法中会调用List\<CreatingListener> listeners来遍历具体实现类来访问whenCreating 来创建表结构，这个CreatingListener接口的唯一实现是ModelInstaller，峰回路转，你有没有发现所有流程已经串接起来，也就是AnalyzerModuleProvider初始化model的时候同时创建了表结构。

createTable方法是个虚方法，主要有两个实现

- StorageEsInstaller
- H2TableInstaller

我们主要关注StorageEsInstaller

``` java

@Override
protected void createTable(Model model) throws StorageException {
        if (model.isTimeSeries()) {
            createTimeSeriesTable(model);
        } else {
            createNormalTable(model);
        }
}
```

代码很清晰就是创建我们介绍两种表

#### 数据抽象

了解了索引（或是数据库表结构）的初始化流程之后，再来看 SkyWalking OAP是如何对持久化数据进行抽象的。

SkyWalking 与 ORM 类似，会将索引中的一个 Document （或是数据库表中的一条数据）映射为一个 StorageData 对象。StorageData 接口中定义了一个 id() 方法，负责返回该条数据的唯一标识。我们上面的就是TopN就是一个StorageData 对象实现。

- Metrics：所有监控指标的顶级抽象，其中定义了一个 timeBucket 字段（long类型），它是所有监控指标的公共字段，用于表示该监控点所处的时间窗口。另外，timeBucket 字段被 @Column 注解标记，在 OAP 启动时会被扫描到转换成 Model 中的 ModelColumn，在初始化 ES 索引时就会创建相应的 Field。Metrics 抽象类中还定义了计算监控数据的一些公共方法：

  - calculate() 方法：大部分 Metrics 都会有一个 value 字段来记录该监控点的值，例如，CountMetrics 中的 value 字段记录了时间窗口内事件的次数，MaxLongMetrics 中的 value 字段记录时间窗口内的最大值。calculate() 方法就是用来计算该 value 值。

  - combine() 方法：合并两个监控点。对于不同含义的监控数据，合并方式也有所不同，例如，CountMetrics 中 combine() 方法的实现是将两个监控点的值进行求和；MaxLongMetrics 中 combine() 方法的实现是取两个监控点的最大值。

- Record：抽象了所有记录类型的数据，其子类如下图所示。其中 SegmentRecord 对应的是 TraceSegment 数据、AlarmRecord 对应一条告警、TopNDatabaseStatement 对应一条慢查询记录，这些数据都是一条条的记录。

上述记录类型的数据也有一个公共字段 —— timeBucket（long 类型），表示该条记录数据所在的时间窗口，也同样被 @Column 注解标记。Record 抽象类中没有定义其他的公共方法。

RegisterSource：抽象了服务注册、服务实例注册、EndpointName（以及 NetworkAddress）同步三个过程中产生的数据。其中，定义了三个公共字段，且这三个字段都被 @Column 注解标注了：
sequence（int 类型）：上述三个过程中的数据都会产生一个全局唯一的 ID，该全局 ID 就保存在该字段中。
registerTime（long 类型）：第一注册（或同步）时的时间戳。
heartbeatTime（long 类型）：心跳时间戳。
在这三个抽象类下还有很多具体的实现类，这些实现类会根据对应的具体数据，扩展新的字段和方法

最后，我们来看 StorageBuilder 这个接口，它与 StorageData 接口的关系非常紧密，在 StorageData 的全部实现类中，都有一个内部 Builder 类实现类了 StorageBuilder 接口。StorageBuilder 接口中定义了 map2Data() 和 data2Map() 两个方法，实现了 StorageData 对象与 Map 之间的相互转换。

#### DAO 层架构

在创建 ES 索引时使用到的 ElasticSearchClient， 是对 RestHighLevelClient 进行了一层封装，位于 library-client 模块，其中还提供了 JDBC 以及 gRPC 的 Client

SkyWalking OAP 也提供了DAO 层抽象，如下图所示，大致可以分为三类：Receiver DAO、Cache DAO、Query DAO。

#### 数据 TTL

Metrics、Trace 等（时间相关的数据）对应的 ES 索引都是按照时间进行切分的，随着时间的推移，ES 索引会越来越多。为了解决这个问题，SkyWalking 只会在 ES 中存储一段时间内的数据，CoreModuleProvider 会启动 DataTTLKeeperTimer 定时清理过期数据。
