# Kafka消费者开发详解

## 客户端开发流程

1. 配置消费者客户端参数及创建相应的消费者实例
2. 订阅主题
3. 拉取消息并消费
4. 提交消息位移
5. 关闭消费者实例

## 创建消费者的参数配置

### bootstrap.servers

指定 broker 的地址清单，清单里不需要包含所有的 broker 地址，生产者会 从给定的 broker 里查找 broker 的信息。不过建议至少要提供两个 broker 的信息作为容错。

### group.id

consumer group 是 kafka 提供的可扩展且具有容错性的消费者机制。既然是一个组， 那么组内必然可以有多个消费者或消费者实例(consumer instance)，它们共享一个公共的 ID，即  group ID。组内的所有消费者协调在一起来消费订阅主题(subscribed topics)的所有分区 (partition)。

### auto.offset.reset

这个参数是针对新的 groupid 中的消费者而言的，当有新 groupid 的消费者来消费指定的 topic  时，对于该参数的配置，会有不同的语义。

- none：如果没有为消费者找到先前的offset的值，即没有自动维护偏移量，也没有手动维护偏移量，则抛出异常。
- earliest：在各分区下有提交的offset时：从offset处开始消费；在各分区下无提交的offset时：从头开始消费。
- latest：在各分区下有提交的offset时：从offset处开始消费；在各分区下无提交的offset时：从最新的数据开始消费。

```c++
std::string errorStr;
RdKafka::Conf::ConfResult errorCode;
m_config = RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL);
m_event_cb = new ConsumerEventCb;
errorCode = m_config->set("event_cb", m_event_cb, errorStr);
if (errorCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed: " << errorStr << std::endl;
}
m_rebalance_cb = new ConsumerRebalanceCb;
errorCode = m_config->set("rebalance_cb", m_rebalance_cb, errorStr);
if (errorCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed: " << errorStr << std::endl;
}
errorCode = m_config->set("group.id", m_groupID, errorStr);
if (errorCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed: " << errorStr << std::endl;
}
errorCode = m_config->set("bootstrap.servers", m_brokers, errorStr);
if (errorCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed: " << errorStr << std::endl;
}
// partition.assignment.strategy  range,roundrobin
errorCode = m_config->set("partition.assignment.strategy", "range", errorStr);
if (errorCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed: " << errorStr << std::endl;
}
m_topicConfig = RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC);
// 获取最新的消息数据
errorCode = m_topicConfig->set("auto.offset.reset", "latest", errorStr);
if (errorCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Topic Conf set failed: " << errorStr << std::endl;
}
errorCode = m_config->set("default_topic_conf", m_topicConfig, errorStr);
if (errorCode != RdKafka::Conf::CONF_OK) {
  std::cout << "Conf set failed: " << errorStr << std::endl;
}
m_consumer = RdKafka::KafkaConsumer::create(m_config, errorStr);
if (m_consumer == NULL) {
  std::cout << "Create KafkaConsumer failed: " << errorStr << std::endl;
}
std::cout << "Created consumer " << m_consumer->name() << std::endl;
```

## 订阅主题和分区及消费

订阅主题，可以订阅多个，也可以通过正则表达式方式一次订阅多个主题，比如 “topic-.*”， 则前缀为“topic-.”的主题都被订阅。

```c++
// 订阅主题, 可以订阅多个主题
void KafkaConsumer::pullMessage() {
  // 订阅Topic
  RdKafka::ErrorCode errorCode = m_consumer->subscribe(m_topicVector);
  if (errorCode != RdKafka::ERR_NO_ERROR) {
    std::cout << "subscribe failed: " << RdKafka::err2str(errorCode)
              << std::endl;
  }
  // 消费消息
  while (true) {
    RdKafka::Message *msg = m_consumer->consume(1000);
    msg_consume(msg, NULL);
    delete msg;
  }
}
void msg_consume(RdKafka::Message *msg, void *opaque) {
  switch (msg->err()) {
  case RdKafka::ERR__TIMED_OUT:
    std::cerr << "Consumer error: " << msg->errstr() << std::endl; // 超时
    break;
  case RdKafka::ERR_NO_ERROR: // 有消息进来
    std::cout << " Message in-> topic:" << msg->topic_name() << ", partition:["
              << msg->partition() << "] at offset " << msg->offset()
              << " key: " << msg->key()
              << " payload: " << (char *)msg->payload() << std::endl;
    break;
  default:
    std::cerr << "Consumer error: " << msg->errstr() << std::endl;
    break;
  }
}
```

