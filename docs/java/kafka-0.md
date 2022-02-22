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
2.1.1 生产者发送消息的主要步骤
 ![消费者群组](https://github.com/xiechengsiii/Java_Notes/blob/master/pics/produceStep.png)  
2.1.2 创建生产者
必选属性
1. bootstrap.servers
2. broker地址
3. key.serializer
4. key的序列化器
5. value.serializer
6. value的序列化器
创建一个新的生产者的简单示例，只指定了必要的属性，其他使用默认设置 ：
```Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        //acks = 0, 1, all
        props.put("acks", "0");
        //配置为大于0的值的话，客户端会在消息发送失败时重新发送。
        props.put("retries", 0);
        //当多条消息需要发送到同一个分区时，生产者会尝试合并网络请求。
        props.put("batch.size", 16384);
        props.put("key.serializer", StringSerializer.class.getName());
        props.put("value.serializer", StringSerializer.class.getName());
        this.producer = new KafkaProducer<String, String>(props);```
