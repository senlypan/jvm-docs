# GC日志分析

![访问统计](https://visitor-badge.glitch.me/badge?page_id=senlypan.jvm.05-gc-log-analysis&left_color=blue&right_color=red)

> 作者: 潘深练
>
> 更新: 2022-03-02

## ✨ 内容已经在梳理，持续发布
?> ❤️ 您也可以参与梳理，快来提交 [issue](https://github.com/senlypan/jvm-docs/issues) 或投稿参与吧~




## GC日志参数

### GC日志参数

| 参数 | 说明|
|------|-----|
|-XX:+PrintGC |打印简单GC日志。 类似：-verbose:gc -XX:+PrintGCDetails 打印GC详细信息|
|XX:+PrintGCTimeStamps |输出GC的时间戳（以基准时间的形式）|
|-XX:+PrintGCDateStamps| 输出GC的时间戳（以日期的形式）|
|-XX:+PrintHeapAtGC |在进行GC的前后打印出堆的信息|
|-Xloggc:../logs/gc.log| 指定输出路径收集日志到日志文件|

例如，使用如下参数启动：

> -Xms28m
> -Xmx28m //开启记录GC日志详细信息（包括GC类型、各个操作使用的时间）,并且在程序运行结束打印出JVM的内存占用情况
> -XX:+PrintGCDetails
> -XX:+PrintGCDateStamps
> -XX:+UseGCLogFileRotation 开启滚动生成日志
> -Xloggc:E:/logs/gc.log


### 常用垃圾收集器参数

|参数 |描述|
|------|-----|
|UseSerialGC |虚拟机在运行在 Client 模式下的默认值，打开此开关后，使用Serial+Serial Old 收集器组合进行内存回收|
|UseParNewGC |使用 ParNew + Serial Old 收集器组合进行内存回收 |
|UseConcMarkSweepGC |使用 ParNew + CMS + Serial Old 的收集器组合尽心内存回收，当 CMS 出现 Concurrent Mode Failure 失败后会使用 Serial Old 作为备用收集器|
|UseParallelOldGC |使用 Parallel Scavenge + Parallel Old 的收集器组合 |
|UseParallelGC |使用 Parallel Scavenge + Serial Old （PS MarkSweep）的收集器组合|
|SurvivorRatio| 新生代中 Eden 和任何一个 Survivor 区域的容量比值，默认为 8 |
|PretenureSizeThreshold |直接晋升到老年代对象的大小，单位是Byte |
|UseAdaptiveSizePolicy |动态调整 Java 堆中各区域的大小以及进入老年代的年龄|
|ParallelGCThreads |设置并行 GC 时进行内存回收的线程数|
|GCTimeRatio GC |时间占总时间的比率，默认值为99，只在 Parallel Scavenge 收集器的时候生效|
|MaxGCPauseMillis |设置 GC 最大的停顿时间，只在 Parallel Scavenge 收集器的时候生效|
|CMSInitiatingOccupancyFraction |设置 CMS 收集器在老年代空间被使用多少后触发垃圾收集，默认是68%，仅在 CMS 收集器上生效|
|CMSFullGCsBeforeCompaction |设置 CMS 收集器在进行多少次垃圾回收之后启动一次内存碎片整理|
|UseG1GC |使用 G1 (Garbage First) 垃圾收集器|
|MaxGCPauseMillis |设置最大GC停顿时间(GC pause time)指标(target). 这是一个软性指标(sox goal), JVM 会尽量去达成这个目标. |
|G1HeapRegionSize| 使用G1时Java堆会被分为大小统一的的区(region)。此参数可以指定每个heap区的大小. 默认值将根据 heap size 算出最优解. 最小值为 1Mb, 最大值
为 32Mb.|


## GC日志分析

### 日志的含义



> a: GC 或者是 Full GC 
> b: 用来说明发生这次 GC 的原因 
> c: 表示发生GC的区域，这里表示是新生代发生了GC，上面那个例子是因为在新生代中内存不够给新对象分配了，然后触发GC 
> d: GC 之前该区域已使用的容量 
> e: GC 之后该区域已使用的容量
> f: 该内存区域的总容量 
> g: 表示该区域这次 GC 使用的时间 
> h: 表示 GC 前整个堆的已使用容量 
> i: 表示 GC 后整个堆的已使用容量 
> j: 表示 Java 堆的总容量 
> k: 表示 Java堆 这次 GC 使用的时间 
> l: 代表用户态消耗的 CPU 时间 
> m: 代表内核态消耗的 CPU 时间 
> n: 整个 GC 事件从开始到结束的墙钟时间（Wall Clock Time）

```

### 使用 ParNew + Serial Old 的组合进行内存回收
### 使用 Parallel Scavenge + Parallel Old 的组合进行内存回收

## 日志分析工具

### GCeasy
### GCViewer


（待补充）