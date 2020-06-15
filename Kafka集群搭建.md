# Kafka集群搭建
简介：Kafka 是一种高吞吐量的分布式发布订阅消息系统。有以下特性：

1. 通过O(1)的磁盘数据结构提供消息的持久化，这种结构对于即使数以TB的消息存储也能够保持长时间的稳定性能
2. 高吞吐量：即使是非常普通的硬件Kafka也可以支持每秒数百万的消息。
3. 支持通过Kafka服务器和消费机集群来分区消息。
4. 支持Hadoop并行数据加载。

节点准备：
bigdata-1
bigdata-2
bigdata-3
## 1. 下载Kafka

```
[root@bigdata-1 opt]# wget https://archive.apache.org/dist/kafka/0.10.2.1/kafka_2.11-0.10.2.1.tgz
[root@bigdata-1 opt]# tar -zxf kafka_2.11-0.10.2.1.tgz
[root@bigdata-1 opt]# cd kafka_2.11-0.10.2.1
```

## 2. 配置Kafka

* 配置server.properties

```
[root@bigdata-1 kafka_2.11-0.10.2.1]# vi config/server.properties
# 唯一id，且各个节点依次递增
broker.id=0
# 是否允许删除topic，默认false
delete.topic.enable=true
# kafka socket服务监听，这里建议使用ip，用hostname方式时，nginx+kafka方式解析不了
listeners=PLAINTEXT://192.168.35.10:9092
# kafka日志存放目录，不建议放kafka根目录下
log.dirs=/data/logs/kafka
# kafka数据保留时长，默认7天
log.retention.hours=168
# zookeeper配置
zookeeper.connect=bigdata-1:2181,bigdata-2:2181,bigdata-3:2181
```
* 将配置好的kafka，发送至其它节点

```
[root@bigdata-1 opt]# for i in 1 2 3;do scp -r kafka_2.11-0.10.2.1 bigdata-$i:$PWD;done
//分别在另外两台节点的kafka的 server.properties中，修改其中两个属性：
broker.id 和 listeners，其它配置不变
```

## 3. 启动并验证

* 启动kafka服务

```
[root@bigdata-1 opt]# for i in 1 2 3; do ssh bigdata-$i "/opt/kafka_2.11-0.10.2.1/bin/kafka-server-start.sh -daemon /opt/kafka_2.11-0.10.2.1/config/server.properties"; done
```

* 验证kafka

```
# 创建topic
[root@bigdata-1 kafka_2.11-0.10.2.1]# bin/kafka-topics.sh --create --topic testKafka --replication-factor 3 --partitions 3 --zookeeper bigdata-1:2181,bigdata-2:2181,bigdata-3:2181
# 查看topic
[root@bigdata-1 kafka_2.11-0.10.2.1]# bin/kafka-topics.sh --zookeeper bigdata-1:2181,bigdata-2:2181,bigdata-3:2181 --list
# 删除topic
[root@bigdata-1 kafka_2.11-0.10.2.1]#bin/kafka-topics.sh --delete --zookeeper bigdata-1:2181,bigdata-2:2181,bigdata-3:2181 --topic testKafka
# 生产消息
[root@bigdata-1 kafka_2.11-0.10.2.1]# bin/kafka-console-producer.sh --broker-list bigdata-1:9092,bigdata-2:9092,bigdata-3:9092 --topic testKafka
# 消费消息
[root@bigdata-1 kafka_2.11-0.10.2.1]# bin/kafka-console-consumer.sh --zookeeper bigdata-1:2181,bigdata-2:2181,bigdata-3:2181 --topic testKafka
```



