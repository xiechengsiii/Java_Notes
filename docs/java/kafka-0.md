## Introduction & Get started
### 1 kafka介绍
#### 1.1 起源
​	Kafka 是为了解决 LinkedIn 数据管道问题应运而生的。它的设计目的是提供一个高性能的消息系统，可以处理多种数据类型，并能够实时提供结构化的用户活动数据和系统度量指标。kafka于2010年12月份开源，成Apache的顶级子项目。

![项目结构演变](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/kaka-0.png)
 
​ 
​ 	不同系统之间复杂的数据同步是一中挑战。kafka就是为了解决上述问题而设计的一款基于发布与订阅的消息系统。Kafka可以让合适的数据以合适的形式出现在合适的地方。
#### 1.2 kafka优势
1. 无缝支持多个生产者与消费 Kafka 可以无缝地支持多个生产者，不管客户端在使用单个主题还是多个主题。所以它很适合用来从多个前端系统收集数据，并以统一的格式对外提供数据。例如，一个包含了多个微服务的网站，可以为页面视图创建一个单独的主题，所有服务都以相同的消息格式向该主题写入数据。消费者应用程序会获得统一的页面视图，而无需协调来自不同生产者的数据流。

2. 基于磁盘的数据存储 生产者生产的消息被提交到磁盘，根据设置的保留规则进行保存。每个主题可以设置单独的保留规则，以便满足不同消费者的需求，各个主题可以保留不同数量的消息。消费者可能会因为处理速度慢或突发的流量高峰导致无法及时读取消息，而持久化数据可以保证数据不会丢失。消费者可以在进行应用程序维护时离线一小段时间，而无需担心消息丢失或堵塞在生产者端。消费者可以被关闭，但消息会继续保留在 Kafka 里。消费者可以从上次中断的地方继续处理消息。

3. 伸缩性 为了能够轻松处理大量数据， Kafka 从一开始就被设计成一个具有灵活伸缩性的系统。比如在开发阶段可以先使用单个 broker，再扩展到包含 3 个 broker 的小型开发集群，然后随着数据量不断增长，部署到生产环境的集群可能包含上百个 broker。对在线集群进行扩展丝毫不影响整体系统的可用性。

4. 高性能 通过横向扩展生产者、消费者和 broker， Kafka 可以轻松处理巨大的消息流。在处理大量数据的同时，它还能保证亚秒级的消息延迟。
#### 1.3 使用场景
1. 传递消息
2. 活动跟踪
3. 度量指标和日志记录
4. 提交日志
5. 流处理
	具体来说，一些对实时性要求不高的写操作，不需要立刻将结果反馈给用户，都可以采用kafka来解耦服务和存储过程。
#### 1.4 数据生态系统
​ 大量的工具都能集成了kafka。Ecosystem 中列出了许多工具，包括流处理系统、Hadoop集成、监控和部署工具。
 ![kafka生态](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/kafka-ecosystem.png)

kafka为数据生态系统带来了循环系统。它在基础设施的各个组件之间传递消息，为所有客户端提供一致的接口。当与提供消息模式的系统集成时，生产者和消费者之间不再有紧密的耦合，也不需要在它们之间建立任何类型的直连。
#### 1.5 基本概念
**broker**  
	一个独立的kafka服务器就是broker。broker接受来自生产者的消息，为消息设置偏移量，提交消息到磁盘，为消费者读取分区的请求作出响应，返回已经提交到磁盘上的消息。kafka集群有由一个或多个broker组成  
	  
**producer**  
	创建消息并推到broker  
	  
**consumer**  
	从broker拉取消息，多个消费者组成消费者群组，在一个群组的消费者共同读取一个主体  
	  
**topic**  
	kafka消息通过主题进行分类  
	  
**offset**  
	partion中消息的唯一标识。对于同一个partition，一个不断递增的整数值。在创建消息时，kafka会将其添加到消息中。在给定的分区中，每个消息的偏移量都是唯一的。  
	  
**partition**  
	一个主题上的消息，可以存储在不同分区上。	
	为什么要多个分区呢? 
 ![消费者群组](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/kafka-2.png)  
   
**leader & follower**  
	leader读写，follower读数据。  
	  

