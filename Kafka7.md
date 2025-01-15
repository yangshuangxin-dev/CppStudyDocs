# Kafka的broker参数详解

## broker重要参数详解

broker 端参数需要在 Kafka 目录下的 config/server.properties 文件中进行设置。当前对于绝大多数的  broker 端参数而言， Kafka 尚不支持动态修改一一这就是说，如果要新增、修改，抑或是删除某些  broker 参数的话，需要重启对应的 broker 服务器。

### broker.id

Kafka 使用唯一的一 个整数来标识每个 broker ，这就是 broker.id ；该参数默认是-1 。如果不指定，  Kafka 会自动生成一个唯一值；总之，不管用户指定什么都必须保证该值在 Kafka 集群中是唯一的，不 能与其他 broker 冲突 。在实际使用中，推荐使用从0开始的数字序列，如0、1、2……

### log.dirs

该参数指定了 Kafka 持久化消息的目录；非常重要的参数！若不设置该参数， Kafka 默认使用tmp/kafka-logs 作为消息保存的目录(商用一定要自己设置目录)。把消息保存在 tmp 录下，生产环境中是不可取的。若待保存的消息数量非常多，那么最好确保该文件夹 下有充足的磁盘空间。该参数可以设置多个目录，以逗号分隔，比如/home/kafka1,/home/kafka2 。在实际使用过程中， 指定多个目录的做法通常是被推荐的，因为这样 Kafka 可以把负载均匀地分配到多个目录下。（这里的“均匀”是根据目录下的分区数进行比较的，而不是根据实际的磁盘空间）

若用户机器上有N块物理硬盘，那么设置多个目录（须挂载在不同磁盘上的目录）是一个很好的选择。N  个磁头可以同时执行写操作，极大地提升了吞吐量。

### zookeeper.connect

同样是非常重要的参数。主要是在zookeeper中，kafka的配置数据，有一个公共开始的根节点。该参数没有默认的值，如果不配置，则使用zookeeper的/目录；例如：zkl1:2181,zk2:2181,zk3:2181/kafka_clusterl；/kafka_cluster就是kafka的配置的目录；配置了可以起到很好的隔离效果。这样管理 Kafka 集群将变得更加容易。

### listeners

broker 监听器的 csv (comma-separated values)列表，格式是［协议://:主机名:端口]，[协议]:[// 主机 名:端口]。该参数主要用于客户端连接 broker 使用，可以认为是 broker 端开放给 clients的监听端口 如果不指定主机名，则表示绑定默认网卡。如果0.0.0.0，则表示绑定所有网卡。 Kafka 当前支持的协议类型包括PLAINTEXT、SSL以及 SASL SSL 等。

### advertised.listeners

跟 listeners 类似，该参数也是用于发布给 clients 的监昕器；不过该参数主要用于 IaaS 环境，比如云上 的机器通常都配有多块网卡（私网网卡和公网网卡）。对于这种机器，用户可以设置该参数绑定公网 IP 供外部 clients 使用，然后配置上面的 listeners 来绑定 私网 IP供broker间通信使用。

当然不设置该参数也是可以的，只是云上的机器很容易出现 clients 无法获取数据的问题，原因就是  listeners 绑定的是默认网卡，而默认网卡通常都是绑定私网的。在实际使用场景中，对于配有多块网卡的机器而言，这个参数通常都是需要配置的。

### unclean.leader.election.enable

是否开启 unclean leader 选举。Kafka 社区在1.0.0 版本才正式将该参数默认值调整为 false 。即表明如果发生这种情况， Kafka 不允许从剩下存活的非 ISR 副本中选择一个当 leader。因为如果 true，这样做固然可以让 Kafka 继续提供服务给 clients ，但会造成消息数据的丢失，正式环境中，数据 不丢失是基本的业务需求。

