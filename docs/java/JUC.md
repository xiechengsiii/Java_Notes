## 并发工具类

##### 模仿CopyOnWriteArrayList实现的 CopyOnWriteMap

```java
public class CopyOnWriteMap<K, V> implements Map<K, V>, Cloneable {
    private volatile Map<K, V> internalMap;

    public CopyOnWriteMap() {
        internalMap = new HashMap<K, V>();
    }

    public V put(K key, V value) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            V val = newMap.put(key, value);
            internalMap = newMap;
            return val;
        }
    }

    public V get(Object key) {
        return internalMap.get(key);
    }

    public void putAll(Map<? extends K, ? extends V> newData) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            newMap.putAll(newData);
            internalMap = newMap;
        }
    }
}
```

- 适合读多写少的场景。读写是分离的，遍历和修改操作是作用在不同的容器上，所以在迭代器遍历时不会抛出异常

- 每次写操作会对原容器拷贝一份，数据量大的时候性能不好，内存压力大，可能引起频繁fullGC

- 读的是老容器的数据。如果希望写入的数据立马能准确读取，不要用CopyOnWrite

##### Reentranlock

与 Synchronized的差别

- 更灵活  支持公平锁和非公平锁， 可以条件等待，Synchronized唤醒线程要么随机一个唤醒要么全部唤醒；而Reentranlock可以选择唤醒想要唤醒的线程；

- 获取锁的过程中未获取到锁而进入同步队列中，可以响应中断抛出``InterruptedException``返回

  ​	一个线程在未获取到锁而被阻塞在synchronized上，如果对该线程中断，那么该线程的中断标志位会被修改，但是仍然是阻塞状态；而同步器提供的``acquireInterruptibly(int arg)``方法，在等待获取同步状态时，如果当前线程被中断，会立即返会，并抛出InterruptedException



##### CountDownLatch

```java
// 构造方法 初始化计数器的值
public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

 public void await() throws InterruptedException {
     	//调用tryAcquireShared， 小于0  阻塞；
        sync.acquireSharedInterruptibly(1);
    }

public void countDown() {
        sync.releaseShared(1);
}
// 这个releaseShared(1) 是AQS模板方法：
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }


// 内部的静态类 -- 继承了同步器
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
		// 这里只有 nextC == 0 才返回true  然后才会执行 doReleaseShared()的逻辑，
    	// 唤醒后继节点
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

```



##### AbstractQueuedSynchronizer

AQS使用了模板方法模式，自定义同步器时需要重写下面几个AQS提供的模板方法：

```java
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
```

​	默认情况下，每个方法都抛出 `UnsupportedOperationException`。 这些方法的实现必须是内部线程安全的，并且通常应该简短而不是阻塞。AQS类中的其他方法都是final ，所以无法被其他类使用，只有这几个方法可以被其他类使用。

###### 独占式同步状态的获取与释放

- 独占式获取同步状态   // 模板方法  子类实现``tryAcquire(arg)``

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
        selfInterrupt();
    }
}

private Node addWaiter(Node mode) {
        // 新建Node节点
        Node node = new Node(Thread.currentThread(), mode);
        // 尝试快速添加尾结点
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            // CAS方式设置尾结点
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 如果上面添加失败，这里循环尝试添加，直到添加成功为止
        enq(node);
        return node;
    }

  final boolean acquireQueued(final Node node, int arg) {
        // 操作是否成功标志
        boolean failed = true;
        try {
            // 线程中断标志
            boolean interrupted = false;
            // 不断的自旋循环
            for (;;) {
                // 当前节点的prev节点
                final Node p = node.predecessor();
                // 判断prev是否是头结点 && 是否获取到同步状态
                if (p == head && tryAcquire(arg)) {
                    // 以上条件成立，将当前节点设置成头结点
                    setHead(node);
                    // 将prev节点移除队列中
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 自旋过程中，判断当前线程是否需要阻塞 && 阻塞当前线程并且检验线程中断状态
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                // 取消获取同步状态
                cancelAcquire(node);
        }
    }
```

- 独占式超时获取同步状态  ``doAcquireNanos(int arg, long nanosTimeout)``

  与上一个接近。唯一的差别是acquire未获取到同步状态会一直等待，但是``doAcquireNanos``只会最多等待nanosTimeout纳秒，如果nanosTimeout纳秒内没有获取到同步状态，会从方法中返回
  
- 独占式释放同步状态

  ```java
  public final boolean release(int arg) {
          if (tryRelease(arg)) {
              Node h = head;
              if (h != null && h.waitStatus != 0)
                  unparkSuccessor(h);
              return true;
          }
          return false;
      }
  ```

- 总结：

  ​	在获取同步状态的时候，同步器会维护一个同步队列，获取状态失败的线程会通过``addWaiter``方法构造成一个节点并加入队列，然后节点中的线程调用``acquireQueued``不断自旋；退出自旋条件是该节点的前驱节点是头结点并且获得了同步状态。 

  ​	在释放同步状态时，调用``tryRelease``方法，然后唤醒头结点的后继节点。

###### 共享式同步状态的获取与释放

- 获取同步状态：

  ```java
   public final void acquireShared(int arg) {
          if (tryAcquireShared(arg) < 0)
              // 获取失败，自旋获取同步状态
              doAcquireShared(arg);
      }
  private void doAcquireShared(int arg) {
          // 添加共享模式节点到队列中
          final Node node = addWaiter(Node.SHARED);
          boolean failed = true;
          try {
              boolean interrupted = false;
              // 自旋获取同步状态
              for (;;) {
                  // 当前节点的前驱
                  final Node p = node.predecessor();
                  // 如果前驱节点是head节点
                  if (p == head) {
                      // 尝试去获取共享同步状态
                      int r = tryAcquireShared(arg);
                      if (r >= 0) {
                          // 将当前节点设置为头结点，并且释放也是共享模式的后继节点
                          setHeadAndPropagate(node, r);
                          p.next = null; // help GC
                          if (interrupted)
                              selfInterrupt();
                          failed = false;
                          return;
                      }
                  }
                  if (shouldParkAfterFailedAcquire(p, node) &&
                      parkAndCheckInterrupt())
                      interrupted = true;
              }
          } finally {
              if (failed)
                  cancelAcquire(node);
          }
      }
  ```

  其实也和独占式获取同步状态类似。当子类自己实现的``tryAcquireShared``返回值 >= 0时，表示能够获取。

  退出自旋的条件是前驱节点为头结点，且``tryAcquireShared``方法返回值 >= 0。

- 释放同步状态

  ```java
  public final boolean releaseShared(int arg) {
          if (tryReleaseShared(arg)) {
              doReleaseShared();
              return true;
          }
          return false;
      }
  ```

  与独占式的主要去区别是，``tryReleaseShared``必须保证同步状态（可能多个线程持有）线程安全地释放，一般采用循环 + CAS

###### Condition

- 条件等待队列    

​	用一个非并发的普通队列实现即可，因为只有在获取锁的前提下 ，才会调用``condition.await()``  

###### lockSupport工具

- ``void park()``   阻塞当前线程，调用 unpark(Thread thread)或者当前线程被中断，才能从park方法返回

