---
layout: post
title: SkyWalking之Tracing
subtitle: 从客户端和服务端解读Tracing模块源代码
author: gregorius
header-img: img/post-bg-universe.jpg
catalog: true
tags: [设计]
---

[上文](https://www.jianshu.com/p/7e1229b8cae1)我们说到作为一个全链路监控系统，Tracing是必不可少的部分，今天我们从客户端和服务端的源代码角度分析一下SkyWalking如何实现这块的。

### Agent 代理原理

在介绍Traceing真正的实现之前，我们有必要知道SkyWalking是如何帮助我们完成agent代理的，从上面的文章我们知道，agent的入口是SkyWalkingAgent，我们之前提到了PluginFinder这个加载类，其实它是用来加载agent代理类的总管。如果需要给不同的框架或者代码增加埋点入口，我们需要对框架的代码或者原理非常了解才行，这样才能最大限度抓取到我们需要的trace信息，举例来说，如果我们需要抓取慢SQL，你使用的是mysql驱动，那么需要在创建preparestatement或者statement的时候最好，SkyWalking为我们准备CreatePreparedStatementInterceptor和CreateStatementInterceptor两个解析类，用来拦截ConnectionImpl 的prepareStatement和createStatement方法。

- 这两个解析类的基类是InstanceMethodsAroundInterceptor，这个类是skywalking用来拦截非静态类的入口基类,这些

- 对于静态类的拦截积基类是StaticMethodsInterceptPoint，比如ConnectionCreateInterceptor，这个类是为了拦截com.mysql.jdbc.ConnectionImpl.getInstance() 这个静态方法而准备的。

那么这些解析器什么时候使用呢？那么就涉及到这些拦截类的加载，SkyWalking使用拦截类的加载基类AbstractClassEnhancePluginDefine来加载这些拦截类，这个类对拦截类的加载又分为三类

- 构造器方法 AbstractClassEnhancePluginDefine
- 实例方法  InstanceMethodsInterceptPoint
- 静态方法  StaticMethodsInterceptPoint

这个类还有其它方法，define,enhanceInstance，enhanceClass，因为我们最终目的是对实例或者静态方法进行增强，所以这三个方法分别就是做这两件事情的，最要入口方法define，是返回的是net.bytebuddy.dynamic.DynamicType.Builder，从名字我们看出这个是bytebuddy的，如果不了解bytebuddy是如何对类进行增加的可以参考我之前写的文章。

我们再次回到SkyWalkingAgent的PluginFinder插件查找类，SkyWalking基于插件扩展开发框架方便我们定义拦截类，这样设计非常灵活，我们来看一下PluginFinder的初始化过程：

``` java
pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
```

入口参数是通过new PluginBootstrap().loadPlugins()来加载所有插件，PluginBootstrap使用PluginResourcesResolver加载所有插件目录，PluginResourcesResolver最终调用的是AgentClassLoader从所有插件里面的resource目录找到skywalking-plugin.def的拦截定义类的定义文件，那么AgentClassLoader是从哪些目录加载这些插件的呢？我们知道，AgentClassLoader是SkyWalking自定的ClassLoader加载器，重写的findResource方法，这个方法其实返回就是启动目录的父目录，所以启动目录的所有子目录来找到插件定义文件，其实主要的插件定义都在Plugin目录。

找到所有插件定义类之后PluginFinder现在持有了所有插件定义类，它将增强的准备工作交给了BootstrapInstrumentBoost.inject方法处理，这个方法主要是为了创建bytebuddy的 AgentBuilder，真正做事的方法是prepareJREInstrumentation

``` java
/**
     * Generate dynamic delegate for ByteBuddy
     *
     * @param pluginFinder   gets the whole plugin list.
     * @param classesTypeMap hosts the class binary.
     * @return true if have JRE instrumentation requirement.
     * @throws PluginException when generate failure.
     */
    private static boolean prepareJREInstrumentation(PluginFinder pluginFinder,
        Map<String, byte[]> classesTypeMap) throws PluginException {
        TypePool typePool = TypePool.Default.of(BootstrapInstrumentBoost.class.getClassLoader());
        List<AbstractClassEnhancePluginDefine> bootstrapClassMatchDefines = pluginFinder.getBootstrapClassMatchDefine();
        for (AbstractClassEnhancePluginDefine define : bootstrapClassMatchDefines) {
            if (Objects.nonNull(define.getInstanceMethodsInterceptPoints())) {
                for (InstanceMethodsInterceptPoint point : define.getInstanceMethodsInterceptPoints()) {
                    if (point.isOverrideArgs()) {
                        generateDelegator(
                            classesTypeMap, typePool, INSTANCE_METHOD_WITH_OVERRIDE_ARGS_DELEGATE_TEMPLATE, point
                                .getMethodsInterceptor());
                    } else {
                        generateDelegator(
                            classesTypeMap, typePool, INSTANCE_METHOD_DELEGATE_TEMPLATE, point.getMethodsInterceptor());
                    }
                }
            }

            if (Objects.nonNull(define.getConstructorsInterceptPoints())) {
                for (ConstructorInterceptPoint point : define.getConstructorsInterceptPoints()) {
                    generateDelegator(
                        classesTypeMap, typePool, CONSTRUCTOR_DELEGATE_TEMPLATE, point.getConstructorInterceptor());
                }
            }

            if (Objects.nonNull(define.getStaticMethodsInterceptPoints())) {
                for (StaticMethodsInterceptPoint point : define.getStaticMethodsInterceptPoints()) {
                    if (point.isOverrideArgs()) {
                        generateDelegator(
                            classesTypeMap, typePool, STATIC_METHOD_WITH_OVERRIDE_ARGS_DELEGATE_TEMPLATE, point
                                .getMethodsInterceptor());
                    } else {
                        generateDelegator(
                            classesTypeMap, typePool, STATIC_METHOD_DELEGATE_TEMPLATE, point.getMethodsInterceptor());
                    }
                }
            }
        }
        return bootstrapClassMatchDefines.size() > 0;
    }
```

上面的代码主要是分三步，主要是我们上面介绍的AbstractClassEnhancePluginDefine的三个方法增强

- 调用getInstanceMethodsInterceptPoints 获取到定义类，通过 point.getMethodsInterceptor拿到拦截类generateDelegator和对实例方法进行增强
- 调用getConstructorsInterceptPoints获取到定义类，通过point.getConstructorInterceptor拿到拦截类generateDelegator和对实例方法进行增强
- 调用getStaticMethodsInterceptPoints获取到定义类，通过point.getMethodsInterceptor拿到拦截类generateDelegator和对实例方法进行增强

其实真正的增加是在SkyWalkingAgent的内部类Transformer处理，这个类继承bytebuddy AgentBuilder.Transformer实现，本质是调用我们刚才介绍的AbstractClassEnhancePluginDefine的define类完成实质性的代码增强

``` java

@Override
        public DynamicType.Builder<?> transform(final DynamicType.Builder<?> builder,
                                                final TypeDescription typeDescription,
                                                final ClassLoader classLoader,
                                                final JavaModule module) {
            LoadedLibraryCollector.registerURLClassLoader(classLoader);
            List<AbstractClassEnhancePluginDefine> pluginDefines = pluginFinder.find(typeDescription);
            if (pluginDefines.size() > 0) {
                DynamicType.Builder<?> newBuilder = builder;
                EnhanceContext context = new EnhanceContext();
                for (AbstractClassEnhancePluginDefine define : pluginDefines) {
                    DynamicType.Builder<?> possibleNewBuilder = define.define(
                            typeDescription, newBuilder, classLoader, context);
                    if (possibleNewBuilder != null) {
                        newBuilder = possibleNewBuilder;
                    }
                }
                if (context.isEnhanced()) {
                    LOGGER.debug("Finish the prepare stage for {}.", typeDescription.getName());
                }

                return newBuilder;
            }

            LOGGER.debug("Matched class {}, but ignore by finding mechanism.", typeDescription.getTypeName());
            return builder;
        }
    }
```

最后我们来看一下真正增强实例类的方法org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassEnhancePluginDefine#enhanceInstance

``` java

// ...方法太长，自己看一下

        /**
         * Manipulate class source code.<br/>
         *
         * new class need:<br/>
         * 1.Add field, name {@link #CONTEXT_ATTR_NAME}.
         * 2.Add a field accessor for this field.
         *
         * And make sure the source codes manipulation only occurs once.
         *
         */
        if (!typeDescription.isAssignableTo(EnhancedInstance.class)) {
            if (!context.isObjectExtended()) {
                newClassBuilder = newClassBuilder.defineField(
                    CONTEXT_ATTR_NAME, Object.class, ACC_PRIVATE | ACC_VOLATILE)
                                                 .implement(EnhancedInstance.class)
                                                 .intercept(FieldAccessor.ofField(CONTEXT_ATTR_NAME));
                context.extendObjectCompleted();
            }
        }
// 其它逻辑
 newClassBuilder = newClassBuilder.method(junction)
                                                         .intercept(MethodDelegation.withDefaultConfiguration()
                                                                                    .to(new InstMethodsInter(interceptor, classLoader)));
// ...其它逻辑
```

方法太长，我们只关注主重要的一些代码，第一块是让目标类实现EnhancedInstance接口，在目标方法里面定义一个名称是CONTEXT_ATTR_NAME()_$EnhancedClassField_ws的字段， 定义getSkyWalkingDynamicField() 和setSkyWalkingDynamicField() 两个方法，分别读写新增的_$EnhancedClassField_ws 字段，这个很重要，是用来承载Tracing信息的字段，下面一行是使用bytebuddy 方法拦截类InstMethodsInter，bytebuddy帮我们调用这个拦截类的intercept方法

``` java

/**
     * Intercept the target instance method.
     *
     * @param obj          target class instance.
     * @param allArguments all method arguments
     * @param method       method description.
     * @param zuper        the origin call ref.
     * @return the return value of target instance method.
     * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
     *                   bug, if anything triggers this condition ).
     */
    @RuntimeType
    public Object intercept(@This Object obj, @AllArguments Object[] allArguments, @SuperCall Callable<?> zuper,
        @Origin Method method) throws Throwable {
        EnhancedInstance targetObject = (EnhancedInstance) obj;

        MethodInterceptResult result = new MethodInterceptResult();
        try {
            interceptor.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
        } catch (Throwable t) {
            LOGGER.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
        }

        Object ret = null;
        try {
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                ret = zuper.call();
            }
        } catch (Throwable t) {
            try {
                interceptor.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
            } catch (Throwable t2) {
                LOGGER.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
            }
            throw t;
        } finally {
            try {
                ret = interceptor.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
            } catch (Throwable t) {
                LOGGER.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
            }
        }
        return ret;
    }
```

从代码可以看出，主要是调用SkyWalking定义的拦截类实例 基类是InstanceMethodsAroundInterceptor，比如我们上面提到的CreatePreparedStatementInterceptor和CreateStatementInterceptor

### Tracing上报解析

我们还是以上面的慢SQL上报为例进行说明，上面我们说到增加类会继承接口EnhancedInstance，在JDBC执行的过程中，SkyWalking分别对Connection，PreparedStatement或者createStatement方法进行增强，最后对PreparedStatement的executeQuery，executeUpdate executeLargeUpdate增加的org.apache.skywalking.apm.plugin.jdbc.mysql.PreparedStatementExecuteMethodsInterceptor或Statement的executeQuery，executeUpdate，executeLargeUpdate，executeBatchInternal，executeUpdateInternal，executeQuery，executeBatch方法进行增强的org.apache.skywalking.apm.plugin.jdbc.mysql.StatementExecuteMethodsInterceptor，前面对Connection，PreparedStatement增加主要是为了将链接信息，SQL参数信息放到上下文进行传递，最后PreparedStatementExecuteMethodsInterceptor或者StatementExecuteMethodsInterceptor进行上报处理，我们以PreparedStatementExecuteMethodsInterceptor为例来看一下代码

``` java

@Override
    public final void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments,
                                   Class<?>[] argumentsTypes, MethodInterceptResult result) {
        StatementEnhanceInfos cacheObject = (StatementEnhanceInfos) objInst.getSkyWalkingDynamicField();
        /**
         * For avoid NPE. In this particular case, Execute sql inside the {@link com.mysql.jdbc.ConnectionImpl} constructor,
         * before the interceptor sets the connectionInfo.
         * When invoking prepareCall, cacheObject is null. Because it will determine procedures's parameter types by executing sql in mysql 
         * before the interceptor sets the statementEnhanceInfos.
         * @see JDBCDriverInterceptor#afterMethod(EnhancedInstance, Method, Object[], Class[], Object)
         */
        if (cacheObject != null && cacheObject.getConnectionInfo() != null) {
            ConnectionInfo connectInfo = cacheObject.getConnectionInfo();
            AbstractSpan span = ContextManager.createExitSpan(
                buildOperationName(connectInfo, method.getName(), cacheObject
                    .getStatementName()), connectInfo.getDatabasePeer());
            Tags.DB_TYPE.set(span, "sql");
            Tags.DB_INSTANCE.set(span, connectInfo.getDatabaseName());
            Tags.DB_STATEMENT.set(span, SqlBodyUtil.limitSqlBodySize(cacheObject.getSql()));
            span.setComponent(connectInfo.getComponent());

            if (JDBCPluginConfig.Plugin.JDBC.TRACE_SQL_PARAMETERS) {
                final Object[] parameters = cacheObject.getParameters();
                if (parameters != null && parameters.length > 0) {
                    int maxIndex = cacheObject.getMaxIndex();
                    String parameterString = getParameterString(parameters, maxIndex);
                    SQL_PARAMETERS.set(span, parameterString);
                }
            }

            SpanLayer.asDB(span);
        }
    }

    @Override
    public final Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments,
                                    Class<?>[] argumentsTypes, Object ret) {
        StatementEnhanceInfos cacheObject = (StatementEnhanceInfos) objInst.getSkyWalkingDynamicField();
        if (cacheObject != null && cacheObject.getConnectionInfo() != null) {
            ContextManager.stopSpan();
        }
        return ret;
    }

    @Override
    public final void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments,
                                            Class<?>[] argumentsTypes, Throwable t) {
        StatementEnhanceInfos cacheObject = (StatementEnhanceInfos) objInst.getSkyWalkingDynamicField();
        if (cacheObject != null && cacheObject.getConnectionInfo() != null) {
            ContextManager.activeSpan().log(t);
        }
    }
```

在解释上面的代码之前，首先我们来了解几个概念：

#### Span
  
Span 分为 3 类：

- EntrySpan：当请求进入服务时会创建 EntrySpan 类型的 Span，它也是 TraceSegment 中的第一个 Span。例如，HTTP 服务、RPC 服务、MQ-Consumer 等入口服务的插件在接收到请求时都会创建相应的 EntrySpan。
- LocalSpan：它是在本地方法调用时可能创建的 Span 类型，在后面介绍 @Trace 注解的时候我们还会看到 LocalSpan。
- ExitSpan：当请求离开当前服务、进入其他服务时会创建 ExitSpan 类型的 Span。例如， Http Client 、RPC Client 发起远程调用或是 MQ-producer 生产消息时，都会产生该类型的 Span。

它们都继承自AbstractSpan ，其主要的方法有：

- setComponent() 方法：用于设置组件类型。它有两个重载，在 AbstractTracingSpan 实现中，有 componentId 和 componentName 两个字段，两个重载分别用于设置这两个字段。在 ComponentsDefine 中可以找到 SkyWalking 目前支持的组件类型。
- setLayer() 方法：用于设置 SpanLayer，也就是当前 Span 所处的位置。SpanLayer 是个枚举，可选项有 DB、RPC_FRAMEWORK、HTTP、MQ、CACHE。
- tag(AbstractTag, String) 方法：用于为当前 Span 添加键值对的 Tags。一个 Span 可以有多个 Tags。AbstractTag 中不仅包含了 String 类型的 Key 值，还包含了 Tag 的 ID 以及 canOverwrite 标识。AbstractTracingSpan 实现通过维护一个  List<TagValuePair> 集合（tags 字段）来记录 Tag 信息，TagValuePair 中则封装了 AbstractTag 类型的 Key 以及 String 类型的 Value。
- log() 方法：用于向当前 Span 中添加 Log，一个 Span 可以包含多条日志。在 AbstractTracingSpan 实现中通过维护一个 List<LogDataEntity> 集合（logs 字段）来记录 Log。LogDataEntity 会记录日志的时间戳以及 KV 信息，以异常日志为例，其中就会包含一个 Key 为“stack”的 KV，其 value 为异常堆栈。
- start() 方法：开启 Span，其中会设置当前 Span 的开始时间以及调用层级等信息。
- isEntry() 方法：判断当前是否是 EntrySpan。EntrySpan 的具体实现后面详细介绍。
- isExit() 方法：判断当前是否是 ExitSpan。ExitSpan  的具体实现后面详细介绍。
- ref() 方法：用于设置关联的 TraceSegment 。
  
#### TraceSegment

在 SkyWalking 中，TraceSegment 是一个介于 Trace 与 Span 之间的概念，它是一条 Trace 的一段，可以包含多个 Span。在微服务架构中，一个请求基本都会涉及跨进程（以及跨线程）的操作，例如， RPC 调用、通过 MQ 异步执行、HTTP 请求远端资源等，处理一个请求就需要涉及到多个服务的多个线程。TraceSegment 记录了一个请求在一个线程中的执行流程（即 Trace 信息）。将该请求关联的 TraceSegment 串联起来，就能得到该请求对应的完整 Trace。

#### Context

SkyWalking 中的每个 TraceSegment 都与一个 Context 上下文对象一对一绑定，Context 上下文不仅记录了 TraceSegment 的上下文信息，还提供了管理 TraceSegment 生命周期、创建 Span 以及跨进程（跨线程）传播相关的功能。

AbstractTracerContext 是对上下文概念的抽象，其中定义了 Context 上下文的基本行为：

- inject(ContextCarrier) 方法：在跨进程调用之前，调用方会通过 inject() 方法将当前 Context 上下文记录的全部信息注入到 ContextCarrier 参数中，Agent 后续会将 ContextCarrier 序列化并随远程调用进行传播。ContextCarrier 的具体实现在后面会详细分析。

- extract(ContextCarrier) 方法：跨进程调用的接收方会反序列化得到 ContextCarrier 对象，然后通过 extract() 方法从 ContextCarrier 中读取上游传递下来的 Trace 信息并记录到当前的 Context 上下文中。

- ContextSnapshot capture() 方法：在跨线程调用之前，SkyWalking Agent 会通过 capture() 方法将当前 Context 进行快照，然后将快照传递给其他线程。

- continued(ContextSnapshot) 方法：跨线程调用的接收方会从收到的 ContextSnapshot 中读取 Trace 信息并填充到当前 Context 上下文中。

- getReadableGlobalTraceId() 方法： 用于获取当前 Context 关联的 TraceId。

- createEntrySpan()、createLocalSpan() 方法、createExitSpan() 方法：用于创建 Span。

- activeSpan() 方法：用于获得当前活跃的 Span。在 TraceSegment 中，Span 也是按照栈的方式进行维护的，因为 Span 的生命周期符合栈的特性，即：先创建的 Span 后结束。

- stopSpan(AbstractSpan) 方法：用于停止指定 Span。

AbstractTraceContext 有两个实现类IgnoredTracerContext，TracingContext，IgnoredTracerContext 表示该 Trace 将会被丢失，所以其中不会记录任何信息，里面所有方法也都是空实现。这里重点来看 TracingContext，其核心字段如下：

- samplingService（SamplingService 类型）：负责完成 Agent 端的 Trace 采样，后面会展开介绍具体的采样逻辑。

- segment（TraceSegment 类型）：它是与当前 Context 上下文关联的 TraceSegment 对象，在 TracingContext 的构造方法中会创建该对象。

- activeSpanStack（LinkedList<AbstractSpan> 类型）：用于记录当前 TraceSegment 中所有活跃的 Span（即未关闭的 Span）。实际上 activeSpanStack 字段是作为栈使用的，TracingContext 提供了 push() 、pop() 、peek() 三个标准的栈方法，以及 first() 方法来访问栈底元素。

- spanIdGenerator（int 类型）：它是 Span ID 自增序列，初始值为 0。该字段的自增操作都是在一个线程中完成的，所以无需加锁。

结合上面的解析以及前一篇的介绍，我们知道SkyWalking使用堆栈进行Span管理，EntrySpan为TraceSegment入口，ExitSpan为TraceSegment出口，如果调用链复杂，我们可能会同时用EntrySpan和ExitSpan，但是对于上面的例子，我们只需要创建一个ExitSpan就可以了，所以上面代码不用解析已经不言自明。

那么数据是如何上报的呢？我们关注一下afterMethod方法，  ContextManager.stopSpan()这个方法最要是调用org.apache.skywalking.apm.agent.core.context.TracingContext#finish方法

``` java

/**
     * Finish this context, and notify all {@link TracingContextListener}s, managed by {@link
     * TracingContext.ListenerManager} and {@link TracingContext.TracingThreadListenerManager}
     */
    private void finish() {
        if (isRunningInAsyncMode) {
            asyncFinishLock.lock();
        }
        try {
            boolean isFinishedInMainThread = activeSpanStack.isEmpty() && running;
            if (isFinishedInMainThread) {
                /*
                 * Notify after tracing finished in the main thread.
                 */
                TracingThreadListenerManager.notifyFinish(this);
            }

            if (isFinishedInMainThread && (!isRunningInAsyncMode || asyncSpanCounter == 0)) {
                TraceSegment finishedSegment = segment.finish(isLimitMechanismWorking());
                TracingContext.ListenerManager.notifyFinish(finishedSegment);
                running = false;
            }
        } finally {
            if (isRunningInAsyncMode) {
                asyncFinishLock.unlock();
            }
        }
    }
```

当 TracingContext 通过 stopSpan() 方法关闭最后一个 Span 时，会调用 finish() 方法关闭相应的 TraceSegment，与此同时，还会调用 TracingContext.ListenerManager.notifyFinish() 方法通知所有监听 TracingContext 关闭事件的监听器 —— TracingContextListener，TraceSegmentServiceClient 是 TracingContextListener 接口的实现之一，其主要功能就是在 TraceSegment 结束时对其进行收集，并发送到后端的 OAP 集群。TraceSegmentServiceClient 底层维护了一个 DataCarrier 对象，其底层 Channels 默认有 5 个 Buffer，每个 Buffer 长度为 300，使用的是 IF_POSSIBLE 阻塞写入策略，底层会启动一个 ConsumerThread 线程。

TraceSegmentServiceClient 作为一个 TracingContextListener 接口的实现，会在 notifyFinish() 方法中，将刚刚结束的 TraceSegment 写入到 DataCarrier 中缓存。同时，TraceSegmentServiceClient 实现了 IConsumer 接口，封装了消费 Channels 中数据的逻辑，在 consume() 方法中会首先将消费到的 TraceSegment 对象序列化，然后通过 gRPC 请求发送到后端 OAP 集群，最后我们看一下TraceSegmentServiceClient的consume() 方法

``` java

@Override
    public void consume(List<TraceSegment> data) {
        if (CONNECTED.equals(status)) {
            final GRPCStreamServiceStatus status = new GRPCStreamServiceStatus(false);
            StreamObserver<SegmentObject> upstreamSegmentStreamObserver = serviceStub.withDeadlineAfter(
                Config.Collector.GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS
            ).collect(new StreamObserver<Commands>() {
                @Override
                public void onNext(Commands commands) {
                    ServiceManager.INSTANCE.findService(CommandService.class)
                                           .receiveCommand(commands);
                }

                @Override
                public void onError(
                    Throwable throwable) {
                    status.finished();
                    if (LOGGER.isErrorEnable()) {
                        LOGGER.error(
                            throwable,
                            "Send UpstreamSegment to collector fail with a grpc internal exception."
                        );
                    }
                    ServiceManager.INSTANCE
                        .findService(GRPCChannelManager.class)
                        .reportError(throwable);
                }

                @Override
                public void onCompleted() {
                    status.finished();
                }
            });

            try {
                for (TraceSegment segment : data) {
                    SegmentObject upstreamSegment = segment.transform();
                    upstreamSegmentStreamObserver.onNext(upstreamSegment);
                }
            } catch (Throwable t) {
                LOGGER.error(t, "Transform and send UpstreamSegment to collector fail.");
            }

            upstreamSegmentStreamObserver.onCompleted();

            status.wait4Finish();
            segmentUplinkedCounter += data.size();
        } else {
            segmentAbandonedCounter += data.size();
        }

        printUplinkStatus();
    }
    ```

注意，TraceSegmentServiceClient 在批量发送完 UpstreamSegment 数据之后，会通过 GRPCStreamServiceStatus 进行自旋等待，直至该批 UpstreamSegment 全部发送完毕。
