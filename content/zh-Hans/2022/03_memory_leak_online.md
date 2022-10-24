---
categories: Code
date: 2022-06-20T10:00:00Z
title: "记一次Golang服务内存泄漏问题的定位"
tags:
- Golang
- Microservice
- pprof
- RPC
---

## 一、背景：

有一段时间发现团队各个服务频繁地出现内存告警的现象，去stke上查看发现机器内存以非常恐怖的趋势在增长，怀疑存在内存泄漏的问题。

![](https://docimg3.docs.qq.com/image/KurOPjJKRMCzrgsidLrNWQ.png?w=1280&h=285.2954375467465)

## 二、定位问题：

由于检查代码毫无头绪，所以考虑使用pprof工具来定位内存泄漏问题。

pprof是golang自带的一种可视化的性能分析工具，可以捕捉到多维度的运行状态的数据。

服务使用的是trpc-go框架，trpc-go内置支持pprof，根据文档可以知道只要开启了admin服务，就可以使用服务的pprof；但是在trpc 4.0版本之后，admi已经默认不在集成pprof内存分析命令，所以如果想要进行内存分析，就需要在入口文件使用admin自行注册。

![](https://docimg7.docs.qq.com/image/2IimEUicZkHqpt2MTqY0nw.png?w=1278&h=1102)

### 使用pprof定位问题

将开启了内存分析的服务发布上线。主要分析堆栈（heap）和协程（goroutine）是否存在内存泄漏问题。

首先需要一台可以连上IDC的机子。根据以下指令下载并安装graphviz

```
wget https://graphviz.gitlab.io/pub/graphviz/stable/SOURCES/graphviz.tar.gz
./configure
make
make install
```

通过指令dot -version可以查看是否安装成功

![](https://docimg3.docs.qq.com/image/ijSBl16HlN9xEKO3NXiKEA.png?w=1280&h=384.84272128749086)

首先使用以下指令检查协程相关信息，在服务正常运行的情况下，每隔一段时间获取goroutine的数量，如果获取到的goroutine的数量一直在持续增长，则表示有可能是goroutine泄漏导致的内存泄漏。

```
go tool pprof http://9.215.112.131:9028/debug/pprof/goroutine
```

(ps.需要注意这里的端口号是trpc-go的配置文件trpc_go.yaml里注册的admin的端口号)

![](https://docimg8.docs.qq.com/image/EREyjJoRx_n6w6r6imqXXg.png?w=1082&h=473)

根据一段时间的观察后，发现goroutine的数目是正常的，暂时排除goroutine泄漏的可能。

接着检查heap的情况。使用以下命令可以检查heap的相关信息。

```
go tool pprof http://9.215.112.131:9028/debug/pprof/heap
```

使用`top`指令可以查看分配内存最多的函数调用

![](https://docimg9.docs.qq.com/image/XZFmo8oOxuLyzDZ6iqV4jA.png?w=1131&h=491)

从图里我们可以看到这个函数`net/textproto.(*Reader).ReadMIMEHeader`占用的内存最多。然后，我们可以使用“svg”命令将服务的调用链保存为svg图片，接着可以导出后，使用浏览器打开查看。可以每隔一段时间使用heap打印一张svg图用来观察内存的变化，如果有一个函数使用的内存在不断的增大，就可以考虑是否是该位置导致的内存泄漏。

![](https://docimg2.docs.qq.com/image/qflI0_Zep_7XCrpnj9fEPQ.png?w=1280&h=639.591054313099)

![](https://docimg10.docs.qq.com/image/0KIGBPYOgDW3Te4Iwqq-2g.png?w=1280&h=623.8082556591212)

从图中，可以看到函数`net/textproto.(*Reader).ReadMIMEHeader`占用的内存在不断增大。基本可以断定是这个地方导致的内存泄漏。

根据pprof定位的问题找到内存泄漏的原因

通过google找到类似问题的文章[记一次Go websocket项目内存泄露排查 + 使用Go pprof定位内存泄露](https://pathbox.github.io/2017/05/27/find-the-reason-memory-leak/)。怀疑是在使用

```
"github.com/gorilla/context" 
```

的时候，有一些地方没有clear，导致了这次的内存泄漏问题。

根据这篇文章的介绍，`"github.com/gorilla/context"`这个库定义了两个全局变量的map。 当每个request来的时候，就将往这两个全局map中以http.Request对象为key存值。如果没有调用`context.Clear(r)`方法。即使这个request是已经关闭了，但是，由于这个request作为了全局变量map的key,使得Go GC没法回收已经结束的request对象。内存就很快就被消耗完了。

找到原因后，就在代码中全局搜索使用`gorillaContext`的地方，发现虽然在middleware里我们在最后的时候调用了context.clear方法释放了内存。

![](https://docimg10.docs.qq.com/image/0wsawhDt5s3I84Emst7Ggw.png?w=1280&h=439.88732394366195)

![](https://docimg6.docs.qq.com/image/SV-oEld2LZ2Ej-3WUPwAzg.png?w=1280&h=636.5498652291105)

但是，发现了一个漏网之鱼，为了stke健康检查我们加上了一个http存活探针接口，该接口每5秒会请求一次，但是在注册该接口时没有走middleware，也就导致`gorillaContext`没有被释放。

![](https://docimg9.docs.qq.com/image/sLj03Eh3O1EP8AA73vTlaA.png?w=1126&h=404)

最后，修复问题并发布上线。

![](https://docimg2.docs.qq.com/image/nbhO44y3bKe7Ap8KnMbuiQ.png?w=1108&h=514)

发布之后再使用pprof查看内存使用情况，发现函数`(*Reader).ReadMIMEHeader`已经不在内存占用的Top10里面了，且内存也不再线性增长，证明该问题已经成功解决了。

![](https://docimg2.docs.qq.com/image/kK5uQ9TabSJO4eD7iuKh8A.png?w=1280&h=396.07547169811323)

![](https://docimg2.docs.qq.com/image/Mvnk7HbTFjAQKBggYJ9Bpw.png?w=1280&h=280.48192771084337)

## 三、总结：

要学会使用各种debug工具来定位问题，提高效率。

在写代码时对于使用了db、http client和全局变量（map，slice）等地方，要仔细检查，在正确的时候释放内存。