## 位移提交

Consumer 需要向  Kafka 汇报自己的位移数据，这个汇报过程被称为提交位移（Committing  Offsets）。因为 Consumer 能够同时消费多个分区的数据，所以位移的提交实际上是在分区粒度上进行的，即 Consumer 需要为分配给它的每个分区提交各自的位移数据。

提交位移主要是为了表征 Consumer 的消费进度，这样当 Consumer 发生故障重启之后，就能够从  Kafka 中读取之前提交的位移值，然后从相应的位移处继续消费，从而避免整个消费过程重来一遍。从用户的角度来说，位移提交分为自动提交和手动提交；从 Consumer 端的角度来说，位移提交分为同 步提交和异步提交。

### 自动提交

自动提交默认全部为同步提交，自动提交相关参数有：

1. enable.auto.commit (bool) –– 如果为True，将自动定时提交消费者offset。默认为True。
2. auto.commit.interval.ms(int) ––  自动提交offset之间的间隔毫秒数。如果enable_auto_commit  为true，默认值为: 5000。

当设置 enable.auto.commit 为 true，Kafka 会保证在开始调用 poll 方法时，提交上次 poll 返回的 所有消息。从顺序上来说，poll 方法的逻辑是先提交上一批消息的位移，再处理下一批消息，因此它能 保证不出现消费丢失的情况。

但自动提交位移的一个问题在于，它可能会出现重复消费。如果设置 enable.auto.commit 为 true，Consumer 按照 auto.commit.interval.ms设置的值（默认5秒）自动提交一次位移。假设提交位移之后的 3 秒发生了 Rebalance 操作。在  Rebalance 之后，所有 Consumer 从上一次提交的位移处继续消费，但该位移已经是 3 秒前的位移数据了，故在 Rebalance 发生前 3 秒消费的所有数据都要重新再消费一次。虽然能够通过减 少 auto.commit.interval.ms 的值来提高提交频率，但这么做只能缩小重复消费的时间窗口，不 可能完全消除它。这是自动提交机制的一个缺陷。

> kafka 版本 2.11）实际测试中，未发现重复消费的情况， 而是会等待所有消费者消费完当前消息，或者等待消费者超时（等待过程中会报如下 warning）， 之后才会 reblance 。

### 手动提交

手动提交可以自己选择是同步提交（commitSync）还是异步提交（commitAsync ）。commitAsync  不能够替代 commitSync。commitAsync 的问题在于，出现问题时它不会自动重试。因为它是异步操作，倘若提交失败后自动重试，那么它重试时提交的位移值可能早已经“过期” 或不是最新值了。因此，异步提交的重试其实没有意义，所以commitAsync 是不会重试的。

手动提交，我们需要将 commitSync 和 commitAsync 组合使用才能到达最理想的效果，这是因为：

1. 可以利用同步提交 commitSync 的自动重试来规避那些瞬时错误，比如网络的瞬时抖动，Broker 端 GC  等。因为这些问题都是短暂的，自动重试通常都会成功，因此，当不想自己重试，而是希望  Kafka Consumer 帮我们做这件事。
2. 但是不希望程序总处于阻塞状态，影响 TPS，所以使用异步提交。

所以通常需要同时使用 commitSync() 和 commitAsync()。对于常规性、阶段性的手动提交，可以调用 commitAsync() 避免程序阻塞；而在 Consumer 要关闭前，一般需要调用 commitSync() 方法执行同步阻塞式的位移提交，以确保  Consumer 关闭前能够保存正确的位移数据。将两者结合后，我们既实现了异步无阻塞式的位移管理，也确保了 Consumer 位移的正确性。

### 手动提交和自动提交中的 reblance

- 如果设置为手动提交，当集群满足 reblance 的条件时，集群会直接 reblance，不会等待所 有消息被消费完，这会导致所有未被确认的消息会重新被消费，会出现重复消费的问题。
- 如果设置为自动提交，当集群满足 reblance 的条件时，集群不会马上 reblance，而是会等待所有消费者消费完当前消息，或者等待消费者超时（等待过程中会报如下 warning）， 之后才会 reblance。

### 提交位移相关函数

