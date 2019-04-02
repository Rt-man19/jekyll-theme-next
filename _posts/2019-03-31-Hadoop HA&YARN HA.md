---

title: Hadoop HA & YARN HA

categories:

- 大数据

tags:

---

# Hadoop HA

## Hadoop HA 架构图

![Hadoop HA架构图](https://s2.ax1x.com/2019/03/30/ADkuuD.png)

## 角色职责

### 命名空间

---

在说架构图之前，先了解一下什么是**命名空间**

在通常情况下，读写请求有这几种方式

- hdfs dfs -ls /
- hdfs dfs -ls hdfs://ip:port/

第一种情况是HDFS文件系统在本机上面，直接执行就可以对HDFS进行操作

第二种情况是对分布式的HDFS进行操作的

但是，如果hdfs://ip:port 的ip写具体的某一台NN的地址，当这个NN挂掉了，到时候就要把所有脚本、代码里的这个ip换成别的Active状态的NN

命名空间就是用来解决这一问题,他是**由NameNode节点来管理**。

当有了命名空间时，对HDFS的操作就可以写作hdfs dfs -ls hdfs://namespace/。

在命名空间接收到客户端的请求时，它会去寻找core-site.xml 和 hdfs-site.xml这两个配置,这两个配置文件中会配置NN节点的地址，然后命名空间再将请求随机转发到NN节点，也就是说，命名空间可能会把请求转发到Standby的NN节点，然后再回到命名空间，再转发到Active的节点。

命名空间保证了在Active状态的NameNode出现故障时，StandBy切换到Active这一过程对用户是无感知的。也就是说，用户并不需要关心有几台NameNode节点，只要能访问HDFS就OK。

### Zookeeper

---

Zookeeper是单独的一个进程

Zookeeper在Hadoop集群中的主要作用是用来选举Active NameNode

对于同一个集群的Zookeeper有以下特点:

- Zookeeper是有一个领导者(leader)和多个追随者(follower)组成的集群

- Leader负责发起投票，更新集群状态

- Follower用于接收客户端的请求,并想客户端返回结果，在选举Leader的时候参与投票

- 全局数据一致性;每个Zookeeper服务器保存一份相同的副本

- 数据更新原子性: 一次更新要么成功，要么都失败

  

**注意: Zookeeper 的部署的数量一般都是奇数(2n+1)，但并不是部署的节点越多越好。当集群出现故障的时候，Active要切换到Standby是要选举的，节点越多，投票的时间越长，对外提供的服务越慢。**

一般生产环境中:

20台节点: 部署5台Zookeeper

20-100台节点: 7/9/11台

大于100台节点: 11台

并且，在大于100台节点的时候，部署Zookeeper节点的机器，只部署Zookeeper一个进程。这是因为几百台节点的时候Zookeeper任务很重，如果有其他进程会消耗机器的资源，可能会造成Zookeeper响应超时无法对外提供服务。

### ZKFC

---

ZKFC全称 zookeeperfailovercontrol

它是一个单独的进程,主要职责和特点:

- 负责监控Namenode节点的健康状态
- 向Zookeeper集群发送心跳消息，证明自己是可用的，使得自己可以被选举
- 当被选举为Active时，zkfc通过RPC协议调用使NameNode节点状态变为Active
- 在Standby切换到Active这一过程对用户是无感知的



### JN

---

JN全称journalnode

它是一个单独的进程,主要职责和特点:

- 负责Active Namenode和Standby Namenode之间同步数据
- 一般部署为 2n+1台
- 同一时刻只能有一台 Active Namenode节点写入, Standby Namenode节点读取
- 写入数据时，有n/2+1台节点写入时判定为写入成功
- 读取数据时，任意选择一台server读取即可

### Active Namenode

---

对外提供服务的NameNode节点

它的主要职责和特点:

- 将数据的操作记录写入自己的editlog，同时写入JN集群
- 接收DN的块报告和心跳



### Standby Namenode

---

备选的Namenode节点

它的主要职责和特点:

- 重演机制: 接收JN集群的日志，读取执行日志操作
- 使自己的元数据和Active Namenode节点保持一致
- 接收DN的块报告和心跳

### DataNode

---

真正存储数据的节点

主要是向 Active Namenode 和 Standby Namenode 发送块报告和心跳



# YARN HA

## YARN HA 架构图

![YARN 架构图](https://s2.ax1x.com/2019/03/30/ADAzmF.png)



## 角色职责

### Zookeeper

Zookeeper 在 YARN HA中和 Hadoop HA中功能一致

---

### RMStateStore

保存着RM作业信息,存储在zk的/rmstore目录下。有以下特点

- activeRM会向这个目录写APP信息
- 当activeRM挂了，另外一个standby RM通过
- ZKFC选举成功为active，会从/rmstore读取相应的作业信息。重新构建作业的内存信息，启动内部的服务，开始接收NM的心跳，构建集群的资源信息，并且接收客户端的作业提交请求

---

### ZKFC 

和Hadoop HA中唯一有区别的地方: 在 YARN HA 中, ZKFC 是RM的一个线程

---

### RM

ResourceManager

- 启动时候会向ZK的/rmstore目录写lock文件，写成功就为active，否则standby，rm节点zkfc会一直监控这个lock文件是否存在，假如不存在，就为active，否则为standby.
- 接收client的请求，接收和监控NM的资源状况的汇报，负载资源的分配和调度。
- 启动和监控APPMASTER on NM节点的container。

---

### NM

NameManager

节点资源的管理  启动容器运行task计算  上报资源 

---

