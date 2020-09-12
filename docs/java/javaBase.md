##  JAVA语言相关

1. ```java
   //System.out :: println 是一个方法引用，其等价于 x -> System.out.println(x)
   //使用方法是， 要用 :: 分割方法名与对象或类名 主要三种情况
    Object :: instanceMethod
    Class :: staticMethod
    Class :: instanceMethod
    eg: ArrayList<String> names = ...;
   	Stream<Person> stream = names.stream.map(Person :: new);
   	List<Person> people = stream.collect(Collectors.toList());
    //如果有多个构造器，编译器会选择一个带String参数的构造器
   ```

2. Comparator有很多有意思的用法，见java核心技术6.3.8

3. 什么时候用静态内部类？

   在内部类不要访问外围对象的时候，应该使用静态内部类。
   
4. final关键字

- final修饰的类不能被继承，final类中的所有成员方法都会被隐式的指定为final方法；
- final修饰的方法不能被重写；
- final修饰的变量是常量，如果是基本数据类型的变量，则其数值一旦在初始化之后便不能更改；如果是引用类型的变量，则在对其初始化之后便不能让其指向另一个对象。final static 一般用于修饰常量，并且变量名大写
- final的重排序规则可以保证，在使用对象中的某个final域时，该final域在构造函数中已经被正确初始化了。

5. 子类的初始化过程

   父类修饰的static代码块   -->  子类修饰的static代码块 -->  父类实例变量和非static代码块 -->  父类构造方法 --> 子类实例变量和非static块  --> 子类构造方法 
   
