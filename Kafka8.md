# Kafka生产者开发详解

kafka的生产者客户端开发的基本逻辑为：

1. 配置生产者客户端参数及创建相应的生产者实例。
2. 构建待发送的消息。
3. 发送消息。
4. 关闭生产者实例。

第一步必要的参数配置为`bootstrap.servers`，该参数指定连接 Kafka 集群所需要的 broker 地址清单。具体的内容格式为 host1:port1,host2:port2，可以设置一个或者多个地址，中间以逗号进行隔 开，此参数的默认值为 ""。

> 注意这里并非需要所有的 broker 地址，因为生产者会从给定的 broker 里查找其他 broker 的信息。不过建议至少要设置两个以上的 broker 地址信息，当其中任意一个宕机时，生产者仍然可以连接 到 Kafka 集群上。

```c++
// 创建Kafka Conf对象
m_config = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
if (m_config == NULL) {
  std::cout << "Create RdKafka Conf failed." << std::endl;
}
// 创建Topic Conf对象
m_topicConfig = RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC);
if (m_topicConfig == NULL) {
  std::cout << "Create RdKafka Topic Conf failed." << std::endl;
}
// 设置Broker属性
RdKafka::Conf::ConfResult errCode;
m_dr_cb = new ProducerDeliveryReportCb;
std::string errorStr;
// 传递情况报告
errCode = m_config->set("dr_cb", m_dr_cb, errorStr);
if (errCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed:" << errorStr << std::endl;
}
m_event_cb = new ProducerEventCb;
errCode = m_config->set("event_cb", m_event_cb, errorStr);
if (errCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed:" << errorStr << std::endl;
}
// 可以设置自己的分区策略
m_partitioner_cb = new HashPartitionerCb;
errCode = m_topicConfig->set("partitioner_cb", m_partitioner_cb, errorStr);
if (errCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed:" << errorStr << std::endl;
}
// 设置broker的地址
errCode = m_config->set("bootstrap.servers", m_brokers, errorStr);
if (errCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed:" << errorStr << std::endl;
}
```

接下来进行消息发送，librdkafka提供的异步的生产接口，异步的消费接口和同步的消息接口，没有同步的生产接口。 

```c++
// 创建生产者Producer，可以发布不同的主题
m_producer = RdKafka::Producer::create(m_config, errorStr);
if (m_producer == NULL) {
  std::cout << "Create Producer failed:" << errorStr << std::endl;
}
// 创建Topic对象
m_topic =
    RdKafka::Topic::create(m_producer, m_topicStr, m_topicConfig, errorStr);
if (m_topic == NULL) {
  std::cout << "Create Topic failed:" << errorStr << std::endl;
}
```

在发送消息时，同一个生产者可以发送多个主题的，在内部处理时根据传入的topic对象发送给对应的主题分区。

```c++
dKafka::ErrorCode errorCode = m_producer->produce(
    m_topic, // 指定发送到哪个主题
    RdKafka::Topic::PARTITION_UA, // 指定分区，如果为PARTITION_UA则通过partitioner_cb的回调选择合适的分区
    RdKafka::Producer::RK_MSG_COPY, // 消息拷贝
    payload,                        // 消息本身
    len,                            // 消息长度
    &key,                           // 消息key
    NULL // an optional application-provided per-message opaque pointer
         // that will be provided in the message delivery callback to let
         // the application reference a specific message
);
```

## 重要的生产者相关参数配置

### acks

用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消 息是成功写入的。 acks  是生产者客户端中一个非常重要的参数 ，它涉及消息的可靠性和吞吐量之间的权衡。acks 参数有3种类 型的值（都是字符串类型）。