- ErrorCode commitSync(); 提交当前分配分区的位移，同步操作，会阻塞直到位移被提交或提交失败。如果注册了RdKafka::OffsetCommitCb回调函数，其会在KafkaConsumer::consume()函数内调用并提交位移。
- ErrorCode commitAsync();异步提交位移。
- ErrorCode commitSync(Message *message); 基于消息对单个topic+partition对象同步提交位。
- virtual ErrorCode commitSync (std::vector &offsets) = 0;对指定多个TopicPartition同步提交位移。
- ErrorCode commitAsync(Message *message); 基于消息对单个TopicPartition异步提交位移。
- virtual ErrorCode commitAsync (const std::vector &offsets) = 0; 对多个TopicPartition异步提交位移。

## 消费Rebalance机制

当kafka遇到如下四种情况的时候，kafka会触发Rebalance机制：

1. 消费组成员发生了变更，比如有新的消费者加入了消费组组或者有消费者宕机。
2. 消费者无法在指定的时间之内完成消息的消费。
3. 消费组订阅的Topic发生了变化。
4. 订阅的Topic的partition发生了变化。

后面两个通常都是运维的主动操作，所以它们引发的 Rebalance 大都是不可避免的。主要考虑的是因为组成员数量变化而引发的 Rebalance 该如何避免。相关参数如下：

- ession.timeout.ms 表示 consumer 向 broker 发送心跳的超时时间。例如 session.timeout.ms  = 180000 表示在最长 180 秒内 broker 没收到 consumer 的心跳，那么 broker 就认为该  consumer 死亡了，会启动 rebalance。
- heartbeat.interval.ms 表示 consumer 每次向 broker 发送心跳的时间间隔。 heartbeat.interval.ms = 60000 表示 consumer 每 60 秒向 broker 发送一次心跳。一般来说， session.timeout.ms 的值是 heartbeat.interval.ms 值的 3 倍以上。
- max.poll.interval.ms 表示 consumer 每两次 poll 消息的时间间隔。简单地说，其实就是  consumer 每次消费消息的时长。如果消息处理的逻辑很重，那么时长就要相应延长。否则如果时 间到了 consumer 还么消费完，broker 会默认认为 consumer 死了，发起 rebalance。
- max.poll.records 表示每次消费的时候，获取多少条消息。获取的消息条数越多，需要处理的时 间越长。所以每次拉取的消息数不能太多，需要保证在 max.poll.interval.ms 设置的时间内能消费 完，否则会发生 rebalance。

由此可知，可能会导致重平衡的点就是，消费者心跳超时和消费者消费数据超时。解决方法已经很明朗了，就是适当调参，心跳超时就调整session.timeout.ms和 heartbeat.interval.ms。session.timeout.ms 决定了 Consumer 存活性的时间间隔。

一些常用的配置建议如下：

1. 设置 session.timeout.ms = 6s。
2. 设置 heartbeat.interval.ms = 2s。
3. 要保证 Consumer 实例在被判定为“dead”之前，能够发送至少 3 轮的心跳请求，即  session.timeout.ms >= 3 * heartbeat.interval.ms。
4. 对于消费处理超时问题。一般是增加消费者处理的时间（max.poll.interval.ms），减少每次处理 的消息数（max.poll.records）。
5. 建议 max.poll.records 参数要远小于当前消费组 的消费能力（records < 单个线程每秒消费的条数 * 消费线程的个数 * session.timeout的秒数）。

## 其他重要的消费者参数

在 KafkaConsumer 中，除了前面讲的必要的客户端参数，大部分的参数都有合理的默认值，一般也不需要去修改它们。不过了解这些参数可以更好地开发消费者客户端，其中还有一些重要的参数涉及程序的可用性和性能，如果能够熟练掌握它们，也可以在编写相关的程序时能够更好地进行性能调优与故障排查。

### fetch.min.bytes

该参数用来配置 Consumer 在一次拉取请求（调用 poll() 方法）中能从 Kafka 中拉取的最小数据量，默认值为1（B）。Kafka 在收到 Consumer 的拉取请求时，如果返回给 Consumer 的数据量小于这个参数所配置的值，那么它就需要进行等待，直到数据量满足这个参数的配置大小。可以适当调大这个参数的值以提高一定的吞吐量，不过也会造成额外的延迟（latency），对于延迟敏感的应用可能就不可取了。  

