---
title: "Flink 重启丢失 Catalog 等记录问题"
date: 2025-04-28
tags: ["wsl", "flink"]
draft: false
---

WSL 中 Flink 重启丢失 Catalog 等记录问题

<!--more-->
Flink SQL Client 每次启动时「catalog（目录）、table（表）」等对象消失，主要是因为 **默认的 catalog 和 database 配置是临时的，且没有持久化到外部存储或配置文件中**。以下是造成这种现象的原因及解决方案。

---

### 🌐 原因分析

1. **默认使用内存 Catalog（`default_catalog`）**

   * Flink SQL Client 默认启动时使用内存中的 `default_catalog.default_database`，所有通过 SQL 创建的表（如 `CREATE TABLE`）默认只存在于这个会话中。
   * 当你关闭 Flink SQL Client 后，内存中的 catalog/table 就会丢失。

2. **没有使用持久化 Catalog（如 Hive Catalog / JDBC Catalog / FileSystem Catalog）**

   * 只有当你配置并使用了一个外部持久化的 catalog（如 Hive Metastore、JDBC Catalog 或存储在本地/远程文件系统的 FileSystem Catalog）时，表结构才会保存在外部。

3. **未使用 `.sql` 脚本或 YAML 初始化配置**

   * 如果你没有使用初始化 SQL 文件或 YAML 配置文件，SQL Client 启动时不会自动加载你之前的表定义。

---

### ✅ 解决方案

#### 方法一：使用持久化 Catalog（推荐）

* 示例：使用 HiveCatalog

```sql
CREATE CATALOG my_hive WITH (
  'type' = 'hive',
  'default-database' = 'default',
  'hive-conf-dir' = '/path/to/hive/conf'
);
USE CATALOG my_hive;
```

👉 使用 HiveCatalog 后，所有表会注册进 Hive Metastore，下次重启依然存在。

---

#### 方法二：使用 FileSystem Catalog（轻量级方案）

```sql
CREATE CATALOG my_fs WITH (
  'type' = 'filesystem',
  'default-database' = 'mydb',
  'warehouse' = 'file:///path/to/catalog_dir'
);
```

* 创建的表会写入本地/远程目录（如 HDFS）。
* 重启 Flink SQL Client 并重新 `USE CATALOG my_fs` 后可再次访问。

---

#### 方法三：配置初始化脚本

可以在 Flink SQL Client 启动时使用初始化 SQL 脚本或 YAML 配置：

* **配置 `sql-client-defaults.yaml`：**

```yaml
catalogs:
  - name: my_hive
    type: hive
    hive-conf-dir: /path/to/hive/conf
```

* 放入 `${FLINK_HOME}/conf/sql-client-defaults.yaml` 中，每次启动 SQL Client 时自动加载。

---

### 📌 补充建议

* 对于生产环境或长期项目，**一定不要依赖 default\_catalog**。
* 推荐使用 HiveCatalog 或 FileSystemCatalog 等持久化方案。
* 对于实验环境，建议将建表语句保存成 SQL 脚本，每次启动后自动执行。

---
