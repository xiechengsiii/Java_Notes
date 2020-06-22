# JVM && JUC

## GC相关

​	**1**.各种垃圾回收器

​		![image-20200521214048109](E:\Typora\imgs\image-20200521214048109.png)

​		各种垃圾回收器是配合使用的。		

​		随着内存的增大，GC在不断升级，ZGC是jdk11主推的。



**2**.为什么要2个survivor?

​	1 如果没有survivor区：eden区每进行一次minor GC，就会有对象进入old区，old区会很快填满，从而增加了出发major GC的频率

​	2 如果只有一个survivor区: 

​			Eden区满了之后，触发Minor GC， 存活对象进入survivor区；	下次eden满了之后，eden和survivor各有一些存活对象，如果将eden区存活对象复制到survivor区，必然就会产生碎片如下图所致；所以需要第二个survivor区来容纳eden和survivor1区存活的对象，这样不会产生内存对象；

​			当然如果把eden区当做一个survivor区进行处理的话，就相当于eden区和survivor区的存活对象来回移动，这样要么导致较低的内存利用率，要么就会导致minor GC的频率增加；

![如果只有一个survivor区](E:\Typora\imgs\image-20200521215229143.png)



## 调优

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

  但是jamp执行期间对进程造成很大影响，甚至卡顿：

  ![img](E:\Typora\imgs\clipboard-1590232115277.png)

  -XX:+HeapDumpOnOutOfMemoryError 当出现OOM时，自动生成转储文件。堆转储会使所有进程暂停。

3.

## JUC

#### volatile

volatile的MESI缓存一致性协议，需要不断地从主内存嗅探和CAS不断循环，无效交互会导致总线带宽达到峰值，所以不要大量使用volatile。