6. ``Arrays.asList()``

   首先，这是一个泛型方法，传入的对象必须是个对象数组；

   其次，返回的并不是 `java.util.ArrayList` ，而是 `java.util.Arrays` 的一个内部类,这个内部类并没有实现集合的修改方法或者说并没有重写这些方法。

   所以如何将数组正确的转化成一个`java.util.ArrayList`？

   ```java
   // 1.最简便的方法
       List list = new ArrayList<>(Arrays.asList("a","b", "c"));
   
   //2.使用stream
   Integer [] myArray = { 1, 2, 3 };
   List myList = Arrays.stream(myArray).collect(Collectors.toList());
   //基本类型也可以实现转换（依赖boxed的装箱操作）
   int [] myArray2 = { 1, 2, 3 };
   List myList = Arrays.stream(myArray2).boxed().collect(Collectors.toList());
   ```

   7.注解的原理
   
   [这篇文章](https://cloud.tencent.com/developer/article/1172371)
   
   8.fail-fast 和 fail-safe
   
   一：快速失败（fail—fast） 
   
   ​           在用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的内容进行了修改（增加、删除、修改），则会抛出Concurrent Modification Exception。
   
   ​       原理：迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个        modCount 变量。集合在被遍历期间如果内容发生变化，就会改变modCount的值。每当迭代器使用hashNext()/next()遍历下一个元素之前，都会检测modCount变量是否为expectedmodCount值，是的话就返回遍历；否则抛出异常，终止遍历。      
   
   ​               注意：这里异常的抛出条件是检测到 modCount！=expectedmodCount 这个条件。如果集合发生变化时修改modCount值刚好又设置为了expectedmodCount值，则异常不会抛出。因此，不能依赖于这个异常是否抛出而进行并发操作的编程，这个异常只建议用于检测并发修改的bug。      
   
   ​               场景：java.util包下的集合类都是快速失败的，不能在多线程下发生并发修改（迭代过程中被修改）。      
   
   ​              二：安全失败（fail—safe）      
   
   ​               采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。      
   
   ​                         原理：由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发Concurrent Modification Exception。      
   
   ​               缺点：基于拷贝内容的优点是避免了Concurrent Modification Exception，但同样地，迭代器并不能访问到修改后的内容，即：迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的。      
   
   ​         场景：java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。

## 集合框架

#### 非线程安全类集合

- LinkedHashMap 

  ```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
  ```

  继承自HashMap，在HashMap的基础上，维护一条双向链表，可以保持**遍历顺序**和**插入顺序**一致(默认的)；并且还能对访问顺序提供支持：当``accessOrder == true``,可以维护访问顺序。因此可以以LinkedHashMap为基础实现LRU缓存策略

  ​	内部类``Entry``有before和after两个指针，可以维护一个双向链表，保证遍历顺序和插入顺序一致。

  详见这篇[文章]([http://www.tianxiaobo.com/2018/01/24/LinkedHashMap-%E6%BA%90%E7%A0%81%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90%EF%BC%88JDK1-8%EF%BC%89/](http://www.tianxiaobo.com/2018/01/24/LinkedHashMap-源码详细分析（JDK1-8）))
  
  利用LinkedHashMap实现LRU
  
  ```java
  public class SimpleCache<K, V> extends LinkedHashMap<K, V> {
  
      private static final int MAX_NODE_NUM = 100;
  
      private int limit;
  
      public SimpleCache() {
          this(MAX_NODE_NUM);
      }
  
      public SimpleCache(int limit) {
          super(limit, 0.75f, true);
          this.limit = limit;
      }
  
      public V save(K key, V val) {
          return put(key, val);
      }
  
      public V getOne(K key) {
          return get(key);
      }
  
      public boolean exists(K key) {
          return containsKey(key);
      }
      
      /**
       * 判断节点数是否超限
       * @param eldest
       * @return 超限返回 true，否则返回 false
       */
      @Override
      protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > limit;
      }
  }
  ```
  
  ```java
  // putval方法有 afterNodeInsertion(evict)回调方法：
  ...
      if (++size > threshold) {...}
      afterNodeInsertion(evict);    // 回调方法，后续说明
      return null;
  //linkedHashMap实现了这个方法：
  void afterNodeInsertion(boolean evict) { // possibly remove eldest
      LinkedHashMap.Entry<K,V> first;
      // 根据条件判断是否移除最近最少被访问的节点
      if (evict && (first = head) != null && removeEldestEntry(first)) {
          K key = first.key;
          removeNode(hash(key), key, null, false, true);
      }
  }
  
  // 移除最近最少被访问条件之一，通过覆盖此方法可实现不同策略的缓存
  protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
      return false;
  }
  ```

几种Map的总结：

- HashMap底层基于拉链式的散列结构，并在JDK1.8中引入红黑树优化链表过长的问题。基于这样的结构，HashMap可以实现高效的增删改查
- LinkedHashMap 在HashMap的基础上维护了一条双向链表，实现了散列数据结构的有序遍历
- TreeMap底层基于红黑树的实现，实现了键值对的排序功能

  

- ArrayList

  - LinkedList

  - HashMap

    ​	性能瓶颈在resize方法，会遍历hash表中所有元素，进行一重新hash分配，是很耗时的。所以构造hashmap的时候，最好直接定好capacity，减少自动扩容次数。

    扩容机制：

    ```java
     if (++size > threshold)
            resize();
    ```

    什么时候链表变成红黑树？	

    当链表长度大于``TREEIFY_THRESHOLD = 8``时，会调用``treeifyBin``方法，在这个方法里，先会判断

    ```
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)  resize();
    ```

    如果数组长度小于64时，先对数组进行扩容而不是转化成红黑树，以减少搜索时间。如果大于，才会将链表转化为红黑树。

    






#### 几种常见的并发容器

- ConcurrentHashMap

  1.7和1.8的实现是不同的

  - 1.7 及以前

    Segment[]（Segment继承了Reentranlock) + HashEntry数组 + 链表

    使用分段锁Segment来保护不同段的数据。在插入，获取元素的时候先要定位到对应的Segment

    Segment数量默认16， 最大2 ^16。

    - 如何定位Segment？

    ```java
    segmentShift = 32 - sshift;
    segmentMask = ssize - 1;
    Segment segmentFor(int hash){
        return segments[(hash >>> segmentShift) & segmentMask];
    }
    ```

    - 是否需要扩容？

      判断Segment里的HashEntry数组是否超过容量（阈值threshhold）。如果超过阈值，创建一个容量为预先两倍的数组，元素再散列后插入到新的数组。另外不会对整个容器扩容，而是对当前的Segment扩容。

    

  - 1.8

    Node[] + 链表/红黑树  

    通过Synchronizede和自旋 + CAS的方式实现

    有个关键变量sizeCtl 

    初始化initTable

    ```java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            ／／　如果 sizeCtl < 0 ,说明另外的线程执行CAS 成功，正在进行初始化。
            if ((sc = sizeCtl) < 0)
                // 让出 CPU 使用权
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
    ```

    从源码中可以发现 ConcurrentHashMap 的初始化是通过**自旋和 CAS** 操作完成的。里面需要注意的是变量 `sizeCtl` ，它的值决定着当前的初始化状态。

    1. -1 说明正在初始化
    2. -N 说明有N-1个线程正在进行扩容
    3. 表示 table 初始化大小，如果 table 没有初始化
    4. 表示 table 容量，如果 table　已经初始化。

- ConcurrentLinkedQueue

- CopyOnArrayList

  写时复制， 适合读多写少的场景。存在数据不一致的问题。

  ```java
   public boolean add(E e) {
          final ReentrantLock lock = this.lock;
          lock.lock();
          try {
              Object[] elements = getArray();
              int len = elements.length;
           	//复制原先的数组，在新数组的基础上增加一个元素,然后赋值给内部的 volatile array；
              //此时另外的线程的读操作可能读的还是老版本的array
              Object[] newElements = Arrays.copyOf(elements, len + 1);
              newElements[len] = e;
           
              setArray(newElements);
              return true;
          } finally {
              lock.unlock();
          }
      }
  ```

  

- BlockingQueue

- ConcurrentSkipList
