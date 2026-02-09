---

title: "WSL 下搭建 Spark 调试环境指南"
date: 2025-04-26
categories: ["bigdata"]  
tags: ["WSL", "Spark", "Hadoop"]
draft: false

---

这是一篇记录在 WSL 环境下搭建 Spark 调试环境时遇到的问题及其解决方法的博客，方便日后排雷复用。

<!--more-->

## 💥 Bug 1：用户权限问题

启动 Hadoop 时出现错误：

```bash
ERROR: namenode can only be executed by root.
```

排查后发现，`start-dfs.sh` 脚本中对执行用户有限制（如下图），当前用户非 root 时会报错：

![alt text](/images/image.png)

**解决方法：**
- 使用 root 用户执行启动脚本，或
- 修改 `start-dfs.sh` 中的用户校验（仅建议在开发环境使用）

---

## 🚫 Bug 2：SSH 无密码登录失败

报错信息：

```bash
localhost: zhj20@localhost: Permission denied (publickey,password)
```

说明 SSH 无密码登录未配置，Hadoop 启动过程中的内部通信失败。

**解决步骤：**

1. **生成 SSH 密钥（若无）**
   ```bash
   ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
   ```

2. **配置公钥认证**
   ```bash
   cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

3. **验证是否能免密登录**
   ```bash
   ssh localhost
   ```

成功后应自动登录，无需输入密码，`exit` 退出即可。

---

## 🧭 Bug 3：YARN 节点列表为空

执行 `yarn node -list` 时无任何节点。

**排查方向：**
- 确认 ResourceManager 和 NodeManager 是否都已启动（`jps` 查看）
- 检查 `yarn-site.xml` 配置是否正确
- 查看日志文件是否有网络连接/端口异常

---

## 💣 Bug 4：NameNode 启动失败（EditLog 异常）

```bash
There appears to be a gap in the edit log. We expected txid 1, but got txid 113.
```

**原因分析：**
- Hadoop 恢复 fsimage + editlog 时发现 txid 序号不连续，editlog 可能损坏或缺失。

**解决方案（无重要数据时）：**

```bash
hdfs namenode -format
./sbin/start-dfs.sh
```

⚠️ 注意：此操作会清空 HDFS 中的所有数据，仅建议在调试或开发环境中使用！

---

## 🔥 Bug 5：DataNode 启动失败，无法写入副本

错误信息：

```bash
File ... could only be written to 0 of the 1 minReplication nodes. There are 0 datanode(s) running
```

排查配置：

```bash
cat $HADOOP_HOME/etc/hadoop/hdfs-site.xml | grep -A 5 dfs.datanode.data.dir
```

日志提示：

```bash
Incompatible clusterIDs
namenode clusterID = CID-018939f5...
datanode clusterID = CID-659a4b77...
```

说明 DataNode 保存的 clusterID 与 NameNode 不一致。

**解决办法：**

```bash
# 停止 Hadoop
./sbin/stop-dfs.sh

# 清除 DataNode 数据（不要删 NameNode 数据！）
rm -rf /opt/module/hadoop-3.4.1/hadoop_data/hdfs/datanode/*

# 重启
./sbin/start-dfs.sh

# 查看是否正常启动
jps
```

---

## 📌 问题总结

| 问题描述                         | 建议解决方式                                      |
|----------------------------------|---------------------------------------------------|
| `namenode` 报只能由 root 执行   | 使用 root 启动，或调整脚本权限校验               |
| SSH 登录失败                     | 配置本机免密登录（`ssh localhost`）              |
| YARN 无节点                      | 检查 NodeManager 是否运行，查看日志              |
| NameNode 启动失败（txid gap）   | 格式化 NameNode（开发环境可用）                  |
| DataNode 报 ClusterID 不一致     | 清空 datanode 数据目录，重新启动即可             |

---

## 🔗 参考链接

- [WSL 配置 Spark 的坑（ChatGPT 分享页 1）](https://chatgpt.com/share/680c4e86-85fc-800f-917c-db92b4a6e81c)  
- [调试 HDFS NameNode 启动失败（ChatGPT 分享页 2）](https://chatgpt.com/share/680c4ead-ee00-800f-ab1d-47c1235f3bf0)  
- [DataNode ClusterID 不一致排查记录（ChatGPT 分享页 3）](https://chatgpt.com/share/680c4ebc-5f4c-800f-a2ef-247f8d461413)