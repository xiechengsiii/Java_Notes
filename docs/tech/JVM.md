# JVM

## GC相关

​	**1**.各种垃圾回收器![img](E:\Typora\imgs\garbageCollector.png)

 		各种垃圾回收器是配合使用的。		

​		随着内存的增大，GC在不断升级，ZGC是jdk11主推的。



**2**.为什么要2个survivor?

​	1 如果没有survivor区：eden区每进行一次minor GC，就会有对象进入old区，old区会很快填满，从而增加了出发major GC的频率

​	2 如果只有一个survivor区: 

​			Eden区满了之后，触发Minor GC， 存活对象进入survivor区；	下次eden满了之后，eden和survivor各有一些存活对象，如果将eden区存活对象复制到survivor区，必然就会产生碎片如下图所致；所以需要第二个survivor区来容纳eden和survivor1区存活的对象，这样不会产生内存对象；

​			当然如果把eden区当做一个survivor区进行处理的话，就相当于eden区和survivor区的存活对象来回移动，这样要么导致较低的内存利用率，要么就会导致minor GC的频率增加；

![如果只有一个survivor区](E:\Typora\imgs\image-20200521215229143.png)

3.G1的优势

1.可以预测的停顿时间，比如长度为x毫秒的时间内，消耗在gc的时间不得超过y毫秒

2.与cms标记-清理算法不同，G1收集后提供规整的可用内存，这种特性有利于程序的长时间运行，分配大对象不会因为无法找到连续的内存空间而提前触发下一次GC

3.分成一个一个region，每次回收只做局部区域的垃圾回收，不想其他gc一样回收整个老年代，这样stw时间比较短。G1根据region里面垃圾堆积的价值大小（回收所获得的空间以及回收所需要的时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，有限回收价值最大的region。

**一个调优eg：**

对于CMS来说，要合理设置年轻代和年老代的大小。该如何确定它们的大小呢？这是一个迭代的过程，可以先采用JVM的默认值，然后通过压测分析GC日志。

如果看年轻代的内存使用率处在高位，导致频繁的Minor GC，而频繁GC的效率又不高，说明对象没那么快能被回收，这时年轻代可以适当调大一点。

如果看年老代的内存使用率处在高位，导致频繁的Full GC，这样分两种情况：如果每次Full GC后年老代的内存占用率没有下来，可以怀疑是内存泄漏；如果Full GC后年老代的内存占用率下来了，说明不是内存泄漏，要考虑调大年老代。

对于G1收集器来说，可以适当调大Java堆，因为G1收集器采用了局部区域收集策略，单次垃圾收集的时间可控，可以管理较大的Java堆。

检查内存泄漏：查看visualVM的堆转储文件

![image-20200902105920474](E:\Typora\imgs\image-20200902105920474.png)

## 调优相关

**GC调优的最重要的三个选项：**
第一位：选择合适的GC回收器

第二位：选择合适的堆大小

第三位：选择年轻代在堆中的比重

注意，大多数的java应用不需要gc调优，需要调整的是代码。GC调优是最后的手段

**调优步骤：**

**1，监控GC的状态**

使用各种JVM工具，查看当前日志，分析当前JVM参数设置，并且分析当前堆内存快照和gc日志，根据实际的各区域内存划分和GC执行时间，觉得是否进行优化；

**2，分析结果，判断是否需要优化**

如果各项参数设置合理，系统没有超时日志出现，GC频率不高，GC耗时不高，那么没有必要进行GC优化；如果GC时间超过1-3秒，或者频繁GC，则必须优化；

注：**如果满足下面的指标，则一般不需要进行GC：**

Minor GC执行时间不到50ms；

Minor GC执行不频繁，约10秒一次；

**Full GC执行时间不到1s；**

Full GC执行频率不算频繁，不低于10分钟1次；

**3，调整GC类型和内存分配**

如果内存分配过大或过小，或者采用的GC收集器比较慢，则应该优先调整这些参数，并且先找1台或几台机器进行beta，然后比较优化过的机器和没有优化的机器的性能对比，并有针对性的做出最后选择；

**4，不断的分析和调整**

通过不断的试验和试错，分析并找到最合适的参数

**5，全面应用参数**

如果找到了最合适的参数，则将这些参数应用到所有服务器，并进行后续跟踪。

**推荐策略**

1. 年轻代大小选择

- 响应时间优先的应用:尽可能设大,直到接近系统的最低响应时间限制(根据实际情况选择).在此种情况下,年轻代收集发生的频率也是最小的.同时,减少到达年老代的对象.
- 吞吐量优先的应用:尽可能的设置大,可能到达Gbit的程度.因为对响应时间没有要求,垃圾收集可以并行进行,一般适合8CPU以上的应用.
- 避免设置过小.当新生代设置过小时会导致:1.YGC次数更加频繁 2.可能导致YGC对象直接进入年老代,如果此时年老代满了,会触发FGC.

年老代大小选择

