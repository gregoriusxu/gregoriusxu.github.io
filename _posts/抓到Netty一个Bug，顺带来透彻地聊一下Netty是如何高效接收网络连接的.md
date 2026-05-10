---
layout: post
title: 一文聊透Netty核心引擎Reactor的运转架构
subtitle: 
author: bin 的技术小屋
header-img: img/post-bg-universe.jpg
catalog: true
tags: [设计]
---

版权声明：此文章归原作者所有，转载请注明出处

原创 **bin 的技术小屋**

微信号 gh_6192ca0a769d

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

> 本系列Netty源码解析文章基于 **4.1.56.Final**版本

对于一个高性能网络通讯框架来说，最最重要也是最核心的工作就是如何高效的接收客户端连接，这就好比我们开了一个饭店，那么迎接客人就是饭店最重要的工作，我们要先把客人迎接进来，不能让客人一看人多就走掉，只要客人进来了，哪怕菜做的慢一点也没关系。

本文笔者就来为大家介绍下netty这块最核心的内容，看看netty是如何高效的接收客户端连接的。

下图为笔者在一个月黑风高天空显得那么深邃遥远的夜晚，闲来无事，于是捧起Netty关于如何接收连接这部分源码细细品读的时候，意外的发现了一个影响Netty接收连接吞吐的一个Bug。

![81](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/picture81.png)

于是笔者就在Github提了一个?Issue#11708，阐述了下这个Bug产生的原因以及导致的结果并和Netty的作者一起讨论了下修复措施。如上图所示。

> Issue#11708：<https://github.com/netty/netty/issues/11708>

这里先不详细解释这个Issue，也不建议大家现在就打开这个Issue查看，笔者会在本文的介绍中随着源码深入的解读慢慢的为大家一层一层地拨开迷雾。

之所以在文章的开头把这个拎出来，笔者是想让大家带着怀疑，审视，欣赏，崇敬，敬畏的态度来一起品读世界顶级程序员编写的代码。由衷的感谢他们在这一领域做出的贡献。

好了，问题抛出来后，我们就带着这个疑问来开始本文的内容吧~~~

![82](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/picture82.png)

## 前文回顾

按照老规矩，再开始本文的内容之前，我们先来回顾下前边几篇文章的概要内容帮助大家梳理一个框架全貌出来。

> 笔者这里再次想和读者朋友们强调的是本文可以独立观看，并不依赖前边系列文章的内容，只是大家如果对相关细节部分感兴趣的话，可以在阅读完本文之后在去回看相关文章。

在前边的系列文章中，笔者为大家介绍了驱动Netty整个框架运转的核心引擎Reactor的创建，启动，运行的全流程。从现在开始Netty的整个核心框架就开始运转起来开始工作了，本文要介绍的主要内容就是Netty在启动之后要做的第一件事件：监听端口地址，高效接收客户端连接。

