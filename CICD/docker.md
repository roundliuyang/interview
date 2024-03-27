# docker



## Docker平时怎么使用的

会使用docker 构建后端服务（如java,go）、前端服务的镜像。

k8s或者docker swarm 编排这些服务



## Java 程序运行在 Docker 等容器环境有哪些新问题

对于 Java 来说，Docker 毕竟是一个较新的环境，例如，其内存、CPU 等资源限制是通过 CGroup（Control Group）实现的，早期的 JDK 版本（8u131 之前）并不能识别这些限制，进而会导致一些基础问题：

- 如果未配置合适的 JVM 堆和元数据区、直接内存等参数，Java 就有可能试图使用超过容器限制的内存，最终被容器 OOM kill，或者自身发生 OOM。
- 错误判断了可获取的 CPU 资源，例如，Docker 限制了 CPU 的核数，JVM 就可能设置不合适的 GC 并行线程数等。

从应用打包、发布等角度出发，JDK 自身就比较大，生成的镜像就更为臃肿，当我们的镜像非常多的时候，镜像的存储等开销就比较明显了。

如果考虑到微服务、Serverless 等新的架构和场景，Java 自身的大小、内存占用、启动速度，都存在一定局限性，因为 Java 早期的优化大多是针对长时间运行的大型服务器端应用。





## **从 JVM 运行机制的角度，为什么这些“沟通障碍”会导致 OOM 等问题呢**

这个问题实际是反映了 JVM 如何根据系统资源（内存、CPU 等）情况，在启动时设置默认参数。

