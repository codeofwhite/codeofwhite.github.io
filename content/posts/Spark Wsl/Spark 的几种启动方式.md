---

title: "Spark 的几种启动方式"
date: 2025-04-26
tags: ["WSL", "Spark", "Hadoop"]
draft: false

---
<!--more-->
我们之前搭建的真实分布式环境允许任务：
```bash
./bin/spark-submit --master yarn ~/pyspark-project/test/word_count.py
```

### 总结

| **目的**               | **推荐方式**                                |
|------------------------|--------------------------------------------|
| **本地开发 / 调试**      | 使用 VSCode 运行 Python 文件（推荐）        |
| **模拟真实运行**        | 使用 `spark-submit --master local[*]`        |
| **真正分布式提交任务**  | 使用 `spark-submit --master yarn`（需要配置 Hadoop） |

### 说明：
- **本地开发 / 调试**：在开发过程中，使用 VSCode 直接运行 Python 文件可以快速进行调试，适合小规模数据和测试。
- **模拟真实运行**：使用 `spark-submit --master local[*]` 可以在本地模拟分布式运行，适用于单机环境下的测试和调试，`[*]` 表示使用所有可用的 CPU 核心。
- **真正分布式提交任务**：当你准备在真实的分布式环境中运行任务时，可以使用 `spark-submit --master yarn` 将任务提交到 YARN 集群，需要事先配置 Hadoop 集群。