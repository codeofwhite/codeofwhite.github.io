---
title: "WSL 配置 Spark"
date: 2025-04-24
categories: ["study"]  
tags: ["WSL", "Spark"]
draft: false
---

这是一篇介绍使用 Windows WSL 配置 Spark 的教程，包含 Java、Hadoop、Spark 的安装与配置过程。

<!--more-->

**提示：别用 root 用户！**

# 一、安装 WSL

首先在 Windows 上启用 WSL（推荐使用 WSL2）：

```powershell
wsl --install
```

安装完成后，建议使用 **Ubuntu** 发行版，可在 Microsoft Store 搜索并安装。

---

# 二、配置 SSH（可选）

```bash
sudo apt install openssh-server

cd ~/.ssh/  # 若无该目录，可先执行 ssh localhost 初始化
ssh-keygen -t rsa   # 一路回车即可
cat id_rsa.pub >> authorized_keys
```

---

# 三、安装 Java

使用 OpenJDK 8（与 Spark 和 Hadoop 高度兼容）：

- 下载文件：`OpenJDK8U-jdk_x64_linux_hotspot_8u442b06.tar.gz`

```bash
mkdir -p ~/soft && cd ~/soft
tar -xvzf OpenJDK8U-jdk_x64_linux_hotspot_8u442b06.tar.gz
sudo mv jdk8u442-b06 /opt/java

# 设置环境变量
echo 'export JAVA_HOME=/opt/java' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# 验证安装
java -version
```

---

# 四、安装 Hadoop

- 下载文件：`hadoop-3.4.1-src.tar.gz`

```bash
cd ~/soft
tar -xvzf hadoop-3.4.1-src.tar.gz
sudo mv hadoop-3.4.1-src /opt/module/hadoop-3.4.1

# 设置环境变量
echo 'export HADOOP_HOME=/opt/module/hadoop-3.4.1' >> ~/.bashrc
echo 'export PATH=$HADOOP_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# 验证安装
hadoop version
```

## 修改配置文件

![alt text](/images/image2.png)

### `core-site.xml`

```xml
<configuration>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>file:/opt/module/hadoop-3.4.1/tmp</value>
    <description>Abase for other temporary directories.</description>
  </property>

  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>

<!--
 配置该属性，可以方便使用当前用户在 Hadoop UI界面进行 CRUD 操作
 <value>选项应替换为你的 username
-->

  <property>
    <name>hadooop.http.staticuser.user</name>
    <value>zhj20</value> # 这边换成你的用户名
  </property>
</configuration>
```

### `hdfs-site.xml`

```xml
<configuration>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///opt/module/hadoop-3.4.1/hadoop_data/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///opt/module/hadoop-3.4.1/hadoop_data/hdfs/datanode</value>
</property>

  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.namenode.checkpoint.dir</name>
    <value>/opt/module/hadoop-3.4.1/tmp/dfs/namesecondary</value>
</property>
<property>
    <name>dfs.namenode.rpc-address</name>
    <value>localhost:9000</value>
</property>


</configuration>
```

```xml
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>localhost</value> <!-- 或者 'localhost' -->
  </property>

  <!-- nodemanager 本地资源路径 -->
  <property>
    <name>yarn.nodemanager.local-dirs</name>
    <value>/opt/module/hadoop-3.4.1/hadoop_data/nm-local-dir</value>
  </property>
  <!-- nodemanager 运行时日志路径 -->
  <property>
    <name>yarn.nodemanager.log-dirs</name>
    <value>/opt/module/hadoop-3.4.1/hadoop_data/logs</value>
  </property>
    <!-- 指定 ResourceManager 地址 -->
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>codeofwhite:8032</value>
  </property>
    <!-- 启动 ResourceManager web UI 地址 -->
  <property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>codeofwhite:8088</value>
  </property>
<!-- Site specific YARN configuration properties -->
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.log.dir</name>
    <value>/opt/module/hadoop-3.4.1/logs</value>
  </property>

  <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>4096</value>
  </property>
</configuration>
```

## 格式化并启动 Hadoop

```bash
cd /opt/module/hadoop-3.2.3
./bin/hdfs namenode -format
./sbin/start-dfs.sh
```

访问地址：[http://localhost:9870](http://localhost:9870)

---

# 五、安装 Spark

- 下载文件：`spark-3.5.1-bin-hadoop3.tgz`

```bash
cd /opt/software
tar -zxvf spark-3.5.1-bin-hadoop3.tgz -C ../module
```

## 配置 Spark 环境

```bash
cd /opt/module/spark-3.5.1-bin-hadoop3
cp conf/spark-env.sh.template conf/spark-env.sh
vim conf/spark-env.sh
```

添加以下内容：

```bash
export SPARK_DIST_CLASSPATH=$(/opt/module/hadoop-3.4.1/bin/hadoop classpath)
```

---

# 六、运行 Spark 示例程序

```bash
cd /opt/module/spark-3.5.1-bin-hadoop3
./bin/run-example SparkPi
```

### 输出示例：

```text
Pi is roughly 3.1451557257786287
```

说明 Spark 运行成功。

---

# 七、Spark 常用界面访问地址

| 服务               | 端口 | 说明                  |
| ------------------ | ---- | --------------------- |
| Spark UI           | 4040 | 默认 shell 执行时激活 |
| Hadoop NameNode UI | 9870 | Hadoop 文件系统界面   |

---

# 八、参考链接

- [CSDN 博客参考](https://blog.csdn.net/Milv_xx/article/details/134365873)