在[《聊聊Netty那些事儿之从内核角度看IO模型》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483737&idx=1&sn=7ef3afbb54289c6e839eed724bb8a9d6&chksm=ce77c71ef9004e08e3d164561e3a2708fc210c05408fa41f7fe338d8e85f39c1ad57519b614e&scene=21#wechat_redirect)一文中，我们是从整个网络框架的基石IO模型的角度整体阐述了下Netty的IO线程模型。

而Netty中的Reactor正是IO线程在Netty中的模型定义。Reactor在Netty中是以Group的形式出现的，分为:

- 主Reactor线程组也就是我们在启动代码中配置的`EventLoopGroup bossGroup`,main reactor group中的reactor主要负责监听客户端连接事件，高效的处理客户端连接。也是本文我们要介绍的重点。

- 从Reactor线程组也就是我们在启动代码中配置的`EventLoopGroup workerGroup`，sub reactor group中的reactor主要负责处理客户端连接上的IO事件，以及异步任务的执行。

最后我们得出Netty的整个IO模型如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUZnjTia9x6OdAvgr1icM1ZsNialDtXCOD5vvVGh56FT2yKauwTch6oYbrn1icPuYKaqY8nPibicWv66sQfw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

netty中的reactor.png

本文我们讨论的重点就是MainReactorGroup的核心工作上图中所示的步骤1，步骤2，步骤3。

在从整体上介绍完Netty的IO模型之后，我们又在[?《Reactor在Netty中的实现(创建篇)》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483907&idx=1&sn=084c470a8fe6234c2c9461b5f713ff30&chksm=ce77c444f9004d52e7c6244bee83479070effb0bc59236df071f4d62e91e25f01715fca53696&scene=21#wechat_redirect)中完整的介绍了Netty框架的骨架主从Reactor组的搭建过程，阐述了Reactor是如何被创建出来的，并介绍了它的核心组件如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUZnjTia9x6OdAvgr1icM1ZsNiaRn9ZX4dJLJdyxSEEXojs5lEmPNiaBfstFOG95KMGibJed4vo3xMhnE6g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

image.png

- `thread`即为Reactor中的IO线程，主要负责监听IO事件，处理IO任务，执行异步任务。

- `selector`则是JDK NIO对操作系统底层IO多路复用技术实现的封装。用于监听IO就绪事件。

- `taskQueue`用于保存Reactor需要执行的异步任务，这些异步任务可以由用户在业务线程中向Reactor提交，也可以是Netty框架提交的一些自身核心的任务。

- `scheduledTaskQueue`则是保存Reactor中执行的定时任务。代替了原有的时间轮来执行延时任务。

- `tailQueue`保存了在Reactor需要执行的一些尾部收尾任务，在普通任务执行完后 Reactor线程会执行尾部任务，比如对Netty 的运行状态做一些统计数据，例如任务循环的耗时、占用物理内存的大小等等

在骨架搭建完毕之后，我们随后又在在[?《详细图解Netty Reactor启动全流程》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484005&idx=1&sn=52f51269902a58f40d33208421109bc3&chksm=ce77c422f9004d340e5b385ef6ba24dfba1f802076ace80ad6390e934173a10401e64e13eaeb&scene=21#wechat_redirect)》一文中介绍了**本文的主角服务端NioServerSocketChannel的创建，初始化，绑定端口地址，向main reactor注册监听`OP_ACCEPT事件`的完整过程**。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUZnjTia9x6OdAvgr1icM1ZsNiaMNnpeQav9HykpMEYenDPdshUtLBMicYHd5F9HwloOsE6FfLVtGW0XRA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Reactor启动后的结构.png

main reactor如何处理OP\_ACCEPT事件将会是本文的主要内容。

自此Netty框架的main reactor group已经启动完毕，开始准备监听OP\_accept事件，当客户端连接上来之后，OP\_ACCEPT事件活跃，main reactor开始处理OP\_ACCEPT事件接收客户端连接了。

而netty中的IO事件分为：OP\_ACCEPT事件，OP\_READ事件，OP\_WRITE事件和OP\_CONNECT事件，netty对于IO事件的监听和处理统一封装在Reactor模型中，这四个IO事件的处理过程也是我们后续文章中要单独拿出来介绍的，本文我们聚焦OP\_ACCEPT事件的处理。

而为了让大家能够对IO事件的处理有一个完整性的认识，笔者写了[?《一文聊透Netty核心引擎Reactor的运转架构》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484087&idx=1&sn=0c065780e0f05c23c8e6465ede86cba0&chksm=ce77c4f0f9004de63be369a664105708bc5975b52993f4a6df223caed34cc1ef6185a16acd75&token=997171731&lang=zh_CN&scene=21#wechat_redirect)这篇文章，在文章中详细介绍了Reactor线程的整体运行框架。

![图片](https://mmbiz.qpic.cn/mmbiz_png/sOIZXFW0vUZnjTia9x6OdAvgr1icM1ZsNiavrDh48SPMAM6BFPzvme6iceQT8aibcKY54GLfSOm2F7yCqynlkI9HkOg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Reactor线程运行时结构.png

Reactor线程会在一个死循环中996不停的运转，在循环中会不断的轮询监听Selector上的IO事件，当IO事件活跃后，Reactor从Selector上被唤醒转去执行IO就绪事件的处理，在这个过程中我们引出了上述四种IO事件的处理入口函数。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>private</span>&nbsp;<span>void</span>&nbsp;<span>processSelectedKey</span><span>(SelectionKey&nbsp;k,&nbsp;AbstractNioChannel&nbsp;ch)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//获取Channel的底层操作类Unsafe</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;AbstractNioChannel.NioUnsafe&nbsp;unsafe&nbsp;=&nbsp;ch.unsafe();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(!k.isValid())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;......如果SelectionKey已经失效则关闭对应的Channel......<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//获取IO就绪事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;readyOps&nbsp;=&nbsp;k.readyOps();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//处理Connect事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;((readyOps&nbsp;&amp;&nbsp;SelectionKey.OP_CONNECT)&nbsp;!=&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;ops&nbsp;=&nbsp;k.interestOps();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//移除对Connect事件的监听，否则Selector会一直通知</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ops&nbsp;&amp;=&nbsp;~SelectionKey.OP_CONNECT;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;k.interestOps(ops);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//触发channelActive事件处理Connect事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unsafe.finishConnect();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//处理Write事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;((readyOps&nbsp;&amp;&nbsp;SelectionKey.OP_WRITE)&nbsp;!=&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ch.unsafe().forceFlush();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//处理Read事件或者Accept事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;((readyOps&nbsp;&amp;&nbsp;(SelectionKey.OP_READ&nbsp;|&nbsp;SelectionKey.OP_ACCEPT))&nbsp;!=&nbsp;<span>0</span>&nbsp;||&nbsp;readyOps&nbsp;==&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unsafe.read();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(CancelledKeyException&nbsp;ignored)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unsafe.close(unsafe.voidPromise());<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

本文笔者将会为大家重点介绍`OP_ACCEPT事件`的处理入口函数`unsafe.read()`的整个源码实现。

当客户端连接完成三次握手之后，main reactor中的selector产生`OP_ACCEPT事件`活跃，main reactor随即被唤醒，来到了`OP_ACCEPT事件`的处理入口函数开始接收客户端连接。

## 1\. Main Reactor处理OP\_ACCEPT事件

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

OP\_ACCEPT事件活跃.png

当`Main Reactor`轮询到`NioServerSocketChannel`上的`OP_ACCEPT事件`就绪时，Main Reactor线程就会从`JDK Selector`上的阻塞轮询API`selector.select(timeoutMillis)`调用中返回。转而去处理`NioServerSocketChannel`上的`OP_ACCEPT事件`。

```
<span></span><code><span>public</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>NioEventLoop</span>&nbsp;<span>extends</span>&nbsp;<span>SingleThreadEventLoop</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>private</span>&nbsp;<span>void</span>&nbsp;<span>processSelectedKey</span><span>(SelectionKey&nbsp;k,&nbsp;AbstractNioChannel&nbsp;ch)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;AbstractNioChannel.NioUnsafe&nbsp;unsafe&nbsp;=&nbsp;ch.unsafe();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;..............省略.................<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;readyOps&nbsp;=&nbsp;k.readyOps();<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;((readyOps&nbsp;&amp;&nbsp;SelectionKey.OP_CONNECT)&nbsp;!=&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;..............处理OP_CONNECT事件.................<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;((readyOps&nbsp;&amp;&nbsp;SelectionKey.OP_WRITE)&nbsp;!=&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;..............处理OP_WRITE事件.................<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;((readyOps&nbsp;&amp;&nbsp;(SelectionKey.OP_READ&nbsp;|&nbsp;SelectionKey.OP_ACCEPT))&nbsp;!=&nbsp;<span>0</span>&nbsp;||&nbsp;readyOps&nbsp;==&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//本文重点处理OP_ACCEPT事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unsafe.read();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(CancelledKeyException&nbsp;ignored)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unsafe.close(unsafe.voidPromise());<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br></code>
```

- 处理IO就绪事件的入口函数`processSelectedKey`中的参数`AbstractNioChannel ch`正是Netty服务端`NioServerSocketChannel`。因为此时的执行线程为main reactor线程，而main reactor上注册的正是netty服务端NioServerSocketChannel负责监听端口地址，接收客户端连接。

- 通过`ch.unsafe()`获取到的NioUnsafe操作类正是NioServerSocketChannel中对底层JDK NIO ServerSocketChannel的Unsafe底层操作类。

> `Unsafe接口`是Netty对Channel底层操作行为的封装，比如NioServerSocketChannel的底层Unsafe操作类干的事情就是`绑定端口地址`，`处理OP_ACCEPT事件`。

这里我们看到，Netty将`OP_ACCEPT事件`处理的入口函数封装在`NioServerSocketChannel`里的底层操作类Unsafe的`read`方法中。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

而NioServerSocketChannel中的Unsafe操作类实现类型为`NioMessageUnsafe`定义在上图继承结构中的`AbstractNioMessageChannel父类中`。

下面我们到`NioMessageUnsafe#read`方法中来看下Netty对`OP_ACCPET事件`的具体处理过程：

## 2\. 接收客户端连接核心流程框架总览

我们还是按照老规矩，先从整体上把整个OP\_ACCEPT事件的逻辑处理框架提取出来，让大家先总体俯视下流程全貌，然后在针对每个核心点位进行各个击破。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接收客户端连接.png

main reactor线程是在一个`do...while{...}`循环read loop中不断的调用JDK NIO `serverSocketChannel.accept()`方法来接收完成三次握手的客户端连接`NioSocketChannel`的，并将接收到的客户端连接NioSocketChannel临时保存在`List<Object> readBuf`集合中，后续会服务端NioServerSocketChannel的pipeline中通过ChannelRead事件来传递，最终会在ServerBootstrapAcceptor这个ChannelHandler中被处理初始化，并将其注册到Sub Reator Group中。

这里的read loop循环会被限定只能读取**16次**，当main reactor从NioServerSocketChannel中读取客户端连接NioSocketChannel的次数达到**16次**之后，无论此时是否还有客户端连接都不能在继续读取了。

因为我们在[?《一文聊透Netty核心引擎Reactor的运转架构》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484087&idx=1&sn=0c065780e0f05c23c8e6465ede86cba0&chksm=ce77c4f0f9004de63be369a664105708bc5975b52993f4a6df223caed34cc1ef6185a16acd75&token=997171731&lang=zh_CN&scene=21#wechat_redirect)一文中提到，netty对reactor线程压榨的比较狠，要干的事情很多，除了要监听轮询IO就绪事件，处理IO就绪事件，还需要执行用户和netty框架本省提交的异步任务和定时任务。

所以这里的main reactor线程不能在read loop中无限制的执行下去，因为还需要分配时间去执行异步任务，不能因为无限制的接收客户端连接而耽误了异步任务的执行。所以这里将read loop的循环次数限定为16次。

如果main reactor线程在read loop中读取客户端连接NioSocketChannel的次数已经满了16次，即使此时还有客户端连接未接收，那么main reactor线程也不会再去接收了，而是转去执行异步任务，当异步任务执行完毕后，还会在回来执行剩余接收连接的任务。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Reactor线程运行时结构.png

main reactor线程退出read loop循环的条件有两个：

1. 在限定的16次读取中，已经没有新的客户端连接要接收了。退出循环。

2. 从NioServerSocketChannel中读取客户端连接的次数达到了16次，无论此时是否还有客户端连接都需要退出循环。

以上就是Netty在接收客户端连接时的整体核心逻辑，下面笔者将这部分逻辑的核心源码实现框架提取出来，方便大家根据上述核心逻辑与源码中的处理模块对应起来，还是那句话，这里只需要总体把握核心处理流程，不需要读懂每一行代码，笔者会在文章的后边分模块来各个击破它们。

```
<span></span><code><span>public</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>AbstractNioMessageChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioChannel</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>NioMessageUnsafe</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioUnsafe</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//存放连接建立后，创建的客户端SocketChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;List&lt;Object&gt;&nbsp;readBuf&nbsp;=&nbsp;<span>new</span>&nbsp;ArrayList&lt;Object&gt;();<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>read</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//必须在Main&nbsp;Reactor线程中执行</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>assert</span>&nbsp;<span>eventLoop</span><span>()</span>.<span>inEventLoop</span><span>()</span></span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//注意下面的config和pipeline都是服务端ServerSocketChannel中的</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;ChannelConfig&nbsp;config&nbsp;=&nbsp;config();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;ChannelPipeline&nbsp;pipeline&nbsp;=&nbsp;pipeline();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//创建接收数据Buffer分配器（用于分配容量大小合适的byteBuffer用来容纳接收数据）</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//在接收连接的场景中，这里的allocHandle只是用于控制read loop的循环读取创建连接的次数。</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;RecvByteBufAllocator.Handle&nbsp;allocHandle&nbsp;=&nbsp;unsafe().recvBufAllocHandle();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;allocHandle.reset(config);<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>boolean</span>&nbsp;closed&nbsp;=&nbsp;<span>false</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Throwable&nbsp;exception&nbsp;=&nbsp;<span>null</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>do</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//底层调用NioServerSocketChannel-&gt;doReadMessages&nbsp;创建客户端SocketChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;localRead&nbsp;=&nbsp;doReadMessages(readBuf);<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//已无新的连接可接收则退出read&nbsp;loop</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(localRead&nbsp;==&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>break</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(localRead&nbsp;&lt;&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;closed&nbsp;=&nbsp;<span>true</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>break</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//统计在当前事件循环中已经读取到得Message数量（创建连接的个数）</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;allocHandle.incMessagesRead(localRead);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>while</span>&nbsp;(allocHandle.continueReading());<span>//判断是否已经读满16次</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(Throwable&nbsp;t)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;exception&nbsp;=&nbsp;t;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;size&nbsp;=&nbsp;readBuf.size();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>for</span>&nbsp;(<span>int</span>&nbsp;i&nbsp;=&nbsp;<span>0</span>;&nbsp;i&nbsp;&lt;&nbsp;size;&nbsp;i&nbsp;++)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;readPending&nbsp;=&nbsp;<span>false</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//在NioServerSocketChannel对应的pipeline中传播ChannelRead事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//初始化客户端SocketChannel，并将其绑定到Sub&nbsp;Reactor线程组中的一个Reactor上</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline.fireChannelRead(readBuf.get(i));<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//清除本次accept&nbsp;创建的客户端SocketChannel集合</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;readBuf.clear();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;allocHandle.readComplete();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//触发readComplete事件传播</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline.fireChannelReadComplete();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;....................省略............<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>finally</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;....................省略............<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;}<br>}<br></code>
```

这里首先要通过断言`assert eventLoop().inEventLoop()`确保处理接收客户端连接的线程必须为Main Reactor 线程。

而main reactor中主要注册的是服务端NioServerSocketChannel，主要负责处理`OP_ACCEPT事件`，所以当前main reactor线程是在NioServerSocketChannel中执行接收连接的工作。

所以这里我们通过`config()`获取到的是NioServerSocketChannel的属性配置类`NioServerSocketChannelConfig`,它是在Reactor的启动阶段被创建出来的。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>NioServerSocketChannel</span><span>(ServerSocketChannel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//父类AbstractNioChannel中保存JDK&nbsp;NIO原生ServerSocketChannel以及要监听的事件OP_ACCEPT</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>super</span>(<span>null</span>,&nbsp;channel,&nbsp;SelectionKey.OP_ACCEPT);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//DefaultChannelConfig中设置用于Channel接收数据用的buffer-&gt;AdaptiveRecvByteBufAllocator</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;config&nbsp;=&nbsp;<span>new</span>&nbsp;NioServerSocketChannelConfig(<span>this</span>,&nbsp;javaChannel().socket());<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

同理这里通过`pipeline()`获取到的也是NioServerSocketChannel中的`pipeline`。它会在NioServerSocketChannel向main reactor注册成功之后被初始化。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

ServerChannelPipeline完整结构.png

前边提到main reactor线程会被限定只能在read loop中向NioServerSocketChannel读取16次客户端连接，所以在开始read loop之前，我们需要创建一个能够保存记录读取次数的对象，在每次read loop循环之后，可以根据这个对象来判断是否结束read loop。

这个对象就是这里的 `RecvByteBufAllocator.Handle allocHandle`专门用于统计read loop中接收客户端连接的次数，以及判断是否该结束read loop转去执行异步任务。

当这一切准备就绪之后，main reactor线程就开始在`do{....}while(...)`循环中接收客户端连接了。

在 read loop中通过调用`doReadMessages函数`接收完成三次握手的客户端连接，底层会调用到JDK NIO ServerSocketChannel的accept方法，从内核全连接队列中取出客户端连接。

返回值`localRead`表示接收到了多少客户端连接，客户端连接通过accept方法只会一个一个的接收，所以这里的`localRead`正常情况下都会返回`1`，当`localRead <= 0`时意味着已经没有新的客户端连接可以接收了，本次main reactor接收客户端的任务到这里就结束了，跳出read loop。开始新的一轮IO事件的监听处理。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>static</span>&nbsp;SocketChannel&nbsp;<span>accept</span><span>(<span>final</span>&nbsp;ServerSocketChannel&nbsp;serverSocketChannel)</span>&nbsp;<span>throws</span>&nbsp;IOException&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;AccessController.doPrivileged(<span>new</span>&nbsp;PrivilegedExceptionAction&lt;SocketChannel&gt;()&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;SocketChannel&nbsp;<span>run</span><span>()</span>&nbsp;<span>throws</span>&nbsp;IOException&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;serverSocketChannel.accept();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(PrivilegedActionException&nbsp;e)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>throw</span>&nbsp;(IOException)&nbsp;e.getCause();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

随后会将接收到的客户端连接占时存放到`List<Object> readBuf`集合中。

```
<span></span><code>&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>NioMessageUnsafe</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioUnsafe</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//存放连接建立后，创建的客户端SocketChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;List&lt;Object&gt;&nbsp;readBuf&nbsp;=&nbsp;<span>new</span>&nbsp;ArrayList&lt;Object&gt;();<br>}<br></code>
```

调用`allocHandle.incMessagesRead`统计本次事件循环中接收到的客户端连接个数，最后在read loop末尾通过`allocHandle.continueReading`判断是否达到了限定的16次。从而决定main reactor线程是继续接收客户端连接还是转去执行异步任务。

main reactor线程退出read loop的两个条件：

1. 在限定的16次读取中，已经没有新的客户端连接要接收了。退出循环。

2. 从NioServerSocketChannel中读取客户端连接的次数达到了16次，无论此时是否还有客户端连接都需要退出循环。

当满足以上两个退出条件时，main reactor线程就会退出read loop，由于在read loop中接收到的客户端连接全部暂存在`List<Object> readBuf`集合中,随后开始遍历readBuf，在NioServerSocketChannel的pipeline中传播ChannelRead事件。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;size&nbsp;=&nbsp;readBuf.size();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>for</span>&nbsp;(<span>int</span>&nbsp;i&nbsp;=&nbsp;<span>0</span>;&nbsp;i&nbsp;&lt;&nbsp;size;&nbsp;i&nbsp;++)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;readPending&nbsp;=&nbsp;<span>false</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//NioServerSocketChannel对应的pipeline中传播read事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//io.netty.bootstrap.ServerBootstrap.ServerBootstrapAcceptor.channelRead</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//初始化客户端SocketChannel，并将其绑定到Sub&nbsp;Reactor线程组中的一个Reactor上</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline.fireChannelRead(readBuf.get(i));<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

最终pipeline中的ChannelHandler(ServerBootstrapAcceptor)会响应ChannelRead事件，并在相应回调函数中初始化客户端NioSocketChannel，并将其注册到Sub Reactor Group中。此后客户端NioSocketChannel绑定到的sub reactor就开始监听处理客户端连接上的读写事件了。

Netty整个接收客户端的逻辑过程如下图步骤1，2，3所示。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

netty中的reactor.png

以上内容就是笔者提取出来的整体流程框架，下面我们来将其中涉及到的重要核心模块拆开，一个一个详细解读下。

## 3\. RecvByteBufAllocator简介

Reactor在处理对应Channel上的IO数据时，都会采用一个`ByteBuffer`来接收Channel上的IO数据。而本小节要介绍的RecvByteBufAllocator正是用来分配ByteBuffer的一个分配器。

还记得这个`RecvByteBufAllocator`在哪里被创建的吗？？

在[?《聊聊Netty那些事儿之Reactor在Netty中的实现(创建篇)》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483907&idx=1&sn=084c470a8fe6234c2c9461b5f713ff30&chksm=ce77c444f9004d52e7c6244bee83479070effb0bc59236df071f4d62e91e25f01715fca53696&scene=21#wechat_redirect)一文中，在介绍`NioServerSocketChannel`的创建过程中提到，对应Channel的配置类NioServerSocketChannelConfig也会随着NioServerSocketChannel的创建而创建。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>NioServerSocketChannel</span><span>(ServerSocketChannel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>super</span>(<span>null</span>,&nbsp;channel,&nbsp;SelectionKey.OP_ACCEPT);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;config&nbsp;=&nbsp;<span>new</span>&nbsp;NioServerSocketChannelConfig(<span>this</span>,&nbsp;javaChannel().socket());<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

在创建`NioServerSocketChannelConfig`的过程中会创建`RecvByteBufAllocator`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>DefaultChannelConfig</span><span>(Channel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>this</span>(channel,&nbsp;<span>new</span>&nbsp;AdaptiveRecvByteBufAllocator());<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

这里我们看到NioServerSocketChannel中的RecvByteBufAllocator实际类型为`AdaptiveRecvByteBufAllocator`，顾名思义，这个类型的RecvByteBufAllocator可以根据Channel上每次到来的IO数据大小来自适应动态调整ByteBuffer的容量。

对于服务端NioServerSocketChannel来说，它上边的IO数据就是客户端的连接，它的长度和类型都是固定的，所以在接收客户端连接的时候并不需要这样的一个ByteBuffer来接收，我们会将接收到的客户端连接存放在`List<Object> readBuf`集合中

对于客户端NioSocketChannel来说，它上边的IO数据时客户端发送来的网络数据，长度是不定的，所以才会需要这样一个可以根据每次IO数据的大小来自适应动态调整容量的ByteBuffer来接收。

那么看起来这个RecvByteBufAllocator和本文的主题不是很关联，因为在接收连接的过程中并不会怎么用到它，这个类笔者还会在后面的文章中详细介绍，之所以这里把它拎出来单独介绍是因为它和本文开头提到的Bug有关系，这个Bug就是由这个类引起的。

### 3.1 RecvByteBufAllocator.Handle的获取

在本文中，我们是通过NioServerSocketChannel中的unsafe底层操作类来获取RecvByteBufAllocator.Handle的

```
<span></span><code><span>final</span>&nbsp;RecvByteBufAllocator.Handle&nbsp;allocHandle&nbsp;=&nbsp;unsafe().recvBufAllocHandle();<br></code>
```

```
<span></span><code><span>protected</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>AbstractUnsafe</span>&nbsp;<span>implements</span>&nbsp;<span>Unsafe</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>public</span>&nbsp;RecvByteBufAllocator.<span>Handle&nbsp;<span>recvBufAllocHandle</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(recvHandle&nbsp;==&nbsp;<span>null</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;recvHandle&nbsp;=&nbsp;config().getRecvByteBufAllocator().newHandle();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;recvHandle;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

我们看到最终会在NioServerSocketChannel的配置类NioServerSocketChannelConfig中获取到`AdaptiveRecvByteBufAllocator`

```
<span></span><code><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>DefaultChannelConfig</span>&nbsp;<span>implements</span>&nbsp;<span>ChannelConfig</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;<span>//用于Channel接收数据用的buffer分配器&nbsp;&nbsp;类型为AdaptiveRecvByteBufAllocator</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>volatile</span>&nbsp;RecvByteBufAllocator&nbsp;rcvBufAllocator;<br>}<br></code>
```

`AdaptiveRecvByteBufAllocator`中会创建自适应动态调整容量的ByteBuffer分配器。

```
<span></span><code><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>AdaptiveRecvByteBufAllocator</span>&nbsp;<span>extends</span>&nbsp;<span>DefaultMaxMessagesRecvByteBufAllocator</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;Handle&nbsp;<span>newHandle</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;<span>new</span>&nbsp;HandleImpl(minIndex,&nbsp;maxIndex,&nbsp;initial);<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;<br>&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>HandleImpl</span>&nbsp;<span>extends</span>&nbsp;<span>MaxMessageHandle</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.................省略................<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

这里的`newHandle`方法返回的具体类型为`MaxMessageHandle`，这个`MaxMessageHandle`里边保存了每次从`Channel`中读取`IO数据`的容量指标，方便下次读取时分配合适大小的`buffer`。

每次在使用`allocHandle`前需要调用`allocHandle.reset(config);`重置里边的统计指标。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span>public</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>MaxMessageHandle</span>&nbsp;<span>implements</span>&nbsp;<span>ExtendedHandle</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;ChannelConfig&nbsp;config;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//每次事件轮询时，最多读取16次</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>int</span>&nbsp;maxMessagePerRead;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//本次事件轮询总共读取的message数,这里指的是接收连接的数量</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>int</span>&nbsp;totalMessages;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//本次事件轮询总共读取的字节数</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>int</span>&nbsp;totalBytesRead;<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>reset</span><span>(ChannelConfig&nbsp;config)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>this</span>.config&nbsp;=&nbsp;config;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//默认每次最多读取16次</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;maxMessagePerRead&nbsp;=&nbsp;maxMessagesPerRead();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;totalMessages&nbsp;=&nbsp;totalBytesRead&nbsp;=&nbsp;<span>0</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

- **maxMessagePerRead**：用于控制每次read loop里最大可以循环读取的次数，默认为16次，可在启动配置类`ServerBootstrap`中通过`ChannelOption.MAX_MESSAGES_PER_READ`选项设置。

```
<span></span><code>ServerBootstrap&nbsp;b&nbsp;=&nbsp;<span>new</span>&nbsp;ServerBootstrap();<br>b.group(bossGroup,&nbsp;workerGroup)<br>&nbsp;&nbsp;.channel(NioServerSocketChannel<span>.<span>class</span>)<br>&nbsp;&nbsp;.<span>option</span>(<span>ChannelOption</span>.<span>MAX_MESSAGES_PER_READ</span>,&nbsp;自定义次数)<br></span></code>
```

- **totalMessages**：用于统计read loop中总共接收的连接个数，每次read loop循环后会调用`allocHandle.incMessagesRead`增加记录接收到的连接个数。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>final</span>&nbsp;<span>void</span>&nbsp;<span>incMessagesRead</span><span>(<span>int</span>&nbsp;amt)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;totalMessages&nbsp;+=&nbsp;amt;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

- **totalBytesRead**：用于统计在read loop中总共接收到客户端连接上的数据大小，这个字段主要用于sub reactor在接收客户端NioSocketChannel上的网络数据用的，本文我们介绍的是main reactor接收客户端连接，所以这里并不会用到这个字段。这个字段会在sub reactor每次读取完NioSocketChannel上的网络数据时增加记录。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>lastBytesRead</span><span>(<span>int</span>&nbsp;bytes)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;lastBytesRead&nbsp;=&nbsp;bytes;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(bytes&nbsp;&gt;&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;totalBytesRead&nbsp;+=&nbsp;bytes;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br></code>
```

MaxMessageHandler中还有一个非常重要的方法就是在每次read loop末尾会调用`allocHandle.continueReading()`方法来判断读取连接次数是否已满16次，来决定main reactor线程是否退出循环。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>do</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//底层调用NioServerSocketChannel-&gt;doReadMessages&nbsp;创建客户端SocketChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;localRead&nbsp;=&nbsp;doReadMessages(readBuf);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(localRead&nbsp;==&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>break</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(localRead&nbsp;&lt;&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;closed&nbsp;=&nbsp;<span>true</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>break</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//统计在当前事件循环中已经读取到得Message数量（创建连接的个数）</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;allocHandle.incMessagesRead(localRead);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>while</span>&nbsp;(allocHandle.continueReading());<br></code>
```

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

红框中圈出来的两个判断条件和本文主题无关，我们这里不需要关注，笔者会在后面的文章详细介绍。

- `totalMessages < maxMessagePerRead`：在本文的接收客户端连接场景中，这个条件用于判断main reactor线程在read loop中的读取次数是否超过了16次。如果超过16次就会返回false，main reactor线程退出循环。

- `totalBytesRead > 0`：用于判断当客户端NioSocketChannel上的OP\_READ事件活跃时，sub reactor线程在read loop中是否读取到了网络数据。

以上内容就是RecvByteBufAllocator.Handle在接收客户端连接场景下的作用，大家这里仔细看下这个`allocHandle.continueReading()`方法退出循环的判断条件，再结合整个`do{....}while(...)`接收连接循环体，感受下是否哪里有些不对劲？Bug即将出现~~~

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

## 4\. 啊哈！！Bug !

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

netty不论是在本文中处理接收客户端连接的场景还是在处理接收客户端连接上的网络数据场景都会在一个`do{....}while(...)`循环read loop中不断的处理。

同时也都会利用在上一小节中介绍的`RecvByteBufAllocator.Handle`来记录每次read loop接收到的连接个数和从连接上读取到的网络数据大小。

从而在read loop的末尾都会通过`allocHandle.continueReading()`方法判断是否应该退出read loop循环结束连接的接收流程或者是结束连接上数据的读取流程。

无论是用于接收客户端连接的main reactor也好还是用于接收客户端连接上的网络数据的sub reactor也好，它们的运行框架都是一样的，只不过是具体分工不同。

所以netty这里想用统一的`RecvByteBufAllocator.Handle`来处理以上两种场景。

而`RecvByteBufAllocator.Handle`中的`totalBytesRead`字段主要记录sub reactor线程在处理客户端NioSocketChannel中OP\_READ事件活跃时，总共在read loop中读取到的网络数据，而这里是main reactor线程在接收客户端连接所以这个字段并不会被设置。totalBytesRead字段的值在本文中永远会是`0`。

所以无论同时有多少个客户端并发连接到服务端上，在接收连接的这个read loop中永远只会接受一个连接就会退出循环，因为`allocHandle.continueReading()方法`中的判断条件`totalBytesRead > 0`永远会返回`false`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>do</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//底层调用NioServerSocketChannel-&gt;doReadMessages&nbsp;创建客户端SocketChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;localRead&nbsp;=&nbsp;doReadMessages(readBuf);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(localRead&nbsp;==&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>break</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(localRead&nbsp;&lt;&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;closed&nbsp;=&nbsp;<span>true</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>break</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//统计在当前事件循环中已经读取到得Message数量（创建连接的个数）</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;allocHandle.incMessagesRead(localRead);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>while</span>&nbsp;(allocHandle.continueReading());<br></code>
```

**而netty的本意是在这个read loop循环中尽可能多的去接收客户端的并发连接，同时又不影响main reactor线程执行异步任务。但是由于这个Bug，main reactor在这个循环中只执行一次就结束了。这也一定程度上就影响了netty的吞吐**。

让我们想象下这样的一个场景，当有16个客户端同时并发连接到了服务端，这时NioServerSocketChannel上的`OP_ACCEPT事件`活跃，main reactor从Selector上被唤醒，随后执行`OP_ACCEPT事件`的处理。

```
<span></span><code><span>public</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>NioEventLoop</span>&nbsp;<span>extends</span>&nbsp;<span>SingleThreadEventLoop</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>void</span>&nbsp;<span>run</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;selectCnt&nbsp;=&nbsp;<span>0</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>for</span>&nbsp;(;;)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{&nbsp;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;strategy;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;strategy&nbsp;=&nbsp;selectStrategy.calculateStrategy(selectNowSupplier,&nbsp;hasTasks());<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>switch</span>&nbsp;(strategy)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>case</span>&nbsp;SelectStrategy.CONTINUE:&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;............省略.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>case</span>&nbsp;SelectStrategy.BUSY_WAIT:<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;............省略.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>case</span>&nbsp;SelectStrategy.SELECT:<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;............监听轮询IO事件.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>default</span>:<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(IOException&nbsp;e)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;............省略.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;............处理IO就绪事件.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;............执行异步任务.........<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

但是由于这个Bug的存在，main reactor在接收客户端连接的这个read loop中只接收了一个客户端连接就匆匆返回了。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>NioMessageUnsafe</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioUnsafe</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>do</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;localRead&nbsp;=&nbsp;doReadMessages(readBuf);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.........省略...........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>while</span>&nbsp;(allocHandle.continueReading());<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

然后根据下图中这个Reactor的运行结构去执行异步任务，随后绕一大圈又会回到`NioEventLoop#run`方法中重新发起一轮OP\_ACCEPT事件轮询。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Reactor线程运行时结构.png

由于现在还有15个客户端并发连接没有被接收，所以此时Main Reactor线程并不会在`selector.select()`上阻塞，最终绕一圈又会回到`NioMessageUnsafe#read`方法的`do{.....}while()`循环。在接收一个连接之后又退出循环。

本来我们可以在一次read loop中把这16个并发的客户端连接全部接收完毕的，因为这个Bug，main reactor需要不断的发起OP\_ACCEPT事件的轮询，绕了很大一个圈子。**同时也增加了许多不必要的selector.select()系统调用开销**

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

issue讨论.png

这时大家在看这个?Issue#11708中的讨论是不是就清晰很多了~~

> Issue#11708：<https://github.com/netty/netty/issues/11708>

### 4.1 Bug的修复

> 笔者在写这篇文章的时候，Netty最新版本是4.1.68.final，这个Bug在4.1.69.final中被修复。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

由于该Bug产生的原因正是因为服务端NioServerSocketChannel（用于监听端口地址和接收客户端连接）和 客户端NioSocketChannel（用于通信）中的Config配置类混用了同一个ByteBuffer分配器`AdaptiveRecvByteBufAllocator`而导致的。

所以在新版本修复中专门为服务端ServerSocketChannel中的Config配置类引入了一个新的ByteBuffer分配器`ServerChannelRecvByteBufAllocator`，专门用于服务端ServerSocketChannel接收客户端连接的场景。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

在`ServerChannelRecvByteBufAllocator`的父类`DefaultMaxMessagesRecvByteBufAllocator`中引入了一个新的字段`ignoreBytesRead`，用于表示是否忽略网络字节的读取，在创建服务端Channel配置类NioServerSocketChannelConfig的时候，这个字段会被赋值为`true`。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

当main reactor线程在read loop循环中接收客户端连接的时候。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>NioMessageUnsafe</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioUnsafe</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>do</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;localRead&nbsp;=&nbsp;doReadMessages(readBuf);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.........省略...........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>while</span>&nbsp;(allocHandle.continueReading());<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

在read loop循环的末尾就会采用从`ServerChannelRecvByteBufAllocator`中创建的`MaxMessageHandle#continueReading`方法来判断读取连接次数是否超过了16次。由于这里的`ignoreBytesRead == true`这回我们就会忽略`totalBytesRead == 0`的情况，从而使得接收连接的read loop得以继续地执行下去。在一个read loop中一次性把16个连接全部接收完毕。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

以上就是对这个Bug产生的原因，以及发现的过程，最后修复的方案一个全面的介绍，因此笔者也出现在了netty 4.1.69.final版本发布公告里的thank-list中。哈哈，真是令人开心的一件事情~~~

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

通过以上对netty接收客户端连接的全流程分析和对这个Bug来龙去脉以及修复方案的介绍，大家现在一定已经理解了整个接收连接的流程框架。

接下来笔者就把这个流程中涉及到的一些核心模块在单独拎出来从细节入手，为大家各个击破~~~

## 5\. doReadMessages接收客户端连接

```
<span></span><code><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>NioServerSocketChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioMessageChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>implements</span>&nbsp;<span>io</span>.<span>netty</span>.<span>channel</span>.<span>socket</span>.<span>ServerSocketChannel</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>int</span>&nbsp;<span>doReadMessages</span><span>(List&lt;Object&gt;&nbsp;buf)</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SocketChannel&nbsp;ch&nbsp;=&nbsp;SocketUtils.accept(javaChannel());<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(ch&nbsp;!=&nbsp;<span>null</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;buf.add(<span>new</span>&nbsp;NioSocketChannel(<span>this</span>,&nbsp;ch));<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;<span>1</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(Throwable&nbsp;t)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;logger.warn(<span>"Failed&nbsp;to&nbsp;create&nbsp;a&nbsp;new&nbsp;channel&nbsp;from&nbsp;an&nbsp;accepted&nbsp;socket."</span>,&nbsp;t);<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ch.close();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(Throwable&nbsp;t2)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;logger.warn(<span>"Failed&nbsp;to&nbsp;close&nbsp;a&nbsp;socket."</span>,&nbsp;t2);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;<span>0</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br></code>
```

- 通过`javaChannel()`获取封装在Netty服务端`NioServerSocketChannel`中的`JDK 原生 ServerSocketChannel`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;ServerSocketChannel&nbsp;<span>javaChannel</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;(ServerSocketChannel)&nbsp;<span>super</span>.javaChannel();<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

- 通过`JDK NIO 原生`的`ServerSocketChannel`的`accept方法`获取`JDK NIO 原生`客户端连接`SocketChannel`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>static</span>&nbsp;SocketChannel&nbsp;<span>accept</span><span>(<span>final</span>&nbsp;ServerSocketChannel&nbsp;serverSocketChannel)</span>&nbsp;<span>throws</span>&nbsp;IOException&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;AccessController.doPrivileged(<span>new</span>&nbsp;PrivilegedExceptionAction&lt;SocketChannel&gt;()&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;SocketChannel&nbsp;<span>run</span><span>()</span>&nbsp;<span>throws</span>&nbsp;IOException&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;serverSocketChannel.accept();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(PrivilegedActionException&nbsp;e)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>throw</span>&nbsp;(IOException)&nbsp;e.getCause();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

这一步就是我们在[?《聊聊Netty那些事儿之从内核角度看IO模型》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483737&idx=1&sn=7ef3afbb54289c6e839eed724bb8a9d6&chksm=ce77c71ef9004e08e3d164561e3a2708fc210c05408fa41f7fe338d8e85f39c1ad57519b614e&scene=21#wechat_redirect)介绍到的调用`监听Socket`的`accept方法`，内核会基于`监听Socket`创建出来一个新的`Socket`专门用于与客户端之间的网络通信这个我们称之为`客户端连接Socket`。这里的`ServerSocketChannel`就类似于`监听Socket`。`SocketChannel`就类似于`客户端连接Socket`。

由于我们在创建`NioServerSocketChannel`的时候，会将`JDK NIO 原生`的`ServerSocketChannel`设置为`非阻塞`，所以这里当`ServerSocketChannel`上有客户端连接时就会直接创建`SocketChannel`，如果此时并没有客户端连接时`accept调用`就会立刻返回`null`并不会阻塞。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>AbstractNioChannel</span><span>(Channel&nbsp;parent,&nbsp;SelectableChannel&nbsp;ch,&nbsp;<span>int</span>&nbsp;readInterestOp)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>super</span>(parent);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>this</span>.ch&nbsp;=&nbsp;ch;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>this</span>.readInterestOp&nbsp;=&nbsp;readInterestOp;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//设置Channel为非阻塞&nbsp;配合IO多路复用模型</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ch.configureBlocking(<span>false</span>);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(IOException&nbsp;e)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;..........省略.............<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

### 5.1 创建客户端NioSocketChannel

```
<span></span><code><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>NioServerSocketChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioMessageChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>implements</span>&nbsp;<span>io</span>.<span>netty</span>.<span>channel</span>.<span>socket</span>.<span>ServerSocketChannel</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>int</span>&nbsp;<span>doReadMessages</span><span>(List&lt;Object&gt;&nbsp;buf)</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SocketChannel&nbsp;ch&nbsp;=&nbsp;SocketUtils.accept(javaChannel());<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(ch&nbsp;!=&nbsp;<span>null</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;buf.add(<span>new</span>&nbsp;NioSocketChannel(<span>this</span>,&nbsp;ch));<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;<span>1</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(Throwable&nbsp;t)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.........省略.......<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;<span>0</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br></code>
```

这里会根据`ServerSocketChannel`的`accept`方法获取到`JDK NIO 原生`的`SocketChannel`（用于底层真正与客户端通信的Channel），来创建Netty中的`NioSocketChannel`。

```
<span></span><code><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>NioSocketChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioByteChannel</span>&nbsp;<span>implements</span>&nbsp;<span>io</span>.<span>netty</span>.<span>channel</span>.<span>socket</span>.<span>SocketChannel</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>NioSocketChannel</span><span>(Channel&nbsp;parent,&nbsp;SocketChannel&nbsp;socket)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>super</span>(parent,&nbsp;socket);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;config&nbsp;=&nbsp;<span>new</span>&nbsp;NioSocketChannelConfig(<span>this</span>,&nbsp;socket.socket());<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br></code>
```

创建客户端`NioSocketChannel`的过程其实和之前讲的创建服务端`NioServerSocketChannel`大体流程是一样的，我们这里只对客户端`NioSocketChannel`和服务端`NioServerSocketChannel`在创建过程中的不同之处做一个对比。

> 具体细节部分大家可以在回看下[?《详细图解Netty Reactor启动全流程》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484005&idx=1&sn=52f51269902a58f40d33208421109bc3&chksm=ce77c422f9004d340e5b385ef6ba24dfba1f802076ace80ad6390e934173a10401e64e13eaeb&scene=21#wechat_redirect)一文中关于`NioServerSocketChannel`的创建的详细细节。

### 5.3 对比NioSocketChannel与NioServerSocketChannel的不同

#### 1：Channel的层次不同

在我们介绍Reactor的创建文章中，我们提到Netty中的`Channel`是具有层次的。由于客户端NioSocketChannel是在main reactor接收连接时在服务端NioServerSocketChannel中被创建的，所以在创建客户端NioSocketChannel的时候会通过构造函数指定了parent属性为`NioServerSocketChanel`。并将`JDK NIO 原生`的`SocketChannel`封装进Netty的客户端`NioSocketChannel`中。

而在Reactor启动过程中创建`NioServerSocketChannel`的时候`parent属性`指定是`null`。因为它就是顶层的`Channel`，负责创建客户端`NioSocketChannel`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>NioServerSocketChannel</span><span>(ServerSocketChannel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>super</span>(<span>null</span>,&nbsp;channel,&nbsp;SelectionKey.OP_ACCEPT);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;config&nbsp;=&nbsp;<span>new</span>&nbsp;NioServerSocketChannelConfig(<span>this</span>,&nbsp;javaChannel().socket());<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

#### 2：向Reactor注册的IO事件不同

客户端NioSocketChannel向Sub Reactor注册的是`SelectionKey.OP_READ事件`，而服务端NioServerSocketChannel向Main Reactor注册的是`SelectionKey.OP_ACCEPT事件`。

```
<span></span><code><span>public</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>AbstractNioByteChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioChannel</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>AbstractNioByteChannel</span><span>(Channel&nbsp;parent,&nbsp;SelectableChannel&nbsp;ch)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>super</span>(parent,&nbsp;ch,&nbsp;SelectionKey.OP_READ);<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br><br><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>NioServerSocketChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioMessageChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>implements</span>&nbsp;<span>io</span>.<span>netty</span>.<span>channel</span>.<span>socket</span>.<span>ServerSocketChannel</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>NioServerSocketChannel</span><span>(ServerSocketChannel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//父类AbstractNioChannel中保存JDK&nbsp;NIO原生ServerSocketChannel以及要监听的事件OP_ACCEPT</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>super</span>(<span>null</span>,&nbsp;channel,&nbsp;SelectionKey.OP_ACCEPT);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//DefaultChannelConfig中设置用于Channel接收数据用的buffer-&gt;AdaptiveRecvByteBufAllocator</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;config&nbsp;=&nbsp;<span>new</span>&nbsp;NioServerSocketChannelConfig(<span>this</span>,&nbsp;javaChannel().socket());<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

#### 3: 功能属性不同造成继承结构的不同

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

NioSocketChannel.png

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

NioServerSocketChannel.png

客户端`NioSocketChannel`继承的是`AbstractNioByteChannel`，而服务端`NioServerSocketChannel`继承的是`AbstractNioMessageChannel`。它们继承的这两个抽象类一个前缀是`Byte`，一个前缀是`Message`有什么区别吗？？

> 客户端`NioSocketChannel`主要处理的是服务端与客户端的通信，这里涉及到接收客户端发送来的数据，而`Sub Reactor线程`从`NioSocketChannel`中读取的正是网络通信数据单位为`Byte`。

> 服务端`NioServerSocketChannel`主要负责处理`OP_ACCEPT事件`，创建用于通信的客户端`NioSocketChannel`。这时候客户端与服务端还没开始通信，所以`Main Reactor线程`从`NioServerSocketChannel`的读取对象为`Message`。这里的`Message`指的就是底层的`SocketChannel`客户端连接。

___

以上就是`NioSocketChannel`与`NioServerSocketChannel`创建过程中的不同之处，后面的过程就一样了。

- 在AbstractNioChannel 类中封装JDK NIO 原生的`SocketChannel`，并将其底层的IO模型设置为`非阻塞`，保存需要监听的IO事件`OP_READ`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>AbstractNioChannel</span><span>(Channel&nbsp;parent,&nbsp;SelectableChannel&nbsp;ch,&nbsp;<span>int</span>&nbsp;readInterestOp)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>super</span>(parent);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>this</span>.ch&nbsp;=&nbsp;ch;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>this</span>.readInterestOp&nbsp;=&nbsp;readInterestOp;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//设置Channel为非阻塞&nbsp;配合IO多路复用模型</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ch.configureBlocking(<span>false</span>);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(IOException&nbsp;e)&nbsp;{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

- 为客户端NioSocketChannel创建全局唯一的`channelId`，创建客户端NioSocketChannel的底层操作类`NioByteUnsafe`，创建pipeline。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>AbstractChannel</span><span>(Channel&nbsp;parent)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>this</span>.parent&nbsp;=&nbsp;parent;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//channel全局唯一ID&nbsp;machineId+processId+sequence+timestamp+random</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;id&nbsp;=&nbsp;newId();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//unsafe用于底层socket的读写操作</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unsafe&nbsp;=&nbsp;newUnsafe();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//为channel分配独立的pipeline用于IO事件编排</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline&nbsp;=&nbsp;newChannelPipeline();<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

- 在NioSocketChannelConfig的创建过程中，将NioSocketChannel的RecvByteBufAllocator类型设置为`AdaptiveRecvByteBufAllocator`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>DefaultChannelConfig</span><span>(Channel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>this</span>(channel,&nbsp;<span>new</span>&nbsp;AdaptiveRecvByteBufAllocator());<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

> 在Bug修复后的版本中服务端NioServerSocketChannel的RecvByteBufAllocator类型设置为`ServerChannelRecvByteBufAllocator`

最终我们得到的客户端`NioSocketChannel`结构如下：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

NioSocketChannel.png

## 6\. ChannelRead事件的响应

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

接收客户端连接.png

在前边介绍接收连接的整体核心流程框架的时候，我们提到main reactor线程是在一个`do{.....}while(...)`循环read loop中不断的调用`ServerSocketChannel#accept`方法来接收客户端的连接。

当满足退出read loop循环的条件有两个：

1. 在限定的16次读取中，已经没有新的客户端连接要接收了。退出循环。

2. 从NioServerSocketChannel中读取客户端连接的次数达到了16次，无论此时是否还有客户端连接都需要退出循环。

main reactor就会退出read loop循环，此时接收到的客户端连接NioSocketChannel暂存与`List<Object> readBuf`集合中。

```
<span></span><code><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>NioMessageUnsafe</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioUnsafe</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;List&lt;Object&gt;&nbsp;readBuf&nbsp;=&nbsp;<span>new</span>&nbsp;ArrayList&lt;Object&gt;();<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>read</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>do</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;........省略.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//底层调用NioServerSocketChannel-&gt;doReadMessages&nbsp;创建客户端SocketChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;localRead&nbsp;=&nbsp;doReadMessages(readBuf);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;........省略.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;allocHandle.incMessagesRead(localRead);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>while</span>&nbsp;(allocHandle.continueReading());<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(Throwable&nbsp;t)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;exception&nbsp;=&nbsp;t;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>int</span>&nbsp;size&nbsp;=&nbsp;readBuf.size();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>for</span>&nbsp;(<span>int</span>&nbsp;i&nbsp;=&nbsp;<span>0</span>;&nbsp;i&nbsp;&lt;&nbsp;size;&nbsp;i&nbsp;++)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;readPending&nbsp;=&nbsp;<span>false</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline.fireChannelRead(readBuf.get(i));<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;........省略.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>finally</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;........省略.........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

随后main reactor线程会遍历`List<Object> readBuf`集合中的NioSocketChannel，并在NioServerSocketChannel的pipeline中传播ChannelRead事件。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

传播ChannelRead事件.png

最终`ChannelRead事件`会传播到`ServerBootstrapAcceptor`中，这里正是Netty处理客户端连接的核心逻辑所在。

`ServerBootstrapAcceptor`主要的作用就是初始化客户端`NioSocketChannel`，并将客户端NioSocketChannel注册到`Sub Reactor Group`中，并监听`OP_READ事件`。

在ServerBootstrapAcceptor 中会初始化客户端NioSocketChannel的这些属性。

比如：从Reactor组`EventLoopGroup childGroup`，用于初始化`NioSocketChannel`中的`pipeline`用到的`ChannelHandler childHandler`，以及`NioSocketChannel`中的一些`childOptions`和`childAttrs`。

```
<span></span><code><span>private</span>&nbsp;<span>static</span>&nbsp;<span><span>class</span>&nbsp;<span>ServerBootstrapAcceptor</span>&nbsp;<span>extends</span>&nbsp;<span>ChannelInboundHandlerAdapter</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;EventLoopGroup&nbsp;childGroup;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;ChannelHandler&nbsp;childHandler;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;Entry&lt;ChannelOption&lt;?&gt;,&nbsp;Object&gt;[]&nbsp;childOptions;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>private</span>&nbsp;<span>final</span>&nbsp;Entry&lt;AttributeKey&lt;?&gt;,&nbsp;Object&gt;[]&nbsp;childAttrs;<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@SuppressWarnings</span>(<span>"unchecked"</span>)<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>channelRead</span><span>(ChannelHandlerContext&nbsp;ctx,&nbsp;Object&nbsp;msg)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;Channel&nbsp;child&nbsp;=&nbsp;(Channel)&nbsp;msg;<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//向客户端NioSocketChannel的pipeline中</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//添加在启动配置类ServerBootstrap中配置的ChannelHandler</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;child.pipeline().addLast(childHandler);<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//利用配置的属性初始化客户端NioSocketChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;setChannelOptions(child,&nbsp;childOptions,&nbsp;logger);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;setAttributes(child,&nbsp;childAttrs);<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>/**<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;* 1：在Sub Reactor线程组中选择一个Reactor绑定<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;* 2：将客户端SocketChannel注册到绑定的Reactor上<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;* 3：SocketChannel注册到sub reactor中的selector上，并监听OP_READ事件<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*&nbsp;*/</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;childGroup.register(child).addListener(<span>new</span>&nbsp;ChannelFutureListener()&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>operationComplete</span><span>(ChannelFuture&nbsp;future)</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(!future.isSuccess())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;forceClose(child,&nbsp;future.cause());<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(Throwable&nbsp;t)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;forceClose(child,&nbsp;t);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

正是在这里，netty会将我们在[?《详细图解Netty Reactor启动全流程》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484005&idx=1&sn=52f51269902a58f40d33208421109bc3&chksm=ce77c422f9004d340e5b385ef6ba24dfba1f802076ace80ad6390e934173a10401e64e13eaeb&scene=21#wechat_redirect)的启动示例程序中在ServerBootstrap中配置的客户端NioSocketChannel的所有属性（child前缀配置）初始化到NioSocketChannel中。

```
<span></span><code><span>public</span>&nbsp;<span>final</span>&nbsp;<span><span>class</span>&nbsp;<span>EchoServer</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;<span>static</span>&nbsp;<span>final</span>&nbsp;<span>int</span>&nbsp;PORT&nbsp;=&nbsp;Integer.parseInt(System.getProperty(<span>"port"</span>,&nbsp;<span>"8007"</span>));<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>static</span>&nbsp;<span>void</span>&nbsp;<span>main</span><span>(String[]&nbsp;args)</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//&nbsp;Configure&nbsp;the&nbsp;server.</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//创建主从Reactor线程组</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;EventLoopGroup&nbsp;bossGroup&nbsp;=&nbsp;<span>new</span>&nbsp;NioEventLoopGroup(<span>1</span>);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;EventLoopGroup&nbsp;workerGroup&nbsp;=&nbsp;<span>new</span>&nbsp;NioEventLoopGroup();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;EchoServerHandler&nbsp;serverHandler&nbsp;=&nbsp;<span>new</span>&nbsp;EchoServerHandler();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ServerBootstrap&nbsp;b&nbsp;=&nbsp;<span>new</span>&nbsp;ServerBootstrap();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b.group(bossGroup,&nbsp;workerGroup)<span>//配置主从Reactor</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.channel(NioServerSocketChannel<span>.<span>class</span>)//配置主<span>Reactor</span>中的<span>channel</span>类型<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.<span>option</span>(<span>ChannelOption</span>.<span>SO_BACKLOG</span>,&nbsp;100)//设置主<span>Reactor</span>中<span>channel</span>的<span>option</span>选项<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.<span>handler</span>(<span>new</span>&nbsp;<span>LoggingHandler</span>(<span>LogLevel</span>.<span>INFO</span>))//设置主<span>Reactor</span>中<span>Channel</span>-&gt;<span>pipline</span>-&gt;<span>handler</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.<span>childHandler</span>(<span>new</span>&nbsp;<span>ChannelInitializer</span>&lt;<span>SocketChannel</span>&gt;()&nbsp;</span>{<span>//设置从Reactor中注册channel的pipeline</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>initChannel</span><span>(SocketChannel&nbsp;ch)</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ChannelPipeline&nbsp;p&nbsp;=&nbsp;ch.pipeline();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//p.addLast(new&nbsp;LoggingHandler(LogLevel.INFO));</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;p.addLast(serverHandler);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//&nbsp;Start&nbsp;the&nbsp;server.&nbsp;绑定端口启动服务，开始监听accept事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ChannelFuture&nbsp;f&nbsp;=&nbsp;b.bind(PORT).sync();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//&nbsp;Wait&nbsp;until&nbsp;the&nbsp;server&nbsp;socket&nbsp;is&nbsp;closed.</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;f.channel().closeFuture().sync();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>finally</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//&nbsp;Shut&nbsp;down&nbsp;all&nbsp;event&nbsp;loops&nbsp;to&nbsp;terminate&nbsp;all&nbsp;threads.</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;bossGroup.shutdownGracefully();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;workerGroup.shutdownGracefully();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

以上示例代码中通过ServerBootstrap配置的NioSocketChannel相关属性，会在Netty启动并开始初始化`NioServerSocketChannel`的时候将`ServerBootstrapAcceptor`的创建初始化工作封装成`异步任务`，然后在`NioServerSocketChannel`注册到`Main Reactor`中成功后执行。

```
<span></span><code><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>ServerBootstrap</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractBootstrap</span>&lt;<span>ServerBootstrap</span>,&nbsp;<span>ServerChannel</span>&gt;&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>void</span>&nbsp;<span>init</span><span>(Channel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;................省略................<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;p.addLast(<span>new</span>&nbsp;ChannelInitializer&lt;Channel&gt;()&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>initChannel</span><span>(<span>final</span>&nbsp;Channel&nbsp;ch)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;ChannelPipeline&nbsp;pipeline&nbsp;=&nbsp;ch.pipeline();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;................省略................<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ch.eventLoop().execute(<span>new</span>&nbsp;Runnable()&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>run</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline.addLast(<span>new</span>&nbsp;ServerBootstrapAcceptor(<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ch,&nbsp;currentChildGroup,&nbsp;currentChildHandler,&nbsp;currentChildOptions,&nbsp;currentChildAttrs));<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

在经过`ServerBootstrapAccptor#chanelRead回调`的处理之后，此时客户端NioSocketChannel中pipeline的结构为：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

客户端channel pipeline初始结构.png

随后会将初始化好的客户端NioSocketChannel向Sub Reactor Group中注册，并监听`OP_READ事件`。

如下图中的步骤3所示：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

netty中的reactor.png

## 7\. 向SubReactorGroup中注册NioSocketChannel

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;childGroup.register(child).addListener(<span>new</span>&nbsp;ChannelFutureListener()&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>operationComplete</span><span>(ChannelFuture&nbsp;future)</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(!future.isSuccess())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;forceClose(child,&nbsp;future.cause());<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br></code>
```

客户端NioSocketChannel向Sub Reactor Group注册的流程完全和服务端NioServerSocketChannel向Main Reactor Group注册流程一样。

> 关于服务端NioServerSocketChannel的注册流程，笔者已经在[?《详细图解Netty Reactor启动全流程》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484005&idx=1&sn=52f51269902a58f40d33208421109bc3&chksm=ce77c422f9004d340e5b385ef6ba24dfba1f802076ace80ad6390e934173a10401e64e13eaeb&scene=21#wechat_redirect)一文中做出了详细的介绍，对相关细节感兴趣的同学可以在回看下。

这里笔者在带大家简要回顾下整个注册过程并着重区别对比客户端NioSocetChannel与服务端NioServerSocketChannel注册过程中不同的地方。

### 7.1 从Sub Reactor Group中选取一个Sub Reactor进行绑定

```
<span></span><code><span>public</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>MultithreadEventLoopGroup</span>&nbsp;<span>extends</span>&nbsp;<span>MultithreadEventExecutorGroup</span>&nbsp;<span>implements</span>&nbsp;<span>EventLoopGroup</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;ChannelFuture&nbsp;<span>register</span><span>(Channel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;next().register(channel);<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;EventExecutor&nbsp;<span>next</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;chooser.next();<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br></code>
```

### 7.2 向绑定的Sub Reactor上注册NioSocketChannel

```
<span></span><code><span>public</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>SingleThreadEventLoop</span>&nbsp;<span>extends</span>&nbsp;<span>SingleThreadEventExecutor</span>&nbsp;<span>implements</span>&nbsp;<span>EventLoop</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;ChannelFuture&nbsp;<span>register</span><span>(Channel&nbsp;channel)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//注册channel到绑定的Reactor上</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;register(<span>new</span>&nbsp;DefaultChannelPromise(channel,&nbsp;<span>this</span>));<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;ChannelFuture&nbsp;<span>register</span><span>(<span>final</span>&nbsp;ChannelPromise&nbsp;promise)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ObjectUtil.checkNotNull(promise,&nbsp;<span>"promise"</span>);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//unsafe负责channel底层的各种操作</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;promise.channel().unsafe().register(<span>this</span>,&nbsp;promise);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;promise;<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br></code>
```

- 当时我们在介绍`NioServerSocketChannel`的注册过程时，这里的`promise.channel()`为`NioServerSocketChannel`。底层的unsafe操作类为`NioMessageUnsafe`。

- 此时这里的`promise.channel()`为`NioSocketChannel`。底层的unsafe操作类为`NioByteUnsafe`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>final</span>&nbsp;<span>void</span>&nbsp;<span>register</span><span>(EventLoop&nbsp;eventLoop,&nbsp;<span>final</span>&nbsp;ChannelPromise&nbsp;promise)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;..............省略....................<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//此时这里的eventLoop为Sub&nbsp;Reactor</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AbstractChannel.<span>this</span>.eventLoop&nbsp;=&nbsp;eventLoop;<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>/**<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*&nbsp;执行channel注册的操作必须是Reactor线程来完成<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*&nbsp;1:&nbsp;如果当前执行线程是Reactor线程，则直接执行register0进行注册<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;* 2：如果当前执行线程是外部线程，则需要将register0注册操作&nbsp;封装程异步Task 由Reactor线程执行<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*&nbsp;*/</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(eventLoop.inEventLoop())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;register0(promise);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>else</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;eventLoop.execute(<span>new</span>&nbsp;Runnable()&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>run</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;register0(promise);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(Throwable&nbsp;t)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;..............省略....................<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

**注意此时传递进来的EventLoop eventLoop为Sub Reactor**。

**但此时的执行线程为`Main Reactor线程`，并不是Sub Reactor线程（此时还未启动）**。

所以这里的`eventLoop.inEventLoop()`返回的是`false`。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

image.png

在`else分支`中向绑定的Sub Reactor提交注册`NioSocketChannel`的任务。

> 当注册任务提交后，此时绑定的`Sub Reactor线程`启动。

### 7.3 register0

我们又来到了Channel注册的老地方`register0方法`。在[?《详细图解Netty Reactor启动全流程》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484005&idx=1&sn=52f51269902a58f40d33208421109bc3&chksm=ce77c422f9004d340e5b385ef6ba24dfba1f802076ace80ad6390e934173a10401e64e13eaeb&scene=21#wechat_redirect)中我们花了大量的篇幅介绍了这个方法。这里我们只对比`NioSocketChannel`与`NioServerSocketChannel`不同的地方。

```
<span></span><code>&nbsp;<span><span>private</span>&nbsp;<span>void</span>&nbsp;<span>register0</span><span>(ChannelPromise&nbsp;promise)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;................省略..................<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>boolean</span>&nbsp;firstRegistration&nbsp;=&nbsp;neverRegistered;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//执行真正的注册操作</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;doRegister();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//修改注册状态</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;neverRegistered&nbsp;=&nbsp;<span>false</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;registered&nbsp;=&nbsp;<span>true</span>;<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline.invokeHandlerAddedIfNeeded();<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(isActive())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(firstRegistration)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//触发channelActive事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline.fireChannelActive();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>else</span>&nbsp;<span>if</span>&nbsp;(config().isAutoRead())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;beginRead();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(Throwable&nbsp;t)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;................省略..................<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

这里 `doRegister()方法`将NioSocketChannel注册到Sub Reactor中的`Selector`上。

```
<span></span><code><span>public</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>AbstractNioChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractChannel</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>void</span>&nbsp;<span>doRegister</span><span>()</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>boolean</span>&nbsp;selected&nbsp;=&nbsp;<span>false</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>for</span>&nbsp;(;;)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>try</span>&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;selectionKey&nbsp;=&nbsp;javaChannel().register(eventLoop().unwrappedSelector(),&nbsp;<span>0</span>,&nbsp;<span>this</span>);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>;<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>catch</span>&nbsp;(CancelledKeyException&nbsp;e)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;...............省略...............<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br></code>
```

这里是Netty客户端`NioSocketChannel`与JDK NIO 原生 SocketChannel关联的地方。此时注册的`IO事件`依然是`0`。目的也是只是为了获取NioSocketChannel在Selector中的`SelectionKey`。

同时通过`SelectableChannel#register`方法将Netty自定义的NioSocketChannel（这里的this指针）附着在SelectionKey的attechment属性上，完成Netty自定义Channel与JDK NIO Channel的关系绑定。这样在每次对Selector进行IO就绪事件轮询时，Netty 都可以从 JDK NIO Selector返回的SelectionKey中获取到自定义的Channel对象（这里指的就是NioSocketChannel）。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

channel与SelectionKey对应关系.png

随后调用`pipeline.invokeHandlerAddedIfNeeded()`回调客户端NioSocketChannel上pipeline中的所有ChannelHandler的`handlerAdded方法`，此时`pipeline`的结构中只有一个`ChannelInitializer`。最终会在`ChannelInitializer#handlerAdded`回调方法中初始化客户端`NioSocketChannel`的`pipeline`。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

客户端channel pipeline初始结构.png

```
<span></span><code><span>public</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>ChannelInitializer</span>&lt;<span>C</span>&nbsp;<span>extends</span>&nbsp;<span>Channel</span>&gt;&nbsp;<span>extends</span>&nbsp;<span>ChannelInboundHandlerAdapter</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>handlerAdded</span><span>(ChannelHandlerContext&nbsp;ctx)</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(ctx.channel().isRegistered())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(initChannel(ctx))&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//初始化工作完成后，需要将自身从pipeline中移除</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;removeState(ctx);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>abstract</span>&nbsp;<span>void</span>&nbsp;<span>initChannel</span><span>(C&nbsp;ch)</span>&nbsp;<span>throws</span>&nbsp;Exception</span>;<br>}<br><br></code>
```

> 关于对Channel中pipeline的详细初始化过程，对细节部分感兴趣的同学可以回看下[?《详细图解Netty Reactor启动全流程》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247484005&idx=1&sn=52f51269902a58f40d33208421109bc3&chksm=ce77c422f9004d340e5b385ef6ba24dfba1f802076ace80ad6390e934173a10401e64e13eaeb&scene=21#wechat_redirect)

此时客户端NioSocketChannel中的pipeline中的结构就变为了我们自定义的样子，在示例代码中我们自定义的`ChannelHandler`为`EchoServerHandler`。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

客户端channel pipeline结构.png

```
<span></span><code><span>@Sharable</span><br><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>EchoServerHandler</span>&nbsp;<span>extends</span>&nbsp;<span>ChannelInboundHandlerAdapter</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>channelRead</span><span>(ChannelHandlerContext&nbsp;ctx,&nbsp;Object&nbsp;msg)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ctx.write(msg);<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>channelReadComplete</span><span>(ChannelHandlerContext&nbsp;ctx)</span>&nbsp;</span>{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ctx.flush();<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>exceptionCaught</span><span>(ChannelHandlerContext&nbsp;ctx,&nbsp;Throwable&nbsp;cause)</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//&nbsp;Close&nbsp;the&nbsp;connection&nbsp;when&nbsp;an&nbsp;exception&nbsp;is&nbsp;raised.</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;cause.printStackTrace();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ctx.close();<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

当客户端NioSocketChannel中的pipeline初始化完毕后，netty就开始调用`safeSetSuccess(promise)方法`回调`regFuture`中注册的`ChannelFutureListener`，通知客户端NioSocketChannel已经成功注册到Sub Reactor上了。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;childGroup.register(child).addListener(<span>new</span>&nbsp;ChannelFutureListener()&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>void</span>&nbsp;<span>operationComplete</span><span>(ChannelFuture&nbsp;future)</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(!future.isSuccess())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;forceClose(child,&nbsp;future.cause());<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;});<br></code>
```

> 在服务端NioServerSocketChannel注册的时候我们会在listener中向Main Reactor提交`bind绑定端口地址任务`。但是在`NioSocketChannel`注册的时候，只会在`listener`中处理一下注册失败的情况。

当Sub Reactor线程通知ChannelFutureListener注册成功之后，随后就会调用`pipeline.fireChannelRegistered()`在客户端NioSocketChannel的pipeline中传播`ChannelRegistered事件`。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

传播ChannelRegister事件.png

**这里笔者重点要强调下**，在之前介绍NioServerSocketChannel注册的时候，我们提到因为此时NioServerSocketChannel并未绑定端口地址，所以这时的NioServerSocketChannel并未激活，这里的`isActive()`返回`false`。`register0方法`直接返回。

> 服务端NioServerSocketChannel判断是否激活的标准为端口是否绑定成功。

```
<span></span><code><span>public</span>&nbsp;<span><span>class</span>&nbsp;<span>NioServerSocketChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractNioMessageChannel</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>implements</span>&nbsp;<span>io</span>.<span>netty</span>.<span>channel</span>.<span>socket</span>.<span>ServerSocketChannel</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>boolean</span>&nbsp;<span>isActive</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;isOpen()&nbsp;&amp;&amp;&nbsp;javaChannel().socket().isBound();<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br>}<br></code>
```

> 客户端`NioSocketChannel`判断是否激活的标准为是否处于`Connected状态`。那么显然这里肯定是处于`connected状态`的。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>public</span>&nbsp;<span>boolean</span>&nbsp;<span>isActive</span><span>()</span>&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SocketChannel&nbsp;ch&nbsp;=&nbsp;javaChannel();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>return</span>&nbsp;ch.isOpen()&nbsp;&amp;&amp;&nbsp;ch.isConnected();<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

`NioSocketChannel`已经处于`connected状态`，这里并不需要绑定端口，所以这里的`isActive()`返回`true`。

```
<span></span><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(isActive())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>/**<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*&nbsp;客户端SocketChannel注册成功后会走这里，在channelActive事件回调中注册OP_READ事件<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*&nbsp;*/</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;(firstRegistration)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//触发channelActive事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;pipeline.fireChannelActive();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;<span>else</span>&nbsp;<span>if</span>&nbsp;(config().isAutoRead())&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.......省略..........<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br></code>
```

最后调用`pipeline.fireChannelActive()`在NioSocketChannel中的pipeline传播`ChannelActive事件`，最终在`pipeline`的头结点`HeadContext`中响应并注册`OP_READ事件`到`Sub Reactor`中的`Selector`上。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

传播ChannelActive事件.png

```
<span></span><code><span>public</span>&nbsp;<span>abstract</span>&nbsp;<span><span>class</span>&nbsp;<span>AbstractNioChannel</span>&nbsp;<span>extends</span>&nbsp;<span>AbstractChannel</span>&nbsp;</span>{&nbsp;{<br><br>&nbsp;&nbsp;&nbsp;&nbsp;<span>@Override</span><br>&nbsp;&nbsp;&nbsp;&nbsp;<span><span>protected</span>&nbsp;<span>void</span>&nbsp;<span>doBeginRead</span><span>()</span>&nbsp;<span>throws</span>&nbsp;Exception&nbsp;</span>{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;..............省略................<br><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>final</span>&nbsp;<span>int</span>&nbsp;interestOps&nbsp;=&nbsp;selectionKey.interestOps();<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>/**<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;* 1：ServerSocketChannel 初始化时 readInterestOp设置的是OP_ACCEPT事件<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;* 2：SocketChannel 初始化时 readInterestOp设置的是OP_READ事件<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*&nbsp;*/</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>if</span>&nbsp;((interestOps&nbsp;&amp;&nbsp;readInterestOp)&nbsp;==&nbsp;<span>0</span>)&nbsp;{<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span>//注册监听OP_ACCEPT或者OP_READ事件</span><br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;selectionKey.interestOps(interestOps&nbsp;|&nbsp;readInterestOp);<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br>&nbsp;&nbsp;&nbsp;&nbsp;}<br><br>}<br></code>
```

> 注意这里的`readInterestOp`为客户端`NioSocketChannel`在初始化时设置的`OP_READ事件`。

___

到这里，Netty中的`Main Reactor`接收连接的整个流程，我们就介绍完了，此时Netty中主从Reactor组的结构就变为：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='<http://www.w3.org/2000/svg>' xmlns:xlink='<http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg> stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

主从Reactor组完整结构.png

## 总结

本文我们介绍了`NioServerSocketChannel`处理客户端连接事件的整个过程。

- 接收连接的整个处理框架。

- 影响Netty接收连接吞吐的Bug产生的原因，以及修复的方案。

- 创建并初始化客户端`NioSocketChannel`。

- 初始化`NioSocketChannel`中的`pipeline`。

- 客户端`NioSocketChannel`向`Sub Reactor`注册的过程

其中我们也对比了`NioServerSocketChannel`与`NioSocketChannel`在创建初始化以及后面向`Reactor`注册过程中的差异之处。

当客户端`NioSocketChannel`接收完毕并向`Sub Reactor`注册成功后，那么接下来`Sub Reactor`就开始监听注册其上的所有客户端`NioSocketChannel`的`OP_READ事件`，并等待客户端向服务端发送网络数据。

后面`Reactor`的主角就该变为`Sub Reactor`以及注册在其上的客户端`NioSocketChannel`了。

下篇文章，我们将会讨论Netty是如何接收网络数据的~~~~ 我们下篇文章见~~