> ISR 全称是in-sync replica ，翻译过来就是与 leader replica 保持同步的 replica 集合 ;一个partition 可以配置N个replica ，那么这是否就意味着该 partition可以容忍 N-1 replica 失效而不丢失数据呢？答案是“否”！
>
> Kafka partition 动态维护一个replica 集合。该集合中的所有 replica 保存的消息日志都与leader  replica 保持同步状态。只有这个集合中的 replica 才能被选举为 leader ，也只有该集合中所有 replica  都接收到了同一条消息， Kafka 才会将该消息置于“己提交”状态，即认为这条消息发送成功。
>
> Kafka 中只要这个集合中至少存在一个 replica ，那些“己提交”状态的消息就不会丢失；Kafka 对于没有提交成功的消息不做任何保证，只保证在 ISR 存活的情况下“己提交”的消息不会丢失。正 常情况下，partition 的所有 replica （含 leader replica ）都应该与 leader replica 保持同步，即所有  replica 都在 ISR 中。
>
> 因为各种各样的原因，一小部分 replica 开始落后于 leader replica的进度 。当滞后 一定程度时， Kafka  会将这些 replica “踢”出 ISR；相反地，当这replica 重新追上leader进度时候，kafka会再次把它们加回 ISR中。

### delete.topic.enable

是否允许 Kafka 删除 topic 。默认情况下， Kafka 集群允许用户删除topic 及其数据。这样当用户发起删除 topic 操作时， broker  端会执行 topic 删除逻辑。在实际生产环境中可以发现允许 Kafka 删除 topic 其实是一个很方便的功能，再加上自Kafka 0.0 新增的  ACL 权限特性，以往对于误操作和恶意操作的担心完全消失了，因此设置该参数为 true 是推荐的做法。

### log.retention. {hours|minutes|ms}

这组参数控制了消息数据的留存时间；默认的留存时间是7天(168小时)；即 Kafka 只会保存最近 7天的 数据，井自动删除 7天前的数据。当前较新版本的Kafka 会根据消息的时间戳信息进行留存与否的判断；老版本消息格式没有时间戳，  Kafka 会根据日志文件的最近修改时间（ last modified time ）进行判断。三个参数如果同时设置，优先选取ms的设置，minutes次之，hours最后。

### log.retention.bytes

这个参数定义了空间维度上的留存策略；参数默认值是-1，表示 Kafka 永远不会根据消息日志文件总大 小来删除日志。对于大小超过该参数的分区日志而言， Kafka 会自动清理该分区的过期日志段文件。

### min.insync.replicas

该参数表示kafka存储的最小副本数，producer发送数据的时候，指定了 broker 端必须成功响应clients 消息发送的最少副本数。该参数其实是与 producer 端的 acks 参数配合使用的 ；并且min.insync.replicas 也只有在 acks=-1 时 才有意义。acks=-1 表示 producer端寻求最高等级的持久化保证；

假如 broker 端无法满足该条件，则 clients 的消息发送并不会被视为成功。它与 acks 配合使用可以令  Kafka集群达成最高等级的消息持久化。例如假设某个topic 的每个分区的副本数是3 ，那么推荐设置该参数为 2，这样我们就能够容忍 一台broker 宕机而不影响服务；若设置参数为3 ，那么只要任何一台 broker 岩机，整个Kafka 集群将 无法继续提供服务。

### num.network.threads (网络数据读取)

一个 broker 在后台用于处理网络请求的线程数，默认是3 。broker启动时会创建多个线程处理来自其他broker和clients 发送过来的各种请求。会将接收到的请求转 发到后面的处理线程中。在真实的环境中，用户需要不断地监控NetworkProcessorAvgldlePercent JMX 指标; 如果该指标持续低于0.3 ;建议适当增加该参数的值。

### num.io.threads (处理我们读取到网络数据)

这个参数就是控制 broker 端实际处理网络请求的线程数，默认值是8。Kafka broker 默认创建 8个线程以轮询方式不停地监昕转发过来的网络请求井进行实时处理。 Kafka 同 样也为请求处理提供了一个 JMX 监控指标 RequestHandler A vgldlePercent。如果发现该指标持续低于 0.3 ，则可以考虑适当增加该参数的值。

### message.max. bytes

Kafka broker 能够接收的最大消息大小，默认是 977kB；还不到1MB ，可见是非常小的。在实际使用场景中，突破 1MB 大小的消息十分常见，因此用户有必要综合考虑 Kafka 集群可能处理的 最大消息尺寸井设置该参数值。

### log.segment.bytes (分片的概念)

kafka数据文件的大小（默认1GB），确保这个数值大于一个消息的长度。一般说来使用默认值即可（一 般一个消息很难大于1G，因为这是一个消息系统，而不是文件系统）。