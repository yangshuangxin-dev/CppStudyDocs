# Kafka 使用的基本操作流程

## 生产者操作

1. 创建Kafka配置实例。`RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL) `

2. 创建Topic配置实例 。`RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC)`

3. 设置Kafka配置实例Broker属性 。`RdKafka::Conf::ConfResult RdKafka::Conf::set(const std::string &name, const std::string &value,  std::string &errstr) `

4. 设置Topic配置实例属性。`RdKafka::Conf::ConfResult RdKafka::Conf::set (const std::string &name, const std::string &value,  std::string &errstr) `

5. 注册回调函数。

6. 创建Kafka Producer客户端实例。`static RdKafka::Producer* RdKafka::Producer::create(RdKafka::Conf *conf, std::string &errstr);`(conf为Kafka配置实例 )

7. 创建Topic实例。` static RdKafka::Topic* RdKafka::Topic::create(RdKafka::Handle *base, const std::string &topic_str, RdKafka::Conf *conf,  std::string &errstr);`(conf为Topic配置实例)

8. 生产消息。` RdKafka::ErrorCode RdKafka::Producer::produce(RdKafka::Topic *topic,  int32_t partition,  int msgflags, void *payload, size_t len,  const std::string *key,  void *msg_opaque);`

9. 阻塞等待Producer生产消息完成 。`int RdKafka::Producer::poll (int timeout_ms);`

10. 等待Produce请求完成。`RdKafka::ErrorCode RdKafka::Producer::flush(int timeout_ms);`

11. 销毁Kafka Producer客户端实例。`int RdKafka::wait_destroyed(int timeout_ms);`

```c++
// 注册回调函数
//事件是从RdKafka传递错误、统计信息、日志等消息到应用程序的通用接口
Conf::ConfResult RdKafka::Conf::set ("event_cb", RdKafka::EventCb *dr_cb, std::string &errstr);
 //SocketCb回调函数用于打开一个Socket套接字
Conf::ConfResult RdKafka::Conf::set ("socket_cb", RdKafka::SocketCb *socket_cb, std::string &errstr);
 //Open回调函数用于使用flags、mode打开指定path的文件
Conf::ConfResult RdKafka::Conf::set ("open_cb", RdKafka::OpenCb *open_cb, std::string &errstr);
 //用于RdKafka::KafkaConsunmer的组再平衡回调函数
Conf::ConfResult RdKafka::Conf::set ("rebalance_cb", RdKafka::RebalanceCb *rebalance_cb, std::string &errstr);
 //用于消费者组的位移提交回调函数
Conf::ConfResult RdKafka::Conf::set ("offset_commit_cb", RdKafka::OffsetCommitCb *offset_commit_cb, std::string &errstr);
 // 对消费的每条消息会调用ConsumeCb回调函数
Conf::ConfResult RdKafka::Conf::set ("consume_cb", RdKafka::ConsumeCb *consume_cb, std::string &errstr);
// 分区策略回调函数需要注册到Topic配置实例：
Conf::ConfResult RdKafka::Conf::set ("partitioner_cb", RdKafka::PartitionerCb *dr_cb, std::string &errstr);
 Conf::ConfResult RdKafka::Conf::set ("partitioner_key_pointer_cb", RdKafka::PartitionerKeyPointerCb *dr_cb, std::string &errstr);
```

完整代码：

