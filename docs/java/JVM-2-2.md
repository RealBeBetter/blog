---
title: 【Java】JVM（中）-垃圾回收（下）
date: 2021-05-21
tags:
- Java
---

## 三、垃圾回收（下）

### 垃圾回收器

**三种垃圾回收器的特点**

- 串行
  - 单线程
  - 堆内存较小，适合个人电脑
- 吞吐量优先
  - 多线程
  - 堆内存较大，多核cpu
  - 让单位时间内，STW的时间最短
- 响应时间优先
  - 多线程
  - 堆内存较大，多核cpu
  - 尽可能让单次STW的时间最短

#### 串行（Serial）

使用串行垃圾回收器的代码：

![image-20210316211933373](https://s2.loli.net/2022/04/07/Az65XeyPsBNV9Z4.png)

在VM Options中设置的参数。前者Serial使用的是复制算法的新生代区，后者SerialOld表示的是老年代，使用的是标记整理算法。

因为串行回收器是单线程的，所以在使用串行回收器的时候，需要将其他的线程阻塞，只留下串行垃圾回收器线程在运行，确保垃圾回收的安全可靠。

![image-20210316212216311](https://s2.loli.net/2022/04/07/T34pthSJCVefKq6.png)

#### 吞吐量优先（Parallel）

使用吞吐量优先垃圾回收器，需要设置的参数如下：

其中，第一行的两个参数，在jdk1.8中是默认打开的。最后一个参数是控制GC的线程数。

第二行表示的采用一个大小自适应的策略，调整的是新生代的大小。第三行：GC时间比率；第四行：最大GC暂停耗时。

| 英文     | 中文                         |
| -------- | ---------------------------- |
| Parallel | 平行的; 同时发生的;  并行的; |
| Ratio    | 比率; 比例;                  |
| Millis   | 耗时                         |

![image-20210316212544222](https://s2.loli.net/2022/04/07/VaomlgQz3yrCvjJ.png)

由于吞吐量优先的垃圾回收器算法是多线程的，所以在进行垃圾回收的时候，同时运行多个垃圾回收线程（一般线程数等于处理器核心数），降低单位内的STW时间。

![image-20210316213611886](https://s2.loli.net/2022/04/07/Ibc6x4BGyDEPRJK.png)

#### 响应时间优先

![image-20210320160533454](https://s2.loli.net/2022/04/07/as25cZLTyuUzSXN.png)

第一个参数，UseConcMarkSweepGC——Concurrent，并发；意即”使用并发标记清除GC算法“。

第二行第二个参数，并发GC线程数，一般设置的线程数为CPU核心数的四分之一。如四核CPU需要使用三个核心留给用户线程，留下一个核心处理垃圾回收。这样就使得原来的用户线程只能使用四分之三的CPU线程，这样会对整个用户程序的吞吐量造成影响。

响应时间优先的垃圾回收中，由于是并发执行的垃圾回收，当垃圾回收时，其他的四分之三的用户线程在执行的时候，也会产生一定量的垃圾，这些垃圾我们叫做浮动垃圾，但是由于并发执行，这个时候浮动垃圾只能留到下一次的垃圾回收再进行清理。由于不像之前的垃圾回收一样是等到堆内存不足再进行垃圾回收，这个时候如果浮动垃圾产生导致堆内存空间不足，则会使得空间不足的情况产生，所以在垃圾回收之前还需要预留一些空间。

此时，第三行的参数则是设置这一预留的垃圾空间占比。比如设置80%，则当堆内存空间占用达到80%的时候，就会开始进行垃圾回收，预留20%的空间留给浮动垃圾。这一参数的缺省值在65%左右，如果这一参数设置的越小，那么CMS进行的时间就要越早。

第四行的参数是留给重新标记之前扫面新生代的垃圾，可以避免一些不需要的重复扫描。

CMS垃圾回收存在一些弊端：当新生代中内存碎片过多时，可能导致并发失败，退化成SerialOld。

#### G1垃圾回收器（Garbage First）

![image-20210320174010992](https://s2.loli.net/2022/04/07/p8LVw25Ix1HGTf4.png)

根据介绍可知，在jdk9中，已经默认使用G1垃圾回收器了，废弃了之前使用的CMS响应时间优先垃圾回收器。G1和CMS同属于并发的垃圾回收器，两者在堆内存空间小的情况下，暂停的时间几乎一致；但是若是堆空间超大的堆内存，则G1的垃圾回收时间更短。

第一个参数是打开G1的使用开关，在jdk9以及之后的版本则不需要使用这一参数了，因为是默认打开的。

其中，第二个参数的使用，size大小必须设置成1、2、4、8……这样的数字大小。

##### G1垃圾回收阶段

![image-20210320175436971](https://s2.loli.net/2022/04/07/WLbfvrpnE2TSPB7.png)

在G1回收器中，以上三个阶段是一个循环的过程。从新生代收集开始，之后进入新生代的收集和并发标记阶段，再进入混合收集阶段，这一阶段同时回收新生代和老年的垃圾。

##### Young Collection

因为G1在使用的时候会将堆内存划分成一个个的Region区域，每一个区域都可以看作包含有老年代与包含Eden伊甸园、幸存区from、幸存区To的新生代。此阶段新生代的垃圾回收跟此前的垃圾回收一样，都会触发一次STW。

![image-20210320180213997](https://s2.loli.net/2022/04/07/2HUGOlZc39hMuXg.png)

一段时间之后，会将新生代中的幸存对象通过复制算法复制到幸存区。

![image-20210320180635421](https://s2.loli.net/2022/04/07/W29YZiuCMmsrt5V.png)

此后，在经历多轮的新生代垃圾回收之后，有一部分的幸存对象会晋升到老年区。

![image-20210320180743628](https://s2.loli.net/2022/04/07/7GMWQ9bLInzJ6s1.png)

##### Young Collection + Concurrent Mark

![image-20210320180848396](https://s2.loli.net/2022/04/07/dUDqljmrYhBtpSg.png)

这个阶段相比于前一个Young Collection阶段多了个并发标记。当前一阶段的Serial幸存区对象晋升到老年代中，这个时候老年代内存空间开始累积。当老年代占用堆空间比例达到阈值时，开始进行并发标记（由于是并发标记，不会STW，参考之前的CMS），阈值可以由图中的参数决定。

##### Mixed Collection

混合收集会对Eden伊甸园、Serial幸存区、Old老年区进行全面的垃圾回收。由于之前提到的最大暂停耗时缺省值是200ms，由于G1的使用场景复杂，为了达到最大暂停耗时的目标（不超出这个最大暂停耗时），使用复制算法的时候需要复制的对象过多时，这个时候就需要G1判断回收最有回收价值的老年区对象（意即回收之后能够获得更多的空闲空间——垃圾最多的区域——以及是否能够进行回收）。

此阶段的垃圾收集存在一个最终标记过程，由于之前在进行并发标记的时候，用户线程在使用，可能产生新的一写浮动垃圾或者改变对象的引用关系，这个时候垃圾回收的效果会受到影响，所以在进行最终标记的时候会出现STW。拷贝存货时候，会回收那些垃圾最多的老年代区域。

![image-20210320181519023](https://s2.loli.net/2022/04/07/NYbym1nfDoFg5uT.png)

##### Full GC

之前使用的垃圾回收器一共有四种：串行垃圾回收器——Serial GC；并行垃圾回收器——Parallel GC；相应时间优先——CMS；Garbage First垃圾回收器。但是只有在串行和并行的垃圾回收时，老年代空间不足触发的垃圾回收才能叫做full GC。

在G1垃圾回收中，当老年代内存空间占比达到阈值（默认时45%），会进入到一个并发标记的阶段以及混合收集的阶段。

①当垃圾回收的速度大于用户线程产生的垃圾速度（只有在最终标记才会STW，而且这个STW时间非常短），这个时候触发的垃圾回收不叫full GC

②当垃圾回收的速度跟不上用户的垃圾产生速度了，这个时候并发收集就会失败，最终导致垃圾回收退化成Serial GC，触发Full GC

在CMS垃圾回收中，当并发失败了，才会进行Full GC。这个阶段，可以根据GC日志中的打印内容判断是否进行了一次Full GC。

**GC分类**

- SerialGC
  - 新生代内存不足发生的垃圾收集：minor gc
  - 老年代内存不足发生的垃圾收集：full gc
- ParallelGC
  - 新生代内存不足发生的垃圾收集：minor gc
  - 老年代内存不足发生的垃圾收集：full gc
- CMS
  - 新生代内存不足发生的垃圾收集：minor gc
  - 老年代内存不足
- G1
  - 新生代内存不足发生的垃圾收集：minor gc
  - 老年代内存不足

##### Young Collection跨代引用

在新生代的垃圾回收中，整个过程是：①对新生代中的对象进行扫描，确定GC Root对象；②进行可达性分析，确定垃圾回收的对象和存活对象；③将存活对象复制到幸存区To；这一阶段，当From中存在的对象，会根据生命周期以及存活是否判断，是进行回收还是晋升到老年区中，没有达到阈值但是仍然存活的对象会被复制到To中。

> 幸存区中有两个区域，分别为From和To，比例为8：1
>
> HotSpot默认Eden与Survivor的大小比例是8 : 1，也就是说Eden : Survivor From : Survivor To = 8:1:1。所以每次新生代可用内存空间为整个新生代容量的90%，而剩下的10%用来存放回收后存活的对象。
>
> 因为新生代中的对象大多是”朝生夕死“的，存活周期很短，只有非常少的对象才会需要长时间使用。

经过这一次的Minor GC之后，From的空间会被全部清空，这个时候就会交换From区域和To区域。这个时候Minor GC已经基本完成。在进行下一次的Minor GC之前（也就是GC刚开始的时候），幸存区To的空间是空的，因为要将Eden中的对象和From区域的对象进行复制到To区域。

![image-20210320185148752](https://s2.loli.net/2022/04/07/W5IZpgB2mOi7GNf.png)

整个过程中，那么在寻找GC Root对象的时候，就会产生问题：GC Root中有一些对象是老年区的，老年代引用新生代中的对象时，需要怎么处理呢？这个时候如果遍历整个老年代的话，会产生非常高的消耗。这个时候我们采取”卡（card）表（table）“的方式，将老年代区域划分成多个细块——卡。其中有引用新生代对象的卡，称之为”脏卡“。脏卡引用的新生代对象是不能够被直接垃圾回收的。

![image-20210320185943338](https://s2.loli.net/2022/04/07/8aZmrMsT1Qlpwzq.png)

##### Remark

![image-20210320190858047](https://s2.loli.net/2022/04/07/tqR9DcXKMf6SbyO.png)

其中，黑色表示已经处理完成的；灰色表示尚在处理当中的；白色表示尚未处理的。此图中，当灰色的处理完成，因为强引用（直线箭头），灰色最终会变成黑色；而连接着灰色的白色最终也会变成黑色。独立的白色最终会被当成垃圾进行回收。

但是在实际的垃圾回收中，情况往往更加复杂。当对象的引用在标记进行的时候发生改变，原本可能可以回收的却不能被回收，但是在标记之后仍为“白色”状态，这个时候就不能对这个对象进行垃圾回收，但是标记的状态却没被更改，如何处理？

这个过程的实际处理中，JVM加入了一种写屏障——`pre-write-barrier`。当该对象的引用关系发生改变，写屏障就会得知状态改变，而将该对象加入到一个特定的队列（`stab_mark_queue`）中。

在原本的处理中，下列的对象的处理中，将C对象（原本是白色状态）转变为处理中的状态（灰色）。这个时候就会进入重新标记的状态，重新标记会发生STW，并将队列中的对象重新取出来做检查

![image-20210320191845723](https://s2.loli.net/2022/04/07/SepGLKqFuhnCUYz.png)

##### JDK 8u20 字符串去重

这个字符串去重开关为默认开启的。具体开启方法参看下图中的开启参数。因为该过程发生于新生代的垃圾回收，所以占用更多的CPU时间，但是相比于带来的内存空间收益，这个时间上的些微更多的花费更加划算。

![image-20210320192316973](https://s2.loli.net/2022/04/07/CeZtL3YW4MBXHzd.png)

##### JDK 8u40 并发标记类卸载

所有对象都经过并发标记后，就能知道哪些类不再被使用。当一个类加载器的所有类都不再被使用，就卸载它所加载的所有类，此功能默认开启。

```
-Xx:+ClassUnloadingWithConcurrentMark
```

##### JDK 8u60 回收巨型对象

- 一个对象大于 region 的一半时，称之为巨型对象
- G1 不会对巨型对象进行拷贝
- 回收时被优先考虑
- G1 会跟踪老年代所有 incoming 引用，这样老年代 incoming 引用为 0 的巨型对象就可以在新生代垃圾回收时处理掉

巨型对象的内存占用情况，是指占用的区域超过region的一半。

当老年代中的卡表对于巨型对象的引用变为0的时候，巨型对象在新生代的时候就能够被垃圾回收。

根据内存占用情况以及垃圾回收所得的效益来看，垃圾回收的时候会优先回收掉巨型对象。

![image-20210323092253901](https://s2.loli.net/2022/04/07/S4eB2tiJRQ6mxfa.png)

##### JDK 9并发标记起始时间调整

- 并发标记必须在堆空间占满前完成，否则退化为 FullGC
- JDK 9 之前需要使用 `-XX:InitiatingHeapOccupancyPercent`
- JDK 9 可以动态调整
  - `-XX:InitiatingHeap0ccupancyPercent` 用来设置初始值
  - 进行数据采样并动态调整
  - 总会添加一个安全的空档空间

根据之前的G1回收的情况，G1在垃圾回收时会经历三个阶段：Young Collection、Young Collection+CM、Mixed Collection。这个时候由第一个阶段进入到第二个阶段，开始并发标记，这个阶段会产生一个阈值（即何时开始进行并发标记）。这个时候可以通过调整上述的参数得以调整堆内存的占比设定开始并发标记的时机，此参数的缺省值是45%。而在jdk 9的版本中，该值可以动态调整以达到最好的效果。

- 如果参数过大，会导致退化成Full GC
- 如果参数过小，会导致频繁进行并发标记，降低效率

##### JDK 9更高效的回收

- 250+ 增强、180+ bug 修复
- 文档：https://docs.oracle.com/en/java/javase/12/gctuning

### 垃圾回收调优

- 掌握 GC 相关的 VM 参数，会基本的空间调整
- 掌握相关工具
- 明白一点：调优跟应用、环境有关，没有放之四海而皆准的法则

#### 调优领域

调优领域一般包括：内存、锁竞争、CPU占用、IO

#### 确定目标

根据具体的需求，确定调优的目标。第二行的垃圾回收器针对的是低延迟、快速响应；第三行的垃圾回收器是针对高吞吐量。

其中，ZGC是jdk 12中的一个尚处于体验阶段的GC。此外还有`Zing`垃圾回收器，没有STW时间。

最后，如果发现没有合适的垃圾回收器选项，可以选择HotSpot之外的其他虚拟机的垃圾回收器。

- 【低延迟】还是【高吞吐量】，选择合适的回收器
- CMS，G1，ZGC
- ParallelGC

#### 最快的GC是不发生GC

- 查看 Full GC 前后的内存占用，考虑下面几个问题
  - 数据是不是太多：resultSet = statement. executeQuery("select * from 大表 limit n")
- 数据表示是否太臃肿：
  - 对象图
  - 对象大小 16 Integer 24 int 4
- 是否存在内存泄漏：
  - static Map map = ... 静态变量
  - 软引用
  - 弱引用
  - 第三方缓存实现

#### 新生代调优

- 新生代的特点
  - 所有的 new 操作的内存分配非常廉价：TLAB thread-local allocation buffer
  - 死亡对象的回收代价是零
  - 大部分对象用过即死
  - Minor GC 的时间远远低于 Full GC

以上的特点，说明了新生代优化空间大。在实际情况中，我们一般通过预设新生代的堆内存大小（预设堆内存），达到调优新生代的目的。

![image-20210323101252815](https://s2.loli.net/2022/04/07/4L1ButXP7FWQYyD.png)

根据Oracle的官方建议，新生代的内存预设最好控制在整个堆内存的25%-50%之间。

吞吐量和新生代空间大小的关系大致如下所示：

![image-20210323101805081](https://s2.loli.net/2022/04/07/3D9yHicG67vBj4d.png)

新生代需要能容纳所有【并发量*（请求-响应）】的数据。其中，新生代中的幸存区需要能满足保留【当前活跃对象+需要晋升对象】的空间大小。

![image-20210323102852967](https://s2.loli.net/2022/04/07/GAtxQHRcYIkezbg.png)

其中，第一个参数是设置新生代晋升为老年代的阈值；第二个参数是打印相关的参数信息，参数信息例如存活周期、内存大小、总内存。

#### 老年代调优

以 CMS 为例：

- CMS的老年代内存越大越好
- 先尝试不做调优，如果没有 Full GC 那么已经... ，否则先尝试调优新生代
- 观察发生 Full GC 时老年代内存占用，将老年代内存预设调大 1/4 ~ 1/3
- `-XX:CMSInitiatingOccupancyFraction=percent`

#### 案例

根据调优相关的问题，分析几个案例。

- 案例 1：Full GC 和 Minor GC 频繁
- 案例 2：请求高峰期发生 Full GC ，单次暂停时间特别长（CMS）
- 案例 3：老年代充裕情况下，发生 Full GC（1.7）

这些案例中，需要先了解到GC过程中的一些流程。

> 初始标记：仅仅标记GC Roots的直接关联对象，并触发STW
>
> 并发标记：使用GC Roots Tracing算法，进行跟踪标记，不触发STW
>
> 重新标记：因为并发标记的缘故，其他用户线程不暂停，可能产生浮动垃圾，所以这一阶段产生STW

案例三中，JDK1.7版本使用的是元空间以及永久代，当元空间不足的时候，就会触发一次Full GC。解决方案就是增大元空间的初始值和最大值。
