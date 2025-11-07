---
title: "WSL Kafka 安装教程"
date: 2025-04-28
categories: ["bigdata"]  
tags: ["Kafka", "WSL"]
draft: false
---

在 Windows WSL 上部署 Kafka

<!--more-->

## 安装 Kafka

```bash
cd /opt/module

wget https://archive.apache.org/dist/kafka/3.6.1/kafka_2.13-3.6.1.tgz

# 解压
tar -zxvf kafka_2.13-3.6.1.tgz

# 改名
mv kafka_2.13-3.6.1 kafka
```

## 快速部署

```bash
# 创建 topic
./bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1

# 检查 topic 是否创建成功
./bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# 启动生产者
./bin/kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092

# 在另一个终端启动消费者
./bin/kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server localhost:9092
```

## Python 上进行测试

```bash
pip install kafka-python

```

producer.py

```python
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers='localhost:9092')

for i in range(5):
    message = f"Message {i}"
    producer.send('test-topic', message.encode('utf-8'))
    print(f"Sent: {message}")

producer.flush()

```

consumer.py

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'test-topic',
    bootstrap_servers='localhost:9092',
    auto_offset_reset='earliest',
    group_id='my-group')

for message in consumer:
    print(f"Received: {message.value.decode('utf-8')}")

```