### fetch.max.bytes  

该参数与 fetch.min.bytes 参数对应，它用来配置 Consumer 在一次拉取请求中从Kafka中拉取的最大数据量，默认值为52428800（B），也就是50MB。  

如果这个参数设置的值比任何一条写入 Kafka 中的消息要小，那么会不会造成无法消费呢？很多资料对此参数的解读认为是无法消费的，比如一条消息的大小为10B，而这个参数的值是1（B），既然此参数设定的值是一次拉取请求中所能拉取的最大数据量，那么显然1B<10B，所以无法拉取。这个观点是错误的，该参数设定的不是绝对的最大值，如果在第一个非空分区中拉取的第一条消息大于该值，那么该消息将仍然返回，以确保消费者继续工作。也就是说，上面问题的答案是可以正常消费。  

与此相关的，Kafka 中所能接收的最大消息的大小通过服务端参数 message.max.bytes（对应于主题端参数 max.message.bytes）来设置。  

### fetch.max.wait.ms  

这个参数也和 fetch.min.bytes 参数有关，如果 Kafka 仅仅参考 fetch.min.bytes 参数的要求，那么有可能会一直阻塞等待而无法发送响应给 Consumer，显然这是不合理的。fetch.max.wait.ms 参数用于指定 Kafka 的等待时间，默认值为500（ms）。如果 Kafka 中没有足够多的消息而满足不了fetch.min.bytes 参数的要求，那么最终会等待500ms。这个参数的设定和 Consumer 与 Kafka 之间的延迟也有关系，如果业务应用对延迟敏感，那么可以适当调小这个参数。  

### max.partition.fetch.bytes  

这个参数用来配置从每个分区里返回给 Consumer 的最大数据量，默认值为1048576（B），即1MB。这个参数与 fetch.max.bytes 参数相似，只不过前者用来限制一次拉取中每个分区的消息大小，而后者用来限制一次拉取中整体消息的大小。同样，如果这个参数设定的值比消息的大小要小，那么也不会造成无法消费，Kafka 为了保持消费逻辑的正常运转不会对此做强硬的限制。  

### max.poll.records  

这个参数用来配置 Consumer 在一次拉取请求中拉取的最大消息数，默认值为500（条）。如果消息的大小都比较小，则可以适当调大这个参数值来提升一定的消费速度。  如果用户的消息处理逻辑很轻量，默认的 500 条消息通常不能满足实际的消息处理速度 。  

### connections.max.idle.ms  

这个参数用来指定在多久之后关闭闲置的连接，默认值是540000（ms），即9分钟。  

### exclude.internal.topics  

Kafka 中有两个内部的主题： consumer_offsets 和 transaction_state。exclude.internal.topics 用来指定 Kafka 中的内部主题是否可以向消费者公开，默认值为 true。如果设置为 true，那么只能使用subscribe(Collection)的方式而不能使用 subscribe(Pattern)的方式来订阅内部主题，设置为 false 则没有这个限制。  

### receive.buffer.bytes  

这个参数用来设置 Socket 接收消息缓冲区（SO_RECBUF）的大小，默认值为65536（B），即64KB。如果设置为-1，则使用操作系统的默认值。如果 Consumer 与 Kafka 处于不同的机房，则可以适当调大这个参数值。  

### send.buffer.bytes  

个参数用来设置Socket发送消息缓冲区（SO_SNDBUF）的大小，默认值为131072（B），即128KB。与receive.buffer.bytes参数一样，如果设置为-1，则使用操作系统的默认值。  

### request.timeout.ms  

这个参数用来配置 Consumer 等待请求响应的最长时间，默认值为30000（ms）。  

### metadata.max.age.ms  

这个参数用来配置元数据的过期时间，默认值为300000（ms），即5分钟。如果元数据在此参数所限定的时间范围内没有进行更新，则会被强制更新，即使没有任何分区变化或有新的 broker 加入。  

### reconnect.backoff.ms  

这个参数用来配置尝试重新连接指定主机之前的等待时间（也称为退避时间），避免频繁地连接主机，默认值为50（ms）。这种机制适用于消费者向 broker 发送的所有请求。  

### retry.backoff.ms  

这个参数用来配置尝试重新发送失败的请求到指定的主题分区之前的等待（退避）时间，避免在某些故障情况下频繁地重复发送，默认值为100（ms）  

