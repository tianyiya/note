## 一、场景

本次提交位移失败了，原因是消费者组已经开启了 Rebalance 过程，并且将要提交位移的分区分配给了另一个消费者实例。

出现这个情况的原因是，你的消费者实例连续两次调用 poll 方法的时间间隔超过了期望的 max.poll.interval.ms 参数值。

这通常表明，你的消费者实例花费了太长的时间进行消息处理，耽误了调用 poll 方法

### max.poll.interval.ms超时

- 增加单条消息的消费速度
- 增加max.poll.interval.ms
- max.poll.records 减少每次拉取的消息数
- 多线程加速消费

### 独立消费者

Kafka 独立消费者模式下，Kafka 集群并不会维护消费者的消费偏移量，需要每个消费者维护监听分区的消费偏移量，因此，独立消费者模式与 group 模式切勿混合使用！

group 模式的重平衡机制在消费者异常时可将其监听的分区重分配给其它正常的消费者，使得这些分区不会停止被监听消费，但是独立消费者由于是手动进行监听指定分区，因此独立消费者发生异常时，并不会将其监听的分区进行重分配，这就会造成某些分区消息堆积。因此，在该模式下，独立消费者需要实现高可用


```text
public static void main(String[] args) {
  Properties kafkaProperties = new Properties();
  kafkaProperties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
  kafkaProperties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.ByteArrayDeserializer");
  kafkaProperties.put("bootstrap.servers", "localhost:9092");
  KafkaConsumer<String, byte[]> consumer = new KafkaConsumer<>(kafkaProperties);
  List<TopicPartition> partitions = new ArrayList<>();
  partitions.add(new TopicPartition("test_topic", 0));
  partitions.add(new TopicPartition("test_topic", 1));
  consumer.assign(partitions);
  while (true) {
    ConsumerRecords<String, byte[]> records = consumer.poll(Duration.ofMillis(3000));
    for (ConsumerRecord<String, byte[]> record : records) {
      System.out.printf("topic:%s, partition:%s%n", record.topic(), record.partition());
    }
  }
}
```

如果你的应用中同时出现了设置相同 group.id 值的消费者组程序和独立消费者程序，那么当独立消费者程序手动提交位移时，Kafka 就会立即抛出 CommitFailedException 异常，
因为 Kafka 无法识别这个具有相同 group.id 的消费者实例，于是就向它返回一个错误，表明它不是消费者组内合法的成员