```c++
// KafkaProducer.h
#pragma once
#include "rdkafkacpp.h"
#include <iostream>
#include <string>
class ProducerDeliveryReportCb : public RdKafka::DeliveryReportCb {
public:
  void dr_cb(RdKafka::Message &message) {
    if (message.err())
      std::cerr << "Message delivery failed: " << message.errstr() << std::endl;
    else
      std::cerr << "Message delivered to topic " << message.topic_name() << " ["
                << message.partition() << "] at offset " << message.offset()
                << std::endl;
  }
};
class ProducerEventCb : public RdKafka::EventCb {
public:
  void event_cb(RdKafka::Event &event) {
    switch (event.type()) {
    case RdKafka::Event::EVENT_ERROR:
      std::cout << "RdKafka::Event::EVENT_ERROR: "
                << RdKafka::err2str(event.err()) << std::endl;
      break;
    case RdKafka::Event::EVENT_STATS:
      std::cout << "RdKafka::Event::EVENT_STATS: " << event.str() << std::endl;
      break;
    case RdKafka::Event::EVENT_LOG:
      std::cout << "RdKafka::Event::EVENT_LOG " << event.fac() << std::endl;
      break;
    case RdKafka::Event::EVENT_THROTTLE:
      std::cout << "RdKafka::Event::EVENT_THROTTLE " << event.broker_name()
                << std::endl;
      break;
    }
  }
};
class HashPartitionerCb : public RdKafka::PartitionerCb {
public:
  int32_t partitioner_cb(const RdKafka::Topic *topic, const std::string *key,
                         int32_t partition_cnt, void *msg_opaque) {
    char msg[128] = {0};
    sprintf(msg, "HashPartitionerCb:[%s][%s][%d]", topic->name().c_str(),
            key->c_str(), partition_cnt);
    std::cout << msg << std::endl;
    return generate_hash(key->c_str(), key->size()) % partition_cnt;
  }

private:
  static inline unsigned int generate_hash(const char *str, size_t len) {
    unsigned int hash = 5381;
    for (size_t i = 0; i < len; i++)
      hash = ((hash << 5) + hash) + str[i];
    return hash;
  }
};
class KafkaProducer {
public:
  /**
   * @brief KafkaProducer
   * @param brokers
   * @param topic
   * @param partition
   */
  explicit KafkaProducer(const std::string &brokers, const std::string &topic,
                         int partition);
  /**
   * @brief push Message to Kafka
   * @param str, message data
   */
  void pushMessage(const std::string &str, const std::string &key);
  ~KafkaProducer();

protected:
  std::string m_brokers;         // Broker列表，多个使用逗号分隔
  std::string m_topicStr;        // Topic名称
  int m_partition;               // 分区
  RdKafka::Conf *m_config;       // Kafka Conf对象
  RdKafka::Conf *m_topicConfig;  // Topic Conf对象
  RdKafka::Topic *m_topic;       // Topic对象
  RdKafka::Producer *m_producer; // Producer对象
  RdKafka::DeliveryReportCb *m_dr_cb;
  RdKafka::EventCb *m_event_cb;
  RdKafka::PartitionerCb *m_partitioner_cb;
};

//  KafkaProducer.cpp
#include "KafkaProducer.h"
KafkaProducer::KafkaProducer(const std::string &brokers,
                             const std::string &topic, int partition) {
  m_brokers = brokers;
  m_topicStr = topic;
  m_partition = partition;
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
  errCode = m_config->set("dr_cb", m_dr_cb, errorStr);
  if (errCode != RdKafka::Conf::CONF_OK) {
    std::cout << "Conf set failed:" << errorStr << std::endl;
  }
  m_event_cb = new ProducerEventCb;
  errCode = m_config->set("event_cb", m_event_cb, errorStr);
  if (errCode != RdKafka::Conf::CONF_OK) {
    std::cout << "Conf set failed:" << errorStr << std::endl;
  }
  m_partitioner_cb = new HashPartitionerCb;
  errCode = m_topicConfig->set("partitioner_cb", m_partitioner_cb, errorStr);
  if (errCode != RdKafka::Conf::CONF_OK) {
    std::cout << "Conf set failed:" << errorStr << std::endl;
  }
  errCode = m_config->set("statistics.interval.ms", "10000", errorStr);
  if (errCode != RdKafka::Conf::CONF_OK) {
    std::cout << "Conf set failed:" << errorStr << std::endl;
  }
  errCode = m_config->set("message.max.bytes", "10240000", errorStr);
  if (errCode != RdKafka::Conf::CONF_OK) {
    std::cout << "Conf set failed:" << errorStr << std::endl;
  }
  errCode = m_config->set("bootstrap.servers", m_brokers, errorStr);
  if (errCode != RdKafka::Conf::CONF_OK) {
    std::cout << "Conf set failed:" << errorStr << std::endl;
  }
  // 创建Producer
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
}
void KafkaProducer::pushMessage(const std::string &str,
                                const std::string &key) {
  int32_t len = str.length();
  void *payload = const_cast<void *>(static_cast<const void *>(str.data()));
  RdKafka::ErrorCode errorCode = m_producer->produce(
      m_topic, RdKafka::Topic::PARTITION_UA, RdKafka::Producer::RK_MSG_COPY,
      payload, len, &key, NULL);
  m_producer->poll(0);
  if (errorCode != RdKafka::ERR_NO_ERROR) {
    std::cerr << "Produce failed: " << RdKafka::err2str(errorCode) << std::endl;
    if (errorCode == RdKafka::ERR__QUEUE_FULL) {
      m_producer->poll(1000);
    }
  }
}
KafkaProducer::~KafkaProducer() {
  while (m_producer->outq_len() > 0) {
    std::cerr << "Waiting for " << m_producer->outq_len() << std::endl;
    m_producer->flush(5000);
  }
  delete m_config;
  delete m_topicConfig;
  delete m_topic;
  delete m_producer;
  delete m_dr_cb;
  delete m_event_cb;
  delete m_partitioner_cb;
}

// main.cpp
#include "KafkaProducer.h"
#include <iostream>
using namespace std;
int main() {
  // 创建Producer
  KafkaProducer producer("192.168.0.105:9092", "test", 0);
  for (int i = 0; i < 10000; i++) {
    char msg[64] = {0};
    sprintf(msg, "%s%4d", "Hello RdKafka", i);
    // 生产消息
    char key[8] = {0};
    sprintf(key, "%d", i);
    producer.pushMessage(msg, key);
  }
  RdKafka::wait_destroyed(5000);
}

```