- acks = 1。默认值即为 1。生产者发送消息之后，只要分区的 leader 副本成功写入消息，那么它就 会收到来自服务端的成功响应。如果消息无法写入 leader 副本，比如在 leader 副本崩溃、重新选举新的 leader 副本的过程中，那么生产者就会收到一个错误的响应，为了避免消息丢失，生产者 可以选择重发消息。如果消息写入 leader 副本并返回成功给生产者，且在被其他 follower 副本拉 取之前 leader 副本崩溃，那么此时消息还是会丢失，因为新选举的 leader 副本中并没有这条对应 的消息。
- acks = 0。生产者发送消息之后不需要等待任何服务端的响应。如果在消息从发送到写入 Kafka 的过程中出现了某些异常，导致 Kafka 并没有收到这条消息，那么 生产者也无从得知，消息也就丢失了。
- acks = -1 或 acks = all。生产者在消息发送之后，需要等待 ISR 中的所有副本都成功写入消息之后 才能够收到来自服务端的成功响应。但是并不意味着消息就一定可靠，因为 ISR 中可能只有 leader 副本，这样就退化成了 acks = 1 的 情况。要获得更高的消息可靠性需要配合 min.insync.replicas 等参数的联动。

> acks 设置为 1，是消息可靠性和吞吐量之间的折中方案。
>
> 在其他配置环境相同的情况下，acks 设置为 0 可以达到最大的吞吐量。
>
> 在其他配置环境相同的情况下，acks 设置为 -1 可以达到最强的可靠性。

```c++
// 注意 acks 参数配置的值是一个字符串类型，而不是整数类型
RdKafka::Conf *conf = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
 ConfResult ret = conf->set("acks", "1", errstr); 
ConfResult ret = conf->set("acks", "0", errstr); 
ConfResult ret = conf->set("acks", "all", errstr);
```

### max.request.size

这个参数用来限制生产者客户端能够发送的消息的最大值，默认值为 1048576 B，即 1 MB。一般情况下，这个默认值就可以满足大多数的应用场景了。不建议盲目地增大这个参数的配置值，尤其是在对 Kafka 整体脉络没有足够把控的时候。

因为这个参数还涉及一些其他参数的联动，比如 broker 端的 message.max.bytes 参数，如果配置错误可能会引起一些**不必要的异常**。比如讲 broker 端的 message.max.bytes 参数配置为 10， 而 max.request.size 参数配置为 20， 那么当我们发送一条消息大小为 15 的消息时，生产者客户端就会报出异常：The reqeust included a message larger than the max message size the server will accept.

```c++
errCode = conf->set("message.max.bytes", "10240000", errorStr);
```

### retries 和 retry.backoff.ms

retries 重试次数，默认0；retry.backoff.ms 重试间隔，默认100。

retries 参数用来配置生产者重试的次数，默认值为0，即发生异常的时候不进行任何的重试动作。消息在从生产者发出到成功写入服务器之前可能发生一些临时性的异常，比如网络抖动、Leader  副本的选举等，这种异常往往是可以自行恢复的，生产者可以通过配置 retries 大于 0 的值，以此 通过内部重试来恢复而不是一味的将异常抛给生产者的应用程序。

如果重试达到设定的次数，那么生产者就会放弃重试并返回异常。不过并不是所有的异常都是可以通过重试来解决的，比如消息太大，超过` max.request.size `参数 配置的值时，这种方式就不行了。重试还和另一个参数 `retry.backoff.ms `有关，这个参数的默认值为 100，它用来设定两次重试之 间的时间间隔，避免无效的频繁重试。

在配置 retries 和 retry.backoff.ms 之前，最好先估算一下可能的异常恢复时间，这样可以设定总 的重试时间大于这个异常恢复时间，以此来避免生产者过早地放弃重试。

