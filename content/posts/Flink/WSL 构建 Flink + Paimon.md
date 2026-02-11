---
title: "WSL 构建 Flink + Paimon 实践"
date: 2025-04-28
categories: ["bigdata"]  
tags: ["wsl", "flink", "paimon"]
draft: false
---

wsl Flink paimon 实践

<!--more-->
使用 Apache Paimon + Apache Flink + WSL（Windows Subsystem for Linux）可以高效实现大规模数据湖实时处理和分析。以下是这三者结合使用的 **最佳实践指南**，包括开发环境配置、作业运行、表操作与调试建议等方面。

---

## 一、准备WSL

### 1. 安装 WSL 和 Ubuntu

确保使用的是 **WSL2**，推荐安装 Ubuntu 20.04/22.04：

```bash
wsl --install -d Ubuntu
```

### 2. 安装 Java 和 Maven

```bash
sudo apt update
sudo apt install openjdk-17-jdk maven -y
```

### 3. 安装 Flink

```bash
wget https://downloads.apache.org/flink/flink-1.17.1/flink-1.17.1-bin-scala_2.12.tgz
tar -xvzf flink-1.17.1-bin-scala_2.12.tgz
```

### 4. 安装 Paimon

可直接使用预编译的 paimon-flink jar，也可源码编译。

* 推荐版本：Paimon 0.6+
* 下载二进制：

```bash
wget https://archive.apache.org/dist/paimon/paimon-0.6.1/apache-paimon-0.6.1-bin.tar.gz
```
* 解压后，将 `paimon-flink-*.jar` 放入 Flink 的 `lib/` 目录中。

---

## 二、Flink + Paimon 的集成要点

### 1. 配置 FLINK\_HOME

```bash
export FLINK_HOME=~/flink-1.17.1
export PATH=$FLINK_HOME/bin:$PATH
```

### 2. 启动 Flink 集群（本地模式）

```bash
start-cluster.sh
```

### 3. 启动 SQL 客户端（这里我推荐使用 SQL CLI）

```bash
sql-client.sh
```

### 复制 jar 包到你的 flink 里面（需要下载 hadoop）
```bash
cp /opt/module/hadoop-3.4.1/share/hadoop/common/hadoop-common-*.jar .
cp /opt/module/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-client-core-*.jar .
cp /opt/module/hadoop-3.4.1/share/hadoop/hdfs/hadoop-hdfs-*.jar .
cp /opt/module/hadoop-3.4.1/share/hadoop/common/hadoop-auth-*.jar .

```

### 4. 加载 Paimon Catalog

```sql
CREATE CATALOG paimon WITH (
  'type' = 'paimon',
  'warehouse' = 'file:///home/your_user/paimon-warehouse'
);

USE CATALOG paimon;
```

---

## 三、表定义与最佳实践

### 1. 创建表（示例：以流数据写入）

```sql
CREATE TABLE user_events (
  user_id BIGINT,
  event_type STRING,
  event_time TIMESTAMP(3),
  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) PARTITIONED BY (event_type)
WITH (
  'bucket' = '2',
  'file.format' = 'parquet',
  'sink.parallelism' = '2'
);
```

### 2. 数据写入示例

使用 DataStream 或 SQL 插入：

```sql
INSERT INTO user_events
SELECT user_id, event_type, event_time
FROM source_table;
```

---

## 四、实时查询（Streaming Query）

使用 `Flink SQL` 或 `Flink DataStream` 读取 Paimon 表：

```sql
-- 实时读取（ChangeLog Mode）
SELECT * FROM user_events /*+ OPTIONS('scan.mode'='incremental') */;
```

最佳实践参数说明：

* `scan.mode=incremental`：读取增量变化。
* `log.system=none`：无日志系统，可选 kafka/none。

---