响应时间优先的应用:年老代使用并发收集器,所以其大小需要小心设置,一般要考虑并发会话率和会话持续时间等一些参数.如果堆设置小了,可以会造成内存碎片,高回收频率以及应用暂停而使用传统的标记清除方式;如果堆大了,则需要较长的收集时间.最优化的方案,一般需要参考以下数据获得:
**并发垃圾收集信息、持久代并发收集次数、传统GC信息、花在年轻代和年老代回收上的时间比例。**

吞吐量优先的应用:一般吞吐量优先的应用都有一个很大的年轻代和一个较小的年老代.原因是,这样可以尽可能回收掉大部分短期对象,减少中期的对象,而年老代尽存放长期存活对象

5、JDK为我们提供的工具

![img](https://pic3.zhimg.com/80/v2-872a55b9fc7029d6bad647810af113fc_1440w.jpg)

JVM dump分析工具MAT



6、JVM老年代和新生代的比例问题

[Java](https://link.zhihu.com/?target=http%3A//lib.csdn.net/base/java) 中的堆是 JVM 所管理的最大的一块内存空间，主要用于存放各种类的实例对象。
在 Java 中，堆被划分成两个不同的区域：新生代 ( Young )、老年代 ( Old )。新生代 ( Young ) 又被划分为三个区域：Eden、From Survivor、To Survivor。
这样划分的目的是为了使 JVM 能够更好的管理堆内存中的对象，包括内存的分配以及回收。
堆的内存模型大致为：



![img](https://pic2.zhimg.com/80/v2-0ca6ecd63aee990a0c75121ca96b5d48_1440w.jpg)



从图中可以看出： 堆大小 = 新生代 + 老年代。其中，堆的大小可以通过参数 –Xms、-Xmx 来指定。
（本人使用的是 JDK1.6，以下涉及的 JVM 默认值均以该版本为准。）
默认的，新生代 ( Young ) 与老年代 ( Old ) 的比例的值为 1:2 ( 该值可以通过参数 –XX:NewRatio 来指定 )，即：新生代 ( Young ) = 1/3 的堆空间大小。老年代 ( Old ) = 2/3 的堆空间大小。其中，新生代 ( Young ) 被细分为 Eden 和 两个 Survivor 区域，这两个 Survivor 区域分别被命名为 from 和 to，以示区分。
默认的，Edem : from : to = 8 : 1 : 1 ( 可以通过参数 –XX:SurvivorRatio 来设定 )，即： Eden = 8/10 的新生代空间大小，from = to = 1/10 的新生代空间大小。
JVM 每次只会使用 Eden 和其中的一块 Survivor 区域来为对象服务，所以无论什么时候，总是有一块 Survivor 区域是空闲着的。
因此，新生代实际可用的内存空间为 9/10 ( 即90% )的新生代空间。

7、空间分配担保



在发生minor gc之前，虚拟机会检测 : 老年代最大可用的连续空间>**新生代all对象总空间**？

1、满足，minor gc是安全的，可以进行minor gc。

2、不满足，虚拟机查看HandlePromotionFailure参数：

（1）为true，允许担保失败，会继续检测老年代最大可用的连续空间>**历次晋升到老年代对象的平均大小**。若大于，将尝试进行一次minor gc，若失败，则重新进行一次full gc。

（2）为false，则不允许冒险，要进行full gc（对老年代进行gc）。

配置：-*XX:+Handle*PromotionFailure



1.一个不错的Java诊断工具 [Arthas](https://alibaba.github.io/arthas/)  可以在线诊断

- 可以在直接在服务器上修改代码  再通过``redefine``指令重新加载编译好的

  ```java
  // 反编译 UserControlller
  $ jad --source-only com.example.demo.arthas.user.UserController > /tmp/UserController.java
  // 通过 vim 修改你的源码
  $ vim /tmp/UserController.java
  //sc 查找加载UserController类的ClassLoader
  $ sc -d *UserController | grep classLoaderHash
   classLoaderHash   1be6f5c3
  //保存好/tmp/UserController.java之后，使用mc(Memory Compiler)命令来编译，并且通过-c参数指定ClassLoader
  $ mc -c 1be6f5c3 /tmp/UserController.java -d /tmp
  Memory compiler output:
  /tmp/com/example/demo/arthas/user/UserController.class
  Affect(row-cnt:1) cost in 346 ms
  //再使用redefine命令重新加载新编译好的UserController.class：
  $ redefine /tmp/com/example/demo/arthas/user/UserController.class
  redefine success, size: 1
  ```

  不过 arthas 没有实现 jmap 命令

2.一些常用的命令：

-  jps 查看当前java进程信息 

-  jstack 查看线程的调用栈，可以查看死锁

- jmap 查看进程产生了哪些类的实例，大小， 个数， 可以用于处理OOM错误

  ​	![img](E:\Typora\imgs\jmap.png)

  但是jmap行期间对进程造成很大影响，甚至卡顿：

  ![img](E:\Typora\imgs\clipboard-1590232115277.png)

  -XX:+HeapDumpOnOutOfMemoryError 当出现OOM时，自动生成转储文件。堆转储会使所有进程暂停。

3.



