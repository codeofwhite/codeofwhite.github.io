---
title: "wsl Flink 配置项"
date: 2025-04-28
categories: ["bigdata"]  
tags: ["wsl", "flink"]
draft: false
---

wsl Flink 实践

<!--more-->

flink-conf.yaml
```yaml
jobmanager.rpc.address: localhost
jobmanager.rpc.port: 6123
taskmanager.numberOfTaskSlots: 2
parallelism.default: 1
rest.port: 8081

# 内存配置（Flink 1.20+ 必需）
jobmanager.memory.process.size: 1024m
taskmanager.memory.process.size: 2048m

```