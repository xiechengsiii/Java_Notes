# JVM

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