Kafka 可以保证同一个分区中的消息时有序的，如果生产者按照一定的顺序发送消息，那么这些消息也会顺序的写入分区，进而消费者也可以按照 同样的顺序消费它们。对于某些应用来说，顺序性非常重要，比如 Mysql 的 binlog 传输，如果出现错误就会造成非常严 重的后果。如果讲 retries 参数设置为非零值，并且 `max.in.flight.requests.per.connection `参数配置为大于 1  的值，那么就会出现错序的现象：如果第一批次消息写入失败，而第二批次消息写入成功，那么生 产者会重试发送第一批次的消息，此时如果第一批次的消息写入成功，那么这两个批次的消息就出 现了错序。

一般而言，在需要保证顺序的场合建议把参数 `max.in.flight.requests.per.connection` 配置为  1，而不是把 retries 配置为 0. 不过这样也会影响整体的吞吐。 `max.in.flight.requests.per.connection = 1 `限制客户端在单个连接上能够发送的未响应请求的个数。设 置此值是1表示kafka broker在响应请求之前client不能再向同一个broker发送请求。（设置此参数是为了避免消息乱序）

### compression.type

这个参数用来指定消费的压缩方式，默认值为 “none”，即默认情况下，消息不会被压缩。该参数还可以配置为 “gzip”，“snappy”，“lz4”。

对消息进行压缩可以极大地减少网络传输量、降低网络 I/O ，从而提高整体的性能。消息压缩是一种使用时间换空间的优化方式，如果对时延有一定的要求，则不推荐对消息进行压缩。

### connection.max.idle.ms

这个参数用来指定在多久之后关闭闲置的连接，默认值时 540000 ms，即 9 分钟。

### linger.ms

这个参数用来指定生产者发送 Producer Batch 之前等待更多消息（ProducerRecord）加入  ProducerBatch 的时间，默认值为 0。生产者客户端会在 ProducerBatch 被填满或等待时间超过 linger.ms 值时发送出去。增大这个参数的值会增加消息的延迟，但是同时能提升一定的吞吐量。这个 linger.ms 参数与 TCP 协议中的 Nagle 算法有异曲同工之妙。

### receive.buffer.bytes

这个参数用来设置 Socket 接受消息缓冲区（SO_RECBUF）的大小，默认值为 32768（B），即 32  KB。如果设置为 -1，则使用操作系统的默认值。如果 Producer 与 Kafka 处于不同的机房，则可以适当调大这个参数值。

### send.buffer.bytes

这个参数用来设置 Socket 发送消息缓冲区（SO_SNDBUF）的大小，默认值为 131072 （B），即  128 KB。与 receive.buffer.bytes 参数一样，如果设置为 -1 ，则使用操作系统默认值。

### request.timeout.ms

这个参数用来配置 Producer 等待请求响应的最长时间，默认值为 30000 ms。请求超时之后可以选择进行重试。注意这个参数需要比 broker 端参数 replica.lag.time.max.ms 的值要大，这样可以减少因客户端重试而引起的消息重复的概率。

### client.id

用来设定 KafkaProducer 对应的客户端 id。默认值为 ""。

### batch.size

batch.size 是 producer 最重要的参数之一 ！它对于调优 producer **吞吐量和延时性能指标都有着非常重要**的作用 。producer 会将发往同一分区的多条消息封装进一个 batch中，当 batch 满了的时候， producer 会发送  batch 中的所有消息 。不过， producer并不总是等待batch满了才发送消息，很有可能当batch还有很 多空闲空间时 producer 就发送该 batch 。显然，batch 的大小就显得非常重要 。

通常来说，一个小的 batch 中包含的消息数很少，因而一次发送请求能够写入的消息数也很少，所以  producer 的吞吐量会很低；一个 batch 非常之巨大，那么会给内存使用带来极大的压力，因为不管是否能够填满，producer 都会为该batch 分配固定大小的内存。

因此batch.size 参数的设置其实是一种时间与空间权衡的体现 。batch.size 参数默认值是 16384 ，即  16KB 。这其实是一个非常保守的数字。 在实际使用过程中合理地增加该参数值，通常都会发现  producer 的吞吐量得到了相应的增加 。