### isolation.level  

这个参数用来配置消费者的事务隔离级别。字符串类型，有效值为“read_uncommitted”和“read_committed”，表示消费者所消费到的位置。如果设置为“read_committed”，那么消费者就会忽略事务未提交的消息，即只能消费到LSO（LastStableOffset）的位置。默认情况下为“read_uncommitted”，即可以消费到 HW（High Watermark）处的位置。  

### session.timeout.ms  

该参数是非常重要的参数之一 ！  session.timeout.ms 是 consumer group 检测组内成员发送崩溃的时间。

假设你设置该参数为 5 分钟，那么当某个 group 成员突然崩溃了（比如被 kill -9 或岩机）， 管理group 的 Kafka 组件（即消费者组协调者，也称 group  coordinator）有可能需要 5 分钟才能感知到这个崩溃。显然缩短这个时间，可以让coordinator 能够更快地检测到 consumer 失败 。  

这个参数还有另外一重含义 ：consumer 消息处理逻辑的最大时间。  如果consumer 两次 poll 之间的间隔超过了该参数所设置的阑值，那么 coordinator 就会认为这个consumer 己经追不上组内其他成员的消费进度了，因此会将该 consumer 实例“踢出”组，该consumer 负责的分区也会被分配给其他 consumer。  

在最好的情况下，这会导致不必要的 rebalance，因为 consumer 需要重新加入 group 。更糟的是，对于那些在被踢出 group 后处理的消息， consumer 都无法提交位移一一这就意味着这些消息在rebalance 之后会被重新消费一遍。如果一条消息或一组消息总是需要花费很长的时间处理，那么consumer 甚至无法执行任何消费，除非用户重新调整参数 。鉴于以上的“窘境”， Kafka 社区于 0 .10.1.0 版本对该参数的含义进行了拆分 。 在该版本及以后的版本中， session.timeout.ms 参数被明确为“ coordinator 检测失败的时间” 。  

因此在实际使用中，用户可以为该参数设置一个比较小的值，让 coordinator 能够更快地检测consumer 崩溃的情况，从而更快地开启 rebalance，避免造成更大的消费滞后（ consumer lag ） 。目前该参数的默认值是 10 秒。  

### max.poll.interval.ms  

消费者组中的一员在拉取消息时如果超过了设置的最大拉取时间，则会认为消费者消费消息失败，kafka会重新进行重新负载均衡，以便把消息分配给另一个消费组成员。  

在一个典型的 consumer 使用场景中，用户对于消息的处理可能需要花费很长时间。这个参数就是用于设置消息处理逻辑的最大时间的 。 假设用户的业务场景中消息处理逻辑是把消息、“落地”到远程数据库中，且这个过程平均处理时间是 2 分钟，那么用户仅需要将 max.poll.interval.ms 设置为稍稍大于 2 分钟的值即可，而不必为 session. timeout.ms 也设置这么大的值。  

通过将该参数设置成实际的逻辑处理时间再结合较低的 session.timeout.ms 参数值，consumer group既实现了快速的 consumer 崩溃检测，也保证了复杂的事件处理逻辑不会造成不必要的 rebalance 。  

### heartbeat.interval.ms  

该参数和 request.timeout.ms 、max.poll.interval.ms 参数是最难理解的 consumer 参数 。从表面上看，该参数似乎是心跳的问隔时间，但既然己经有了上面的 session.timeout.ms 用于设置超时，为何还要引入这个参数呢？  这里的关键在于要搞清楚 consumer group 的其他成员，如何得知要开启新一轮 rebalance 。

当coordinator 决定开启新一轮 rebalance 时，它会将这个决定以REBALANCE_IN_PROGRESS 异常的形式“塞进” consumer 心跳请求的 response 中，这样其他成员拿到 response 后才能知道它需要重新加入group。显然这个过程越快越好，而heartbeat. interval.ms 就是用来做这件事情的 。比较推荐的做法是设置一个比较低的值，让 group 下的其他 consumer 成员能够更快地感知新一轮rebalance. 开启了。  

注意，该值必须小于 session.timeout.ms ！这很容易理解，毕竟如果 consumer 在session.timeout.ms 这段时间内都不发送心跳， coordinator 就会认为它已经 dead，因此也就没有必要让它知晓 coordinator 的决定了。  