**HW & LEO**  
 	![Hw&LEO](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/HWAndLEO.png)<br> 

**逻辑架构**  
	 kafka体系架构包括若干Producer、Broker、Consumer和一个zookeeper集群。
 	![Hw&LEO](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/architecture.png)  
push vs pull?  
参考[push vs pull](http://kafkadoc.beanmr.com/040_design/01_design_cn.html#theconsumer)	
### 2 kafka生产者和消费者
#### 2.1 生产者
2.1.1 **生产者发送消息的主要步骤**  
 ![消费者群组](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/produceStep.png)  
2.1.2 **创建生产者**  
必选属性  
1. bootstrap.servers
2. broker地址
3. key.serializer
4. key的序列化器
5. value.serializer
6. value的序列化器
创建一个新的生产者的简单示例，只指定了必要的属性，其他使用默认设置 ：
``` Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        //acks = 0, 1, all
        props.put("acks", "0");
        //配置为大于0的值的话，客户端会在消息发送失败时重新发送。
        props.put("retries", 0);
        //当多条消息需要发送到同一个分区时，生产者会尝试合并网络请求。
        props.put("batch.size", 16384);
        props.put("key.serializer", StringSerializer.class.getName());
        props.put("value.serializer", StringSerializer.class.getName());
        this.producer = new KafkaProducer<String, String>(props);
 ``` 
2.1.3  **消息发送**  
1. 同步发送  
使用 send() 方法发送消息，它会返回一个 Future 对象，调用 get() 方法进行等待，就可以知道消息是否发送成功。
2. 异步发送  
通过 Callback 模式异步获取消息发送的响应结果，即不管消息发送成功还是失败，都会以回调的方式通知客户端，客户端期间不需要阻塞等待。
```private class DemoProducerCallback implements Callback {
   @Override
   public void onCompletion(RecordMetadata recordMetadata, Exception e) {
       if (e != null) {
           e.printStackTrace(); 
           }
       }
   }
   ProducerRecord<String, String> record =
   new ProducerRecord<>("testTopic", "Products2", "USA"); 
   producer.send(record, new DemoProducerCallback());
```  
tips: 上述例子都使是单线程，但其实生产者是可以使用多线程来发送消息的。如果需要更高的吞吐量，可以在生产者数量不变的前提下增加线程数量。如果这样做还不够，可以增加生产者数量 。

2.1.4 **分区**  

如果键值为 null，并且使用了默认的分区器，那么消息将被随机地发送到主题内各个可用的分区上。分区器使用轮询（Round Robin）将消息均衡地分布到各个分区上。如果键不为空，并且使用了默认的分区器，那么 Kafka 会对键进行散列 ，散列值把消息映射到特定的分区上
同样也可以自定义分区策略,需要实现Partitioner 接口：
```public class CustomerPartition implements Partitioner {
    @Override
    public void close() {

    }
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        if ("testKey".equals(key)) {
            return 0;
        }
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int partitionsSize = partitions.size();
        return Utils.toPositive(Utils.murmur2(keyBytes)) % partitionsSize;

    }

    @Override
    public void configure(Map<String, ?> map) {

    }
}
```
#### 2.2 消费者
2.2.1 创建消费者
与生产者一样，有三个必要属性：bootstrap.servers、 key.deserializer 和 value.deserializer。 
``` Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "CountryCounter");
props.put("key.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<String,String>(props)
```
**订阅主题**   
```
consumer.subscribe(Collections.singletonList("testTopic")); 
```
**轮询**  
poll方法是消费者 API 的核心，通过一个简单的轮询向服务器请求数据。一旦消费者订阅了主题，轮询就会处理所有的细节，包括群组协调、分区再均衡、发送心跳和获取数据。
```
try {
    while (true) { 
        ConsumerRecords<String, String> records = consumer.poll(100); 
        for (ConsumerRecord<String, String> record : records) {
            int updatedCount = 1;
            if (custCountryMap.countainsValue(record.value())) {
                updatedCount = custCountryMap.get(record.value()) + 1;
            }
            custCountryMap.put(record.value(), updatedCount)
            JSONObject json = new JSONObject(custCountryMap);
            System.out.println(json.toString(4)) 
            }
        }
    } finally {
    consumer.close();
}
```
2.2.2 提交和偏移量
​	提交就是更新分区当前消息位置的操作。offest 可以自己控制
1. 自动提交
​	如果 enable.auto.commit 被设为 true，那么每过 auto.commit.interval.ms，消费者会自动把从 poll() 方法接收到的最大偏移量提交上去。
​ 
2 手动提交（推荐）
​	把 auto.commit.offset 设为 false, 由自己控制控制偏移量提交时间来消除丢失消息的可能性，并在发生再均衡时减少重复消息的数量 。
```
//同步提交
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records){
        System.out.printf("topic = %s, partition = %s, offset =
        %d, customer = %s, country = %s\n",
        record.topic(), record.partition(),
        record.offset(), record.key(), record.value()); 
    }
    try {
        consumer.commitSync(); 
    } catch (CommitFailedException e) {
    log.error("commit failed", e) 
    }
}

//异步提交
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records){
        System.out.printf("topic = %s, partition = %s,
        offset = %d, customer = %s, country = %s\n",
        record.topic(), record.partition(), record.offset(),
        record.key(), record.value());
    }
    consumer.commitAsync(); 
}
// 异步提交同样支持回调
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %s,
        offset = %d, customer = %s, country = %s\n",
        record.topic(), record.partition(), record.offset(),
        record.key(), record.value());
    }
    consumer.commitAsync(new OffsetCommitCallback() {
    public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception e) {
        if (e != null) log.error("Commit failed for offsets {}", offsets, e);
        }
    }); 
}
```
在成功提交或碰到无法恢复的错误之前， commitSync() 会一直重试，但是 commitAsync() 不会。它之所以不进行重试，是因为在它收到服务器响应的时候，可能有一个更大的偏移量已经提交成功。
#### 2.3 一些说明
2.3.1 **生产者配置**  
1. acks  
指定了必须要有多少个分区副本收到消息，生产者才会认为消息写入是成功的。这个参数对消息丢失的可能性有重要影响。
​ 	acks = 0 生产者在成功写入消息之前不会等待任何来自服务器的响应。生产者不知道消息是否丢失，不过，因为生产者不需要等待服务器的响应，所以它可以以网络能够支持的最大速度发送消息，从而达到很高的吞吐量。
​ 	acks = 1 只要集群的首领节点收到消息，生产者就会收到一个来自服务器的成功响应。如果消息无法到达首领节点（比如首领节点崩溃，新的首领还没有被选举出来），生产者会收到一个错误响应，为了避免数据丢失，生产者会重发消息。
​	acks = all（-1） 只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应。这种模式是最安全的，它可以保证不止一个服务器收到消息，就算有服务器发生崩溃，整个集群仍然可以运行。不过，它的延迟比 acks=1 时更高，因为需要等待不只一个服务器节点接收消息。
 	![Hw&LEO](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/acks.png)  
	2. buffer.memory
该参数用来设置生产者内存缓冲区的大小，生产者用它缓冲要发送到服务器的消息。如果应用程序发送消息的速度超过发送到服务器的速度，会导致生产者空间不足。这个时候，send() 方法调用要么被阻塞，要么抛出异常，取决于如何设置 max.block.ms参数，表示在抛出异常之前可以阻塞多长时间。

3. compression.type    
默认情况下，消息发送时不会被压缩。该参数可以设置为 snappy、 gzip 或 lz4，它指定了消息被发送broker 之前使用哪一种压缩算法进行压缩 。snappy 算法占用较少的 CPU，能提供较好的性能和相当可观的压缩比，如果比较关注性能和网络带宽，可以使用这种算法。 gzip 压缩算法一般会占用较多的 CPU，但会提供更高的压缩比，所以如果网络带宽比较有限，可以使用这种算法。使用压缩可以降低网络传输开销和存储开销，而这往往是向 Kafka 发送消息的瓶颈所在。

4. retries  
生产者可以重发消息的次数，如果达到这个次数，生产者会放弃重试并返回错误。默认情况下，生产者会在每次重试之间等待 100ms。

5. batch.size  
当有多个消息需要被发送到同一个分区时，生产者会把它们放在同一个批次里。
6. linger.ms  
该参数指定了生产者在发送批次之前等待更多消息加入批次的时间。 KafkaProducer 会在批次填满或 linger.ms 达到上限时把批次发送出去。默认情况下，只要有可用的线程，生产者就会把消息发送出去，就算批次里只有一个消息。把 linger.ms 设置成比 0 大的数，让生产者在发送批次之前等待一会儿，使更多的消息加入到这个批次。虽然这样会增加延迟，但也会提升吞吐量。

7. max.in.flight.requests.per.connection  
指定了生产者在收到服务器响应之前可以发送多少个消息。它的值越高，就会占用越多的内存，不过也会提升吞吐量。把它设为 1 可以保证消息是按照发送的顺序写入服务器的，即使发生了重试。

8. timeout.ms、 request.timeout.ms 和 metadata.fetch.timeout.ms  
request.timeout.ms 指定了生产者在发送数据时等待服务器返回响应的时间， metadata.fetch.timeout.ms 指定了生产者在获取元数据（比如目标分区的首领是谁）时等待服务器返回响应的时间。如果等待响应超时，那么生产者要么重试发送数据，要么返回一个错误（抛出异常或执行回调）。 timeout.ms 指定了 broker 等待同步副本返回消息确认的时间。

9. max.request.size  
用于控制生产者发送的请求大小。它可以指能发送的单个消息的最大值，也可以指单个请求里所有消息总的大小。例如，假设这个值为 1MB， 那么可以发送的单个最大消息为 1MB，或者生产者可以在单个请求里发送一个批次，该批次包含了 1000 个消息，每个消息大小为 1KB。另外， broker 对可接收的消息最大值也有自己的限制（message.max.bytes），所以两边的配置最好可以匹配，避免生产者发送的消息被 broker 拒绝。

10. receive.buffer.bytes 和 send.buffer.bytes  
分别指定了 TCP socket 接收和发送数据包的缓冲区大小。如果它们被设为 -1，就使用操作系统的默认值。
tips：Kafka 可以保证同一个分区里的消息是有序的。 如果把 retries 设为非零整数，同时把 max.in.flight.requests.per.connection 设为比 1 大的数，那么，如果第一个批次消息写入失败，而第二个批次写入成功， broker 会重试写入第一个批次。如果此时第一个批次也写入成功，那么两个批次的顺序就反过来了。一般来说，如果消息对顺序敏感，可以把 max.in.flight.requests.per.connection 设为 1，这样在生产者尝试发送第一批消息时，就不会有其他的消息发送给 broker。不过这样会降低生产者的吞吐量，
2.3.2 **消费者配置**  
1. fetch.min.bytes  
消费者从服务器获取记录的最小字节数。 broker 在收到消费者的数据请求时，如果可用的数据量小于 fetch.min.bytes 指定的大小，那么它会等到有足够的可用数据时才把它返回给消费者。这样可以降低消费者和 broker 的工作负载，因为它们在主题不是很活跃的时候（或者一天里的低谷时段）就不需要来来回回地处理消息。

2. fetch.max.wait.ms  
用于指定 broker 的等待时间，默认是 500ms。 如果 fetch.max.wait.ms 被设为 100ms，并fetch.min.bytes 被设为 1MB，那么 Kafka 在收到消费者的请求后，要么返回 1MB 数据，要么在 100ms 后返回所有可用的数据，就看哪个条件先得到满足。

3. max.partition.fetch.bytes  
指定了服务器从每个分区里返回给消费者的最大字节数。它的默认值是 1MB，。如果一个主 题有 20 个分区和 5 个消费者，那么每个消费者需要至少 4MB 的可用内存来接收记录。在为消费者分配内存时，可以给它们多分配一些，因为如果群组里有消费者发生崩溃，剩下的消费者需要处理更多的分区。 max.partition. fetch.bytes 的值必须比 broker 能够接收的最大消息的字节数（通过 max.message.size 属性配置）大，否则消费者可能无法读取这些消息，导致消费者一直挂起重试 。

4. session.timeout.ms  
指定了消费者在被认为死亡之前可以与服务器断开连接的时间，默认是 3s。如果消费者没有在 session.timeout.ms 指定的时间内发送心跳给群组协调器，就被认为已经死亡，协调器就会触发再均衡，把它的分区分配给群组里的其他消费者。

5. auto.offset.reset  
指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下，该作何处理。默认值是 latest.
tips ：分区没有提交偏移量时才生效  
 	![offsetReset.png](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/offsetReset.png)  
6. enable.auto.commit  
该属性指定了消费者是否自动提交偏移量，默认值是 true。为了尽量避免出现重复数据和数据丢失 一般设为 false，由自己控制何时提交偏移量
如果设置手动提交，但是没有提交，会从最近的一次提交的offerset拉取数据


7. partition.assignment.stratege  
Range---把主题的若干个连续的分区分配给消费者。
RoundRobin---把主题的所有分区逐个分配给消费者。

8. max.poll.records  
单次调用 poll() 方法能够返回的记录数量，用于控制在轮询里需要处理的数据量

9. receive.buffer.bytes 和 send.buffer.bytes  
socket 在读写数据时用到的 TCP 缓冲区也可以设置大小。如果它们被设为 -1，就使用操 作系统的默认值。  

2.3.3  **主题相关配置 **  
1. num.partitions  
指定新创建的主题将包含多少个分区 。
tips： 如何选定分区数量？可以从以下几个因素考虑  
● 主题需要达到多大的吞吐量？  
● 从单个分区读取数据的最大吞吐量是多少？每个分区一般都会有一个消费者，如果消费者将数据写入数据库的速度不会超过每秒 50MB，那么从个分区读取数据的吞吐量不需要超过每秒 50MB。  
● 可以通过类似的方法估算生产者向单个分区写入数据的吞吐量，不过生产者的速度一般比消费者快得多，最好为生产者多估算一些吞吐量。  
● 每个 broker 包含的分区个数、可用的磁盘空间和网络带宽。  
● 单个 broker 对分区个数是有限制的，因为分区越多，占用的内存越多，完成首领选举需要的时间也越长。  

    2.log.retention.ms   
决定数据可以被保留多久，默认值为 168 小时  
    3. log.retention.bytes  
通过保留的消息字节数来判断消息是否过期 .比如有一个包含 8 个分区的主题，并且 log.retention.bytes 设为 1GB，那么这个主题最多可以保留 8GB 的数据。
    
   4.log.segment.bytes  
日志片段大小达到 log.segment.bytes 指定的上限（默认是 1GB）时，当前日志片段就会被关闭，一个新的日志片段被打开
   
   5.log.segment.ms  
   指定了多长时间之后日志片段会被关闭
 
  6.message.max.bytes  
单个消息的最大大小，默认值是 1000000，即1MB。如果生产者尝试发送的消息超过这个大小，不仅消息不会被接收，会收到broker返回的错误信息
tips1：这个值对性能有显著的影响。值越大，那么负责处理网络连接和请求的线程就需要花越多的时间来处理这些请求。它还会增加磁盘写入块的大小，从而影响 IO 吞吐量
tips2：消费者客户端设置的 fetch.message.max.bytes 必须与服务器端设置的消息大小进行协调 。如果这个值比 message.max.bytes 小，那么消费者就无法读取比较大的消息，导致出现消费者被阻塞的情况。  
2.3.4 **消息传递语义**  
	at most once   消息可能丢失但绝不重传  
	at least once	  消息可以重传但绝不丢失  
	exatly once  每个消息只传递一次  
producer：

consumer：  
  	![消息语义](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/消息语义.png)  

exatly once  ： two-phase commit  
2.3.5 **同步的定义**  
1. broker节点必须可以维护和 ZooKeeper 的连接，Zookeeper 通过心跳机制检查每个节点的连接。  
2. 如果节点是个 follower ，它必须能及时的同步 leader 的写操作，并且延时不能太久。  

当所有的分区上 in sync repicas 的commit 到log上时候，消息才认为是‘”commited“，消息只有 committed 消息才会给 consumer。
只要有至少一个同步中的节点存活，提交的消息就不会丢失。  
2.3.6 **可用性和持久性**  
min.insync.replicas 
unclean.leader.election.enable  

