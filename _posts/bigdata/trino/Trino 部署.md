---
title: Trino 部署
date: 2022/02/14 22:59:59
updated: 2022/02/14 22:59:59
permalink: /bigdata/trino/installation.html
categories:
- bigdata
tags:
- Trino
---

## 集群规划

| 主机名 | 服务        |
| ------ | ----------- |
| trino1 | coordinator |
| trino2 | worker      |
| trino3 | worker      |

<!--more-->

## 安装依赖

**每台服务器**安装 Java11 和 Python3，其中 Java 11 使用 Trino 官网推荐的 [Azul Zulu](https://www.azul.com/downloads/zulu-community/) 发行版，版本 [JDK 11.0.14.1](https://www.azul.com/downloads/?version=java-11-lts&os=centos&architecture=x86-64-bit&package=jdk)。

> 注意：Trino 需要的 Java 版本为 64-bit 版本的 Java 11，且版本需要在 11.0.11 以上。Java 8 或 Java 11 更早的版本都不被支持。更新的版本如 Java12、 Java13、Java17 或许可用，但没有经过测试，官方并不推荐使用。

### 安装 Java11

创建安装路径：

为了和 cdh 集群统一，我就安装在了 `/usr/java` 目录下

```bash
$ sudo mkdir -p /usr/java
```

下载安装包：

```bash
$ cd /usr/java/
$ sudo wget https://cdn.azul.com/zulu/bin/zulu11.54.25-ca-jdk11.0.14.1-linux_x64.tar.gz
```

解压安装：

```bash
$ sudo tar -xzvf zulu11.54.25-ca-jdk11.0.14.1-linux_x64.tar.gz
……
```

创建软连接：

```bash
$ sudo ln -s zulu11.54.25-ca-jdk11.0.14.1-linux_x64 jdk11.0.14.1
$ sudo ln -s zulu11.54.25-ca-jdk11.0.14.1-linux_x64 jdk
```

配置环境变量：

```bash
$ sudo vim /etc/profile.d/java.sh
# 输入以下内容：

# JAVA 环境变量
export JAVA_HOME=/usr/java/jdk
# PATH
export PATH=$JAVA_HOME/bin:$PATH:.
# CLASSPATH
export CLASSPATH=$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:.

# wq 保存退出

# 使环境变量生效
$ source /etc/profile
```

测试：

```bash
$ java -version

# 输出结果：
openjdk version "11.0.14.1" 2022-02-08 LTS
OpenJDK Runtime Environment Zulu11.54+25-CA (build 11.0.14.1+1-LTS)
OpenJDK 64-Bit Server VM Zulu11.54+25-CA (build 11.0.14.1+1-LTS, mixed mode)
```

### 安装 Python3

使用 yum 安装即可：

```bash
$ sudo yum install python3 -y
```

测试：

```bash
$ python3 --version

# 输出结果：
Python 3.6.8
```



## 下载安装

创建包安装路径

```bash
$ sudo mkdir -p /opt/package
$ cd /opt/package
```

下载 Trino

```bash
$ sudo wget https://repo1.maven.org/maven2/io/trino/trino-server/371/trino-server-371.tar.gz
```

解压

```bash
$ sudo tar -xzvf trino-server-371.tar.gz
……
```

创建软连接

```bash
$ sudo ln -s trino-server-371 trino
```



## 基础配置

在**每台服务器**的 Trino 安装目录中创建一个 `etc` 目录，进入目录并进行以下配置：

```bash
$ sudo mkdir etc
$ cd etc
```

### node.properties

在 `etc` 目录下创建 `node.properties` 文件：

```bash
$ sudo vim node.properties
```

在文件内写入以下内容：

```properties
# 集群所有节点环境名称必须一样
node.environment=trino_cluster
# UUID，设置固定值在集群升级时可以保持和原来的一致
node.id=bd7a6033-c3d5-4086-b16a-d7bbaa2290e6
# 日志文件和数据文件储存目录，确认目录存在，不存在手动创建一下
node.data-dir=/data/trino
```

### jvm.config

在 `etc` 目录下创建 `jvm.config` 文件：

```bash
$ sudo vim jvm.config
```

在文件内写入以下内容：

```bash
-server
-Xmx7G
-XX:-UseBiasedLocking
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-XX:ReservedCodeCacheSize=512M
-XX:PerMethodRecompilationCutoff=10000
-XX:PerBytecodeRecompilationCutoff=10000
-Djdk.attach.allowAttachSelf=true
-Djdk.nio.maxCachedBufferSize=2000000
```

### config.properties

在 `etc` 目录下创建 `config.properties` 文件：

```bash
$ sudo vim config.properties
```

在机器 trino1 写入以下内容，使 trino1 作为 coordinator：

```properties
# 该节点是否作为coordinator
coordinator=true
# coordinator是否同时作为worker节点
node-scheduler.include-coordinator=true
# http连接端口
http-server.http.port=8080
# 所有节点查询可以使用的最大内存
query.max-memory=18GB
# 单个节点查询可以使用的最大用户内存
query.max-memory-per-node=6GB
# 作为 JVM 堆中的余量/缓冲区预留内存，Trino 没有追踪
memory.heap-headroom-per-node=1GB
# 服务发现的地址
discovery.uri=http://trino1:8080
```

在机器 trino2、trino3 写入以下内容，使 trino2、trino3 作为 worker：

```properties
# 该节点是否作为coordinator
coordinator=false
# http连接端口
http-server.http.port=8080
# 所有节点查询可以使用的最大内存
query.max-memory=18GB
# 单个节点查询可以使用的最大用户内存
query.max-memory-per-node=6GB
# 作为 JVM 堆中的余量/缓冲区预留内存，Trino 没有追踪
memory.heap-headroom-per-node=1GB
# 服务发现的地址
discovery.uri=http://trino1:8080
```

### log.properties

在 `etc` 目录下创建 `logging.properties` 文件：

```bash
$ sudo vim logging.properties
```

在文件内写入以下内容：

```properties
# 设置io.trino包的所有类(包括嵌套包的类, 如io.trino.spi.connector)的日志输出级别
io.trino=INFO
# 参数还可以设置为: DEBUG、ERROR
io.trino.plugin.mysql=WARN
io.trino.plugin.hive=WARN
```

全部配置完成后如图：

<img src="https://image-1257603108.cos.ap-guangzhou.myqcloud.com/20220221173940.png" style="zoom:50%;" />

## 启动 Trino

启动器脚本位于安装目录下的 `bin/launcher` 中。

启动顺序为先启动 coordinator，再启动 worker。

- 后台运行：

  ```bash
  $ bin/launcher start
  ```

- 前台运行：

  ```bash
  $ bin/launcher run
  ```

- 关闭节点：

  ```bash
  $ bin/launcher stop
  ```



## 查看Web UI

浏览器打开 http://trino1:8080

- 用户名：trino
- 密码：不用输入

点击登录，显示如下：

![](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/20220221181720.png)



## 客户端部署

Trino CLI 提供了一种基于终端的交互式 shell，用于运行查询。Trino CLI 是一个自执行的 JAR 文件，这意味着它像普通 UNIX 可执行文件一样使用。执行 Trino CLI 仅依赖 Java 环境。

下载 [trino-cli-371-executable.jar](https://repo1.maven.org/maven2/io/trino/trino-cli/371/trino-cli-371-executable.jar)

```bash
$ wget https://repo1.maven.org/maven2/io/trino/trino-cli/371/trino-cli-371-executable.jar
```

移动到 `usr/bin` 目录下，并改名为 trino

```bash
$ sudo mv trino-cli-371-executable.jar /usr/bin/trino
```

赋予其可执行权限

```bash
$ sudo chmod +x /usr/bin/trino
```

使用 Trino CLI 查询 Trino 集群

```bash
$ trino --server trino1:8080
```

![](https://image-1257603108.cos.ap-guangzhou.myqcloud.com/20220221182742.png)