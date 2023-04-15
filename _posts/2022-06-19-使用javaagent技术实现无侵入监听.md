---
layout: post
title: 使用javaagent技术实现无侵入监听
subtitle: 介绍patroller开源工具
date: 2022-06-19T00:00:00+08:00
author: gregorius
header-img: img/post-bg-universe.jpg
catalog: true
tags: [实践]
---

### 什么是 Java Agent

Java Agent是JVM启动时给应用程序一种机会去修改class文件的机制，在启动的VM参数增加-javaagent 再加上自定义的扩展来实现，这种自定义扩展是一种插件开发机制。

使用 Java Agent 的步骤大致如下：

1.  定义一个 MANIFEST.MF 文件，在其中添加 premain-class 配置项。

2.  创建  premain-class 配置项指定的类，并在其中实现 premain() 方法，方法签名如下：

``` java
public static void premain(String agentArgs, Instrumentation inst){

   ... 

}
```

3.  将 MANIFEST.MF 文件和  premain-class 指定的类一起打包成一个 jar 包。

4.  使用 -javaagent 指定该 jar 包的路径即可执行其中的 premain() 方法。

如果使用maven打包，可以省去上面繁琐的操作，配置如下：

``` xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-jar-plugin</artifactId>
	<version>2.4</version>
	<configuration>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
                <mainClass>com.preapm.agent.Bootstrap</mainClass>
            </manifest>
            <manifestEntries>
                <Premain-Class>com.preapm.agent.APMAgentPremain</Premain-Class>
                <Can-Redefine-Classes>true</Can-Redefine-Classes>
                <Can-Retransform-Classes>true</Can-Retransform-Classes>
            </manifestEntries>
        </archive>
	</configuration>
</plugin>
```

实例

``` java

public class TestAgent {
    public static void premain(String agentArgs, 
            Instrumentation inst) {
        System.out.println("this is a java agent with two args");
        System.out.println("参数:" + agentArgs + "\n");
    }

    public static void premain(String agentArgs) {
        System.out.println("this is a java agent only one args");
        System.out.println("参数:" + agentArgs + "\n");
    }

}
```

premain() 方法有两个重载，如下所示，如果两个重载同时存在，【1】将会被忽略，只执行【2】

``` java
public static void premain(String agentArgs) [1]
public static void premain(String agentArgs, 
      Instrumentation inst); [2]
```

agentArgs 参数：-javaagent 命令携带的参数。在前面介绍 SkyWalking Agent 接入时提到，agent.service_name 这个配置项的默认值有三种覆盖方式，其中，使用探针配置进行覆盖，探针配置的值就是通过该参数传入的。
inst 参数：java.lang.instrumen.Instrumentation 是 Instrumention 包中定义的一个接口，它提供了操作类定义的相关方法。

如何用java agent修改class,比如实现监控功能在getNumber前后增加AOP逻辑？

比如有下面这个类

``` java

public class TestClass {
    public int getNumber() { return 1;  }
}

```

接下来需要编写一个Transformer类来实现对类的修改，下面以javassist为字节码注入工具进行演示，在方法前后择增加统计时间。

``` java

public class TestAgent {
    public static void premain(String agentArgs, Instrumentation inst) 
              throws Exception {

        // 注册一个 Transformer，该 Transformer在类加载时被调用
        inst.addTransformer(new Transformer(), true);
        inst.retransformClasses(TestClass.class);
        System.out.println("premain done");
    }

}
```