这就是所谓的[Ergonomics](https://docs.oracle.com/javase/10/gctuning/ergonomics.htm#JSGCT-GUID-DB4CAE94-2041-4A16-90EC-6AE3D91EC1F1)机制，例如：

- JVM 会大概根据检测到的内存大小，设置最初启动时的堆大小为系统内存的 1/64；并将堆最大值，设置为系统内存的 1/4。
- 而 JVM 检测到系统的 CPU 核数，则直接影响到了 Parallel GC 的并行线程数目和 JIT complier 线程数目，甚至是我们应用中 ForkJoinPool 等机制的并行等级。

这些默认参数，是根据通用场景选择的初始值。但是由于容器环境的差异，Java 的判断很可能是基于错误信息而做出的。这就类似，我以为我住的是整栋别墅，实际上却只有一个房间是给我住的。

根据前面的总结，似乎问题非常棘手，那我们在实践中，**如何解决这些问题呢？**

首先，如果你能够**升级到最新的 JDK 版本**，这个问题就迎刃而解了。

- 针对这种情况，JDK 9 中引入了一些实验性的参数，以方便 Docker 和 Java“沟通”，例如针对内存限制，可以使用下面的参数设置：

```
-XX:+UnlockExperimentalVMOptions
-XX:+UseCGroupMemoryLimitForHeap
```

注意，这两个参数是顺序敏感的，并且只支持 Linux 环境。而对于 CPU 核心数限定，Java 已经被修正为可以正确理解“–cpuset-cpus”等设置，无需单独设置参数。

- 如果你可以切换到 JDK 10 或者更新的版本，问题就更加简单了。Java 对容器（Docker）的支持已经比较完善，默认就会自适应各种资源限制和实现差异。前面提到的实验性参数“UseCGroupMemoryLimitForHeap”已经被标记为废弃。

与此同时，新增了参数用以明确指定 CPU 核心的数目。

```
-XX:ActiveProcessorCount=N
```

如果实践中发现有问题，也可以使用“-XX:-UseContainerSupport”，关闭 Java 的容器支持特性，这可以作为一种防御性机制，避免新特性破坏原有基础功能。当然，也欢迎你向 OpenJDK 社区反馈问题。

- 幸运的是，JDK 9 中的实验性改进已经被移植到 Oracle JDK 8u131 之中，你可以直接下载相应[镜像](https://store.docker.com/images/oracle-serverjre-8)，并配置“UseCGroupMemoryLimitForHeap”，后续很有可能还会进一步将 JDK 10 中相关的增强，应用到 JDK 8 最新的更新中。

**但是，如果我暂时只能使用老版本的 JDK 怎么办？**

我这里有几个建议：

- 明确设置堆、元数据区等内存区域大小，保证 Java 进程的总大小可控。

例如，我们可能在环境中，这样限制容器内存：

```shell
$ docker run -it --rm --name yourcontainer -p 8080:8080 -m 800M repo/your-java-container:openjdk
```

那么，就可以额外配置下面的环境变量，直接指定 JVM 堆大小。

```shell
-e JAVA_OPTIONS='-Xmx300m'
```

- 明确配置 GC 和 JIT 并行线程数目，以避免二者占用过多计算资源。

```shell
-XX:ParallelGCThreads
-XX:CICompilerCount
```

除了我前面介绍的 OOM 等问题，在很多场景中还发现 Java 在 Docker 环境中，似乎会意外使用 Swap。具体原因待查，但很有可能也是因为 Ergonomics 机制失效导致的，我建议配置下面参数，明确告知 JVM 系统内存限额。

```shell
-XX:MaxRAM=`cat /sys/fs/cgroup/memory/memory.limit_in_bytes`
```

也可以指定 Docker 运行参数，例如：

```shell
--memory-swappiness=0
```



## Dockerfile

```
FROM base-registry.zhonganinfo.com/env/centos/jdk8-zh-aspc:v5
COPY ./target/za-libra.jar /root/startup/
WORKDIR /root/startup/
EXPOSE 8080
CMD ["java","-Xms2048m","-Xmx2048m","-XX:NewSize=768m","-XX:MaxNewSize=768m","-XX:MetaspaceSize=200m","-Xloggc:/alidata1/admin/za-libra/logs/gc.log","-XX:+UseConcMarkSweepGC","-XX:CMSFullGCsBeforeCompaction=5","-XX:+UseCMSCompactAtFullCollection","-XX:+CMSParallelRemarkEnabled","-XX:+CMSClassUnloadingEnabled","-XX:+UseCMSInitiatingOccupancyOnly","-XX:CMSInitiatingOccupancyFraction=70","-XX:+HeapDumpOnOutOfMemoryError","-verbose:gc","-XX:+PrintGCDetails","-XX:+PrintGCDateStamps","-XX:+PrintClassHistogramBeforeFullGC","-XX:+PrintClassHistogramAfterFullGC","-XX:+PrintTenuringDistribution","-XX:HeapDumpPath=/alidata1/admin/za-libra.hprof","-XX:+DisableExplicitGC","-XX:+UseCompressedOops","-XX:+DoEscapeAnalysis","-XX:MaxTenuringThreshold=10","-DAPP_DOMAIN=za-libra","-jar","za-libra.jar"]
```

这个Dockerfile中的CMD命令指定了在容器启动时运行的命令。下面是每个参数的解释：

1. `java`: 这是Java虚拟机的可执行文件。
2. `-Xms2048m`: 指定Java堆的初始内存大小为2048 MB。
3. `-Xmx2048m`: 指定Java堆的最大内存大小为2048 MB。
4. `-XX:NewSize=768m`: 设置新生代的初始大小为768 MB。
5. `-XX:MaxNewSize=768m`: 设置新生代的最大大小为768 MB。
6. `-XX:MetaspaceSize=200m`: 设置元空间的初始大小为200 MB。
7. `-Xloggc:/alidata1/admin/za-libra/logs/gc.log`: 启用GC日志，并指定日志文件的路径为`/alidata1/admin/za-libra/logs/gc.log`。
8. `-XX:+UseConcMarkSweepGC`: 启用CMS垃圾回收器。
9. `-XX:CMSFullGCsBeforeCompaction=5`: 在进行CMS全量垃圾回收之前触发压缩的次数。
10. `-XX:+UseCMSCompactAtFullCollection`: 指定CMS在进行Full GC时执行压缩。
11. `-XX:+CMSParallelRemarkEnabled`: 启用CMS并行标记。
12. `-XX:+CMSClassUnloadingEnabled`: 启用CMS类卸载功能。
13. `-XX:+UseCMSInitiatingOccupancyOnly`: 只在达到指定的堆占用率时启动CMS收集。
14. `-XX:CMSInitiatingOccupancyFraction=70`: 设置CMS收集器在堆内存使用达到70%时启动。
15. `-XX:+HeapDumpOnOutOfMemoryError`: 在内存溢出错误发生时生成堆转储文件。
16. `-verbose:gc`: 启用GC详细输出。
17. `-XX:+PrintGCDetails`: 打印GC的详细信息。
18. `-XX:+PrintGCDateStamps`: 在GC日志中打印日期时间戳。
19. `-XX:+PrintClassHistogramBeforeFullGC`: 在进行Full GC之前打印类直方图。
20. `-XX:+PrintClassHistogramAfterFullGC`: 在进行Full GC之后打印类直方图。
21. `-XX:+PrintTenuringDistribution`: 打印对象年龄分布信息。
22. `-XX:HeapDumpPath=/alidata1/admin/za-libra.hprof`: 指定堆转储文件的路径为`/alidata1/admin/za-libra.hprof`。
23. `-XX:+DisableExplicitGC`: 禁用显式的垃圾回收调用。
24. `-XX:+UseCompressedOops`: 启用压缩指针。
25. `-XX:+DoEscapeAnalysis`: 启用逃逸分析。
26. `-XX:MaxTenuringThreshold=10`: 设置对象在Survivor区中存活的最大年龄。
27. `-DAPP_DOMAIN=za-libra`: 设置Java系统属性`APP_DOMAIN`的值为`za-libra`。
28. `-jar`: 执行指定的JAR文件。
29. `za-libra.jar`: 要执行的JAR文件的路径。

