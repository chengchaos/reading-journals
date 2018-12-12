# Kafka 简单应用

包括一个命令行的的使用和 java 代码的使用


## 下载安装

需要先安装 zookeeper

```bash
$ bin/zkServer.sh start-foreground
$ cd /work/Kafka
$ /work/kafka/bin/kafka-server-start.sh /work/kafka/config/server.properties    
$ /work/kafka/bin/kafka-topics.sh --create \
    --zookeeper localhost:2181 \
    --replication-factor 1 \
    --partitions 3 --topic chaos-topic
$ /work/kafka/bin/kafka-topics.sh --list \
    --zookeeper localhost:2181

$ /work/kafka/bin/kafka-console-producer.sh \
    --broker-list localhost:9092 --topic chaos-topic 
```


一个消费者:

```bash

$ /work/kafka/bin/kafka-console-consumer.sh --topic chaos-topic --bootstrap-server localhost:9092s
```