Transformer的实现如下（[参考代码](https://github.com/dingjs/javaagent)）：

``` java

class Transformer implements ClassFileTransformer {
    public byte[] transform(ClassLoader l, String className, 
       Class<?> c, ProtectionDomain pd, byte[] b)  {
        ProtectionDomain protectionDomain, byte[] classfileBuffer) {
        byte[] byteCode = classfileBuffer;
        className = className.replace('/', '.');

         if (!isNeedLogExecuteInfo(className)) {
            return byteCode;
        }

        if (null == loader) {
            loader = Thread.currentThread().getContextClassLoader();
        }

        byteCode = aopLog(loader, className, byteCode);
        return byteCode;
    }
}

 private byte[] aopLog(ClassLoader loader, String className, byte[] byteCode) {
        try {
            ClassPool cp = ClassPool.getDefault();
            CtClass cc;
            try {
                cc = cp.get(className);
            } catch (NotFoundException e) {
                cp.insertClassPath(new LoaderClassPath(loader));
                cc = cp.get(className);
            }
            byteCode = aopLog(cc, className, byteCode);
        } catch (Exception ex) {
            System.err.println(ex);
        }
        return byteCode;
    }

    private byte[] aopLog(CtClass cc, String className, byte[] byteCode) throws CannotCompileException, IOException {
        if (null == cc) {
            return byteCode;
        }
        if (!cc.isInterface()) {
            CtMethod[] methods = cc.getDeclaredMethods();
            if (null != methods && methods.length > 0) {
                boolean isOpenPojoMonitor = ConfigUtils.isOpenPojoMonitor();
                Set<String> getSetMethods = Collections.emptySet();
                if (!isOpenPojoMonitor) {
                    getSetMethods = PojoDetector.getPojoMethodNames(methods);
                }
                for (CtMethod m : methods) {
                    if (isOpenPojoMonitor || !getSetMethods.contains(m.getName())) {
                        aopLog(className, m);
                    }
                }
                byteCode = cc.toBytecode();
            }
        }
        cc.detach();
        return byteCode;
    }

    private void aopLog(String className, CtMethod m) throws CannotCompileException {
        if (null == m || m.isEmpty()) {
            return;
        }
        boolean isMethodStatic = Modifier.isStatic(m.getModifiers());
        String aopClassName = isMethodStatic ? "\"" + className + "\"" : "this.getClass().getName()";
        final String timeMethodStr
            = ConfigUtils.isUsingNanoTime() ? "java.lang.System.nanoTime()" : "java.lang.System.currentTimeMillis()";

        // 避免变量名重复
        m.addLocalVariable("dingjsh_javaagent_elapsedTime", CtClass.longType);
        m.insertBefore("dingjsh_javaagent_elapsedTime = " + timeMethodStr + ";");
        m.insertAfter("dingjsh_javaagent_elapsedTime = " + timeMethodStr + " - dingjsh_javaagent_elapsedTime;");
        m.insertAfter(LOG_UTILS + ".log(" + aopClassName + ",\"" + m.getName()
            + "\",(long)dingjsh_javaagent_elapsedTime" + ");");
    }

```

## 如何实现精确配置监听

细心的读者可能会发现，虽然上面的方式采用排除法排除了不需要监控的类，但是监控范围还是过广，同时，可以看到最后的log实在是太简单，无法实现一些复杂的逻辑，不具备很好的扩展性，那么有没有更好的方式呢？

这个[patroller](https://github.com/gregoriusxu/patroller) 项目采用精确匹配的方式进行监控类扩展，配置采用yml配置方式，配置如下：

``` yml

#基础的插件包配置，会优先加载
basePlugins:
  pre-agent-common:
    jarName: pre-agent-common
  pre-zipkin-sdk:
    jarName: pre-zipkin-sdk
#第三方插件包配置
plugins:
  pre-zipkin-plugin:
    jarName: pre-zipkin-plugin
    interceptorNames:
      - com.preapm.agent.plugin.interceptor.ZipkinInterceptor
    loadPatterns:
      - com.preapm.agent.constant.BaseConstants
      

---
#AOP切面配置，类似sparing aop配置
patterns:
  #==========================================================一下是具体业务配置根据具体项目路径修改================================     
      
  #业务service配置    
  pre-service:
    patterns:
      - com.*.service.impl.*
    excludedPatterns:
    includedPatterns:
      #- key: .*(run\(\)|call\(\))
      #  interceptors:
      #     - com.preapm.agent.plugin.interceptor.JdkThreadInterceptor
      - key: .*
        interceptors:
           - com.preapm.agent.plugin.interceptor.ZipkinInterceptor
    interceptors:
      - com.preapm.agent.plugin.interceptor.ZipkinInterceptor
    track:
           inParam: true #记录入参
           outParam: true #记录出参
           time: -1    #不设置时间限制   
           serialize: fastjson
    consMethod: false                 
  #agent测试配置    
  test:
      #插件织入匹配方法路径，支持正则表达式
      patterns:
        - com.preapm.agent.Bootstrap
      #排除匹配路径
      excludedPatterns:  
       - key: com.preapm.agent.Bootstrap\(\)
      includedPatterns:
      - key:  .*
      interceptors:
        - com.preapm.agent.plugin.interceptor.ZipkinInterceptor
      track:
           inParam: false
           outParam: true
           time: -1 
```

同时在AOP实现方面具备很好的扩展性，主要看这个类
``` java
com.preapm.agent.weave.ClassWrapper
```

代码如下：
``` java

public String beginSrc(ClassLoader classLoader,byte[] classfileBuffer,CtClass ctClass, CtMethod ctMethod) {
		String methodName = ctMethod.getName();
		List<String> paramNameList = Arrays.asList(ReflectMethodUtil.getMethodParamNames(classLoader,classfileBuffer,ctClass, ctMethod));
		try {
			 //System.out.println("方法名称："+methodName+" 参数类型大小："+ctMethod.getParameterTypes().length+" paramNameList："+paramNameList.toArray());
			
			  String template = ctMethod.getReturnType().getName().equals("void")
                ?
                "{\n" +
                "    %s        \n" +  beforAgent(methodName,paramNameList)+" \n"+
                "    try {\n" + 
                "        %s$agent($$);\n" +
                "    } catch (Throwable e) {\n" +
                "        %s\n" +doError(BaseConstants.THROWABLE_NAME_STR)+
                "        throw e;\n" +
                "    }finally{\n" +
                "        %s\n" + afterAgent(null)+" \n"+
                "    }\n" +
                "}"
                :
                "{\n" +
                "    %s        \n" +
                "    Object result=null;\n" +beforAgent(methodName,paramNameList)+" \n"+
                "    try {\n" +
                "        result=($w)%s$agent($$);\n" +
                "    } catch (Throwable e) {\n" +
                "        %s            \n" +doError(BaseConstants.THROWABLE_NAME_STR)+
                "        throw e;\n" +
                "    }finally{\n" +
                "        %s        \n" + afterAgent(BaseConstants.RESULT_NAME_STR)+" \n"+
                "    }\n" +
                "    return ($r) result;\n" +
                "}";

			String insertBeginSrc = this.beginSrc == null ? "" : this.beginSrc;
			String insertErrorSrc = this.errorSrc == null ? "" : this.errorSrc;
			String insertEndSrc = this.endSrc == null ? "" : this.endSrc;
			String result = String.format(template,
					new Object[] { insertBeginSrc, ctMethod.getName(), insertErrorSrc, insertEndSrc });
			//log.info("result:"+result);
			return result;
		} catch (NotFoundException localNotFoundException) {
			log.severe(org.apache.commons.lang3.exception.ExceptionUtils.getStackTrace(localNotFoundException));
			throw new RuntimeException(localNotFoundException);
		}
	}
```

最终实现是通过如下代码

``` java
package com.preapm.agent.weave.impl.ClassWrapperAroundInterceptor

class ClassWrapperAroundInterceptor
{
    //... 其他逻辑

    public String beforAgent(String methodName, List<String> argNameList) {

            StringBuilder stringBuilder = new StringBuilder();
            stringBuilder.append("com.preapm.agent.common.bean.MethodInfo preMethondInfo = new com.preapm.agent.common.bean.MethodInfo();").append(line());
            //stringBuilder.append("com.preapm.agent.common.bean.MethodInfo preMethondInfo = com.preapm.agent.common.context.AroundInterceptorContext.loader(Thread.currentThread().getContextClassLoader());").append(line());
            stringBuilder.append("preMethondInfo.setTarget(this);").append(line());
            stringBuilder.append("preMethondInfo.setMethodName(" + toStr(methodName) + ");").append(line());
            if (argNameList != null && argNameList.size() != 0) {
                stringBuilder.append("preMethondInfo.setArgs($args);").append(line());
                stringBuilder.append("String preMethodArgsStr = ").append(toStr(StringUtils.join(argNameList, ",")))
                        .append(";").append(line());
                stringBuilder.append("preMethondInfo.setArgsName(preMethodArgsStr.split(" + toStr(",") + "));")
                        .append(line());
            }
            setSerialize(stringBuilder);
            setPlugin(stringBuilder);
            setTrack(stringBuilder);

            stringBuilder.append("com.preapm.agent.common.context.AroundInterceptorContext.start(Thread.currentThread().getContextClassLoader(),preMethondInfo);")
                    .append(line());
            return stringBuilder.toString();

    }
}
```

最后我们看看AroundInterceptorContext

``` java
public static void start(ClassLoader classLoader,MethodInfo methodInfo, String... names) {
	for (AroundInterceptor i : get(classLoader,names)) {
		i.before(methodInfo);
	}
}
```

可以看到，最终是通过遍历我们配置的AroundInterceptor接口来实现扩展的功能


### 终极方案

大家可以发现以上方案已经是非常好了，我们可以通过精确配置需要监控的类，同时可以通过扩展将监控数据使用扩展的方式存储到任何地方，但是这种方案也有其局限性，首先AroundInterceptor的逻辑实现可能非常复杂，特别是对于一些复杂调用链的情况，其次这种上报方式不是很好调度，容易引发性能问题，同时对于一些复杂的采样，合计，汇总等监控方式还是需要实现复杂的代码才能实现，那么有没有更好的方案呢？

答案是当然有，我推荐[skywalking](https://github.com/apache/skywalking)，如果需要更进一步的学习，可以参考github或者官方文档，原理和上面所讲的是类似的，但是上面只是讲了客户的采集，对于服务端的实现还需要读者进一步挖掘。