```cmake
cmake_minimum_required(VERSION 2.8)
 project(KafkaProducer)
 set(CMAKE_CXX_STANDARD 11)
 set(CMAKE_CXX_COMPILER "g++")
 set(CMAKE_CXX_FLAGS "-std=c++11 ${CMAKE_CXX_FLAGS}")
 set(CMAKE_INCLUDE_CURRENT_DIR ON)
 # Kafka头文件路径
include_directories(/usr/local/include/librdkafka)
 # Kafka库路径
link_directories(/usr/local/lib)
aux_source_directory(. SOURCE)
add_executable(${PROJECT_NAME} ${SOURCE})
TARGET_LINK_LIBRARIES(${PROJECT_NAME} rdkafka++)
```

## 消费者操作

RdKafka提供了两种消费者API，低级API的Consumer和高级API的KafkaConsumer，本文使用 KafkaConsumer。

1. 创建Kafka配置实例。`RdKafka::Conf::create(RdKafka::Conf::CONF_GLOBAL) `

2. 创建Topic配置实例 。`RdKafka::Conf::create(RdKafka::Conf::CONF_TOPIC) `

3. 设置Kafka配置实例Broker属性。`RdKafka::Conf::ConfResult RdKafka::Conf::set(const std::string &name, const std::string &value, std::string &errstr) `

4. 设置Topic配置实例属性 。`RdKafka::Conf::ConfResult RdKafka::Conf::set (const std::string &name, const std::string &value, std::string &errstr) `

5. 注册回调函数。

6. 创建Kafka Consumer客户端实例。`static RdKafka::KafkaConsumer* RdKafka::KafkaConsumer::create(RdKafka::Conf *conf, std::string  &errstr); `(conf为Kafka配置实例)

7. 创建Topic实例。`static RdKafka::Topic* RdKafka::Topic::create(RdKafka::Handle *base,  const std::string &topic_str, RdKafka::Conf *conf,  std::string &errstr);`(conf为Topic配置实例 )

8. 订阅主题。`RdKafka::ErrorCode RdKafka::KafkaConsumer::subscribe(const std::vectorstd::string &topics); `

9. 消费消息。`RdKafka::Message* RdKafka::KafkaConsumer::consume (int timeout_ms);`

10. 关闭消费者实例 。`RdKafka::ErrorCode RdKafka::KafkaConsumer::close(); `

11. 销毁释放RdKafka资源 。`int RdKafka::wait_destroyed(int timeout_ms);`

```c++
// 注册回调函数
Conf::ConfResult RdKafka::Conf::set ("event_cb", RdKafka::EventCb *dr_cb, std::string &errstr);
Conf::ConfResult RdKafka::Conf::set ("socket_cb", RdKafka::SocketCb *socket_cb, std::string &errstr);
Conf::ConfResult RdKafka::Conf::set ("open_cb", RdKafka::OpenCb *open_cb, std::string &errstr);
Conf::ConfResult RdKafka::Conf::set ("rebalance_cb", RdKafka::RebalanceCb *rebalance_cb, std::string &errstr);
Conf::ConfResult RdKafka::Conf::set ("offset_commit_cb", RdKafka::OffsetCommitCb *offset_commit_cb, std::string &errstr);
Conf::ConfResult RdKafka::Conf::set ("consume_cb", RdKafka::ConsumeCb *consume_cb, std::string &errstr);
```

完整代码：

