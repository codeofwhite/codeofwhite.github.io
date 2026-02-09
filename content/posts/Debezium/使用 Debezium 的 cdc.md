---
title: "基于 Debezium 的 CDC 数据同步实践"
date: 2025-05-05
categories: ["bigdata"]  
tags: ["Debezium", "Kafka", "CDC", "Docker", "数据同步"]
draft: false
summary: "一个最小化的 CDC 演示项目，展示如何使用 Debezium 实现 MySQL 到 Kafka 的数据变更捕获(CDC)"
featured: true
---

本文介绍一个最小可运行的 Change Data Capture (CDC) 演示项目，实现 MySQL 数据库变更到 Kafka 的实时同步。项目包含以下核心组件：

Docker Compose 配置：启动 MySQL、Kafka、Zookeeper、Debezium Connect 和 Kafka UI

数据库初始化脚本：创建测试表和初始数据

连接器注册脚本：配置 Debezium MySQL Connector

<!--more-->

# 项目结构
```cpp
cdc-demo/
├── docker-compose.yml # 容器编排配置
├── init.sql    # 数据库初始化脚本
└── register-connector.sh   # Debezium 连接器注册脚本
```

# 核心配置

## docker-compose.yml
```yaml
version: '3.7'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.3.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  mysql:
    image: mysql:8.0
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: testdb
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    command: [
      "--server-id=223344",
      "--log-bin=mysql-bin",
      "--binlog-format=ROW",
      "--binlog-row-image=FULL"
    ]

  connect:
    image: debezium/connect:2.4.1.Final
    ports:
      - "8083:8083"
    depends_on:
      - kafka
      - mysql
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: debezium_config
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_status
      KEY_CONVERTER_SCHEMAS_ENABLE: "false"
      VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_REST_ADVERTISED_HOST_NAME: connect
      CONNECT_PLUGIN_PATH: /kafka/connect

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092

```

## init.sql (MySQL 初始数据)
```sql
CREATE TABLE IF NOT EXISTS testdb.test_table (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  age INT
);

INSERT INTO testdb.test_table (name, age) VALUES ('Alice', 30), ('Bob', 25);

```

## register-connector.sh (注册 Debezium MySQL Connector)
```bash
#!/bin/bash

curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
  "name": "mysql-cdc-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.id": "223344",
    "topic.prefix": "mysql-server", 
    "database.server.name": "mysql-server",
    "database.include.list": "testdb",
    "table.include.list": "testdb.test_table",
    "include.schema.changes": "false",
    "snapshot.mode": "initial",

    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.testdb"
  }
}'


```

**这个配置会在 Kafka 中创建两个不同用途的 topic**，它们分别是：

---
## Topic 说明
### 1. `schema-changes.testdb`

这是 **Debezium 内部使用的 topic**，记录的是 **MySQL 表结构（schema）变化历史**。

* 由配置项控制：

  ```json
  "schema.history.internal.kafka.topic": "schema-changes.testdb"
  ```
* 存什么？

  * 表创建、字段变更、主键变化等结构性信息
  * Kafka Connect 用它来 **恢复任务状态、确保幂等性**
* 默认不会显示在业务消费中，**你一般不需要关心这个 topic 的内容**
* 🛠 通常 partition=1，replication=1 就够了

---

### 2. `mysql-server.testdb.test_table`

这是 **业务数据 topic**，即你真正要消费的 **CDC（变更数据捕获）数据流**。

* 构成方式：

  ```json
  "topic.prefix": "mysql-server"
  "database.include.list": "testdb"
  "table.include.list": "testdb.test_table"
  ```

  最终 topic = `prefix + . + database + . + table`
  → `mysql-server.testdb.test_table`

* 存什么？

  * `INSERT` / `UPDATE` / `DELETE` 操作对应的数据变更记录
  * 比如 `UPDATE age = 32`，会记录 `before` 和 `after` 值

* Kafka UI 中我们关注的就是这个 topic


# 使用步骤
```bash
# 1. 启动服务
docker-compose up -d

# 2. 等待服务启动（尤其是 MySQL 和 Kafka）

# 3. 注册 CDC connector
bash register-connector.sh

# 4. 浏览 Kafka UI 查看消息 (http://localhost:8080/ui)，查看 topic: mysql-server.testdb.test_table

```

测试增量数据同步
```bash
docker exec -it mysql mysql -uroot -proot

USE testdb;
INSERT INTO test_table (name, age) VALUES ('Charlie', 40);
UPDATE test_table SET age = 31 WHERE name = 'Alice';
DELETE FROM test_table WHERE name = 'Bob';

```

# 常见问题处理
## 问题：缺少 Kafka Topic
```bash
# 在容器里执行以下命令
docker exec -it <kafka-container-id> bash

# 创建 topic（名字要和配置中一致）
kafka-topics --bootstrap-server localhost:9092 --create \
  --topic schema-changes.testdb --replication-factor 1 --partitions 1
```
