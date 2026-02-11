---
title: "Spark Streaming 报错记录"
date: 2025-04-28
categories: ["bigdata"]  
tags: ["Spark", "Spark Streaming"]
draft: false
---

部署 Spark Streaming 实现流计算的常见错误。

<!--more-->

报错原因

```bash
java.lang.NoClassDefFoundError: org/apache/spark/kafka010/KafkaConfigUpdater
```

这里报错的原因是少了一些 jar 包，然后有时候就算你加了 jar 包，也有可能 spark 会找不到 jar 包在哪。对应的 jar 包直接在甲骨文的官网下就可以了。

我这里的处理方式是在 config 指明 jar 包在哪里。

```python
# 创建 SparkSession
spark = SparkSession.builder \
    .appName("KafkaSparkStreamingExample") \
    .config("spark.jars", "file:///opt/module/spark-3.5.1-bin-hadoop3/jars/spark-sql-kafka-0-10_2.12-3.5.1.jar,file:///opt/module/spark-3.5.1-bin-hadoop3/jars/kafka-clients-2.7.2.jar,file:///opt/module/spark-3.5.1-bin-hadoop3/jars/spark-token-provider-kafka-0-10_2.12-3.5.3.jar,file:///opt/module/spark-3.5.1-bin-hadoop3/jars/commons-pool2-2.12.0.jar") \
    .getOrCreate()
```

最后成功解决问题
