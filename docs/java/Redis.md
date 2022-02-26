



#### 数据类型



#### 为何redis单线程却效率高

- 纯内存操作
- 核心是基于非阻塞的多路复用处理器
- 单线程避免了多线程的频繁上下文切换

关于redis的线程模型，可以看这边[文章](https://www.javazhiyin.com/22943.html)

#### redis的事务

Redis 可以通过 **MULTI，EXEC，DISCARD 和 WATCH** 等命令来实现事务(transaction)功能。

```
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```

使用 [MULTI](https://redis.io/commands/multi)命令后可以输入多个命令。Redis不会立即执行这些命令，而是将它们放到队列，当调用了[EXEC](https://redis.io/commands/exec)命令将执行所有命令。

redis的事务是不支持**roll back**的，因而不满足原子性

1.若在事务队列中存在命令性错误（类似于java编译性错误），则执行EXEC命令时，所有命令都不会执行
2.若在事务队列中存在语法性错误（类似于java的1/0的运行时异常），则执行EXEC命令时，其他正确命令会被执行，错误命令抛出异常。

#### AOF重写

注意，AOF重写实际上原理是针对当前数据库状态，用多条命令记录状态而生成的新AOF日志，而不是对原有的AOF日志进行重写

##### 为什么AOF重写

- AOF持久化是通过保存被执行的写命令来记录数据库状态的。所以，AOF的文件大小会越来越大，这样会造成：计算机的存储压力加大，并且AOF还原数据库的时间增加

- 为了解决AOF文件膨胀的问题，Redis提供了AOF重写功能：创建一个新的AOF文件替代原有的AOF文件，新文件与老文件的数据库状态是相同高的，但新AOF文件的不会包括任何浪费空间的冗余命令，体积通常会小很多。

##### 实现原理

- 读取数据库的键值，用一条命令去记录键值对，代替之前的键值对的多个命令

- 伪代码：



    def AOF_REWRITE(tmp_tile_name):
    f = create(tmp_tile_name)
    for db in redisServer.db:
    
      # 如果数据库为空，那么跳过这个数据库
      if db.is_empty(): continue
      
      # 写入 SELECT 命令，用于切换数据库
      f.write_command("SELECT " + db.number)
      
      # 遍历所有键
      for key in db:
      
        # 如果键带有过期时间，并且已经过期，那么跳过这个键
        if key.have_expire_time() and key.is_expired(): continue
      
        if key.type == String:
      
          # 用 SET key value 命令来保存字符串键
      
          value = get_value_from_string(key)
      
          f.write_command("SET " + key + value)
      
        elif key.type == List:
      
          # 用 RPUSH key item1 item2 ... itemN 命令来保存列表键
      
          item1, item2, ..., itemN = get_item_from_list(key)
      
          f.write_command("RPUSH " + key + item1 + item2 + ... + itemN)
      
        elif key.type == Set:
      
          # 用 SADD key member1 member2 ... memberN 命令来保存集合键
      
          member1, member2, ..., memberN = get_member_from_set(key)
      
          f.write_command("SADD " + key + member1 + member2 + ... + memberN)
      
        elif key.type == Hash:
      
          # 用 HMSET key field1 value1 field2 value2 ... fieldN valueN 命令来保存哈希键
      
          field1, value1, field2, value2, ..., fieldN, valueN =\
          get_field_and_value_from_hash(key)
      
          f.write_command("HMSET " + key + field1 + value1 + field2 + value2 +\
                          ... + fieldN + valueN)
      
        elif key.type == SortedSet:
      
          # 用 ZADD key score1 member1 score2 member2 ... scoreN memberN
          # 命令来保存有序集键
      
          score1, member1, score2, member2, ..., scoreN, memberN = \
          get_score_and_member_from_sorted_set(key)
      
          f.write_command("ZADD " + key + score1 + member1 + score2 + member2 +\
                          ... + scoreN + memberN)
      
        else:
      
          raise_type_error()
      
        # 如果键带有过期时间，那么用 EXPIREAT key time 命令来保存键的过期时间
        if key.have_expire_time():
          f.write_command("EXPIREAT " + key + key.expire_time_in_unix_timestamp())
      
      # 关闭文件
      f.close()

##### AOF后台重写

​	fork一个子进程在后台执行。

- 子进程AOF重写的时候，主进程可以继续处理命令（如果都用同一个进程就阻塞了）

- 子进程用的是主进程的副本，是子进程而不是线程，可以避免在锁的情况下，保证数据的安全性

  CopyOnWrite的问题是，会造成当前数据库的数据和重写后的AOF文件不一致

  所以Redis是如何解决的呢？

  - redis增加了一个**AOF重写缓存**，这个缓存在fork出子进程之后开始启用。Redis主进程在执行完写命令之后，同时将这个写命令追加到AOF缓冲区和AOF重写缓冲区。相当于这样一个过程：

    <img src="E:\Typora\imgs\image-20200623092857773.png" alt="image-20200623092857773" style="zoom:50%;" />

  子进程完成对AOF文件重写之后，它会向父进程发送一个完成信号，父进程接到该完成信号之后，会调用一个信号处理函数，该函数完成以下工作：

  - 将 **重写缓冲区的内容** 追加到这个新的 AOF 日志文件中，追加完毕后就立即替换原先旧的 AOF 日志文件

  这里会对主进程造成阻塞，其他 时候AOF后台重写对主进程都没用影响。
  
  

#### Redis和MySQL数据一致性的问题

​	在高并发情况下，涉及到数据库数据的更新，就容易出现MySQL和Redis数据不一致的问题， 比如一个线程更新数据，一个线程读数据：

- 如果更新线程W是先删除缓存，然后再写Mysql；此时一个线程R来读取，发现缓存为空，直接读数据库，如果此时写线程还没来得及将数据更新到DB，那么R读到的就是脏数据，并把脏数据更新到了缓存中。

  <font color = red>先删除缓存这个逻辑是错误的，[这篇文章](https://coolshell.cn/articles/17416.html)指出的</font>

- 如果W先写库，在删除缓存前，R来读，也是读的脏数据。

所以如何解决这种个问题呢 ，我看到网上有几种方案：

- 延时双删

  在写库的时候，写库前后都	进行``redis.del(key)``的操作：

  ```java
  public void write( String key, Object data )
  {
  	redis.delKey( key );
  	db.updateData( data );
  	Thread.sleep( 500 );
  	redis.delKey( key );
  }
  ```

  这个休眠时间要根据读数据业务的耗时来定。这可以保证，读请求结束后，写请求也能删除读请求造成的缓存脏数据。

  <font color = red>删除缓存后，大量读请求走数据库，造成缓存击穿如何解决？ </font>



- Cache Aside Pattern

  这是最常用最常用的pattern了。其具体逻辑如下：

  - **失效**：应用程序先从cache取数据，没有得到，则从数据库中取数据，成功后，放到缓存中。

  - **命中**：应用程序从cache中取数据，取到后返回。

  - **更新**：先把数据存到数据库中，成功后，再让缓存失效。

    就是先更新数据库，然后让缓存失效。

![image-20200906095937435](E:\Typora\imgs\image-20200906095937435.png)

那么，是不是Cache Aside这个就不会有并发问题了？不是的，比如，一个是读操作，但是没有命中缓存，然后就到数据库中取数据，此时来了一个写操作，写完数据库后，让缓存失效，然后，之前的那个读操作再把老的数据放进去，所以，会造成脏数据。

但，这个case理论上会出现，不过，实际上出现的概率可能非常低，因为这个条件需要发生在读缓存时缓存失效，而且并发着有一个写操作。而实际上数据库的写操作会比读操作慢得多，而且还要锁表，而读操作必需在写操作前进入数据库操作，而又要晚于写操作更新缓存，所有的这些条件都具备的概率基本并不大。

- 异步更新缓存（基于订阅binlog的同步机制）

  整体思路：MySQL增量订阅消费 + 消息队列 + 增量数据更新到redis

  1.数据操作主要分为两大块

  - 全量  将全部数据一次写入到redis
  - 增量  实时更新

  这里的增量就是Mysql的delete， insert， update

  **redis更新**

  读取binlog，利用消息队列，推送更新redis的缓存数据

  这样一旦MySQL中产生了新的写入、更新、删除等操作，就可以把binlog相关的消息推送至Redis，Redis再根据binlog中的记录，对Redis进行更新。

  **这种机制有点像MySQL的主备复制啊，也是通过binlog来实现主从数据的一致性**

  这里可以结合使用canal(阿里的一款开源框架)，通过该框架可以对MySQL的binlog进行订阅，而canal正是模仿了mysql的slave数据库的备份请求，使得Redis的数据更新达到了相同的效果。

  <font color = red>这里其实并没有保证实时的一致性，只保证了最终的一致性啊？</font>

  网上有人的回答：暂时不对外提供服务，直到数据同步再恢复。可用性和一致性不可兼得。