```c++
// KafkaConsumer.h
#pragma once
#include "rdkafkacpp.h"
#include <iostream>
#include <stdio.h>
#include <string>
#include <vector>
class ConsumerEventCb : public RdKafka::EventCb {
public:
  void event_cb(RdKafka::Event &event) {
    switch (event.type()) {
    case RdKafka::Event::EVENT_ERROR:
      if (event.fatal()) {
        std::cerr << "FATAL ";
      }
      std::cerr << "ERROR (" << RdKafka::err2str(event.err())
                << "): " << event.str() << std::endl;
      break;
    case RdKafka::Event::EVENT_STATS:
      std::cerr << "\"STATS\": " << event.str() << std::endl;
      break;
    case RdKafka::Event::EVENT_LOG:
      fprintf(stderr, "LOG-%i-%s: %s\n", event.severity(), event.fac().c_str(),
              event.str().c_str());
      break;
    case RdKafka::Event::EVENT_THROTTLE:
      std::cerr << "THROTTLED: " << event.throttle_time() << "ms by "
                << event.broker_name() << " id " << (int)event.broker_id()
                << std::endl;
      break;
    default:
      std::cerr << "EVENT " << event.type() << " ("
                << RdKafka::err2str(event.err()) << "): " << event.str()
                << std::endl;
      break;
    }
  }
};
class ConsumerRebalanceCb : public RdKafka::RebalanceCb {
private:
  static void printTopicPartition(
      const std::vector<RdKafka::TopicPartition *> &partitions) {
    for (unsigned int i = 0; i < partitions.size(); i++)
      std::cerr << partitions[i]->topic() << "[" << partitions[i]->partition()
                << "], ";
    std::cerr << "\n";
  }

public:
  void rebalance_cb(RdKafka::KafkaConsumer *consumer, RdKafka::ErrorCode err,
                    std::vector<RdKafka::TopicPartition *> &partitions) {
    std::cerr << "RebalanceCb: " << RdKafka::err2str(err) << ": ";
    printTopicPartition(partitions);
    if (err == RdKafka::ERR__ASSIGN_PARTITIONS) {
      consumer->assign(partitions);
      partition_count = (int)partitions.size();
    } else {
      consumer->unassign();
      partition_count = 0;
    }
  }

private:
  int partition_count;
};
class KafkaConsumer {
public: /**
     * @brief KafkaConsumer
     * @param brokers
     * @param groupID
     * @param topics
     * @param partition
     */
  explicit KafkaConsumer(const std::string &brokers, const std::string &groupID,
                         const std::vector<std::string> &topics, int partition);
  void pullMessage();
  ~KafkaConsumer();

protected:
  std::string m_brokers;
  std::string m_groupID;
  std::vector<std::string> m_topicVector;
  int m_partition;
  RdKafka::Conf *m_config;
  RdKafka::Conf *m_topicConfig;
  RdKafka::KafkaConsumer *m_consumer;
  RdKafka::EventCb *m_event_cb;
  RdKafka::RebalanceCb *m_rebalance_cb;
};

// KafkaConsumer.cpp
#include "KafkaConsumer.h"
KafkaConsumer::KafkaConsumer(const std::string &brokers,
                             const std::string &groupID,
                             const std::vector<std::string> &topics,
                             int partition) {
  m_brokers = brokers;
  m_groupID = groupID;
  m_topicVector = topics;
  m_partition = partition;
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
  errorCode = m_config->set("enable.partition.eof", "false", errorStr);
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
  errorCode = m_config->set("max.partition.fetch.bytes", "1024000", errorStr);
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
}
void msg_consume(RdKafka::Message *msg, void *opaque) {
  switch (msg->err()) {
  case RdKafka::ERR__TIMED_OUT:
    std::cerr << "Consumer error: " << msg->errstr() << std::endl;
    break;
  case RdKafka::ERR_NO_ERROR:
    std::cout << " Message in " << msg->topic_name() << " [" << msg->partition()
              << "] at offset " << msg->offset() << "key: " << msg->key()
              << " payload: " << (char *)msg->payload() << std::endl;
    break;
  default:
    std::cerr << "Consumer error: " << msg->errstr() << std::endl;
    break;
  }
}
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
KafkaConsumer::~KafkaConsumer() {
  m_consumer->close();
  delete m_config;
  delete m_topicConfig;
  delete m_consumer;
  delete m_event_cb;
  delete m_rebalance_cb;
}
// main.cpp
#include "KafkaConsumer.h"
int main() {
  std::string brokers = "192.168.0.105:9092";
  std::vector<std::string> topics;
  topics.push_back("test");
  topics.push_back("test2");
  std::string group = "testGroup";
  KafkaConsumer consumer(brokers, group, topics,
                         RdKafka::Topic::OFFSET_BEGINNING);
  consumer.pullMessage();
  RdKafka::wait_destroyed(5000);
  return 0;
}
```

```cmake
cmake_minimum_required(VERSION 2.8)
 project(KafkaConsumer)
 set(CMAKE_CXX_STANDARD 11)
 set(CMAKE_CXX_COMPILER "g++")
 set(CMAKE_CXX_FLAGS "-std=c++11 ${CMAKE_CXX_FLAGS}")
 set(CMAKE_INCLUDE_CURRENT_DIR ON)
 # Kafka头文件路径
include_directories(/usr/local/include/librdkafka)
 # Kafka库路径
link_directories(/usr/local/lib)
aux_source_directory(. SOURCE)
add_executable(${PROJECT_NAME} ${SOURCE})
TARGET_LINK_LIBRARIES(${PROJECT_NAME} rdkafka++)
```



