# Zookeeper集群搭建
简介：ZooKeeper是一个经典的分布式数据一致性解决方案，致力于为分布式应用提供一个高性能、高可用，且具有严格顺序访问控制能力的分布式协调服务。

节点准备：
1. bigdata-1
2. bigdata-2
3. bigdata-3
## 1.下载Zookeeper
```
[root@bigdata-1 opt]# wget https://mirrors.bfsu.edu.cn/apache/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
[root@bigdata-1 opt]# tar -zvf zookeeper-3.4.14.tar.gz
[root@bigdata-1 opt]# cd zookeeper-3.4.14
```

## 2.配置Zookeeper
* 修改zoo.cfg文件

```
[root@bigdata-1 zookeeper-3.4.14]# cd conf/
[root@bigdata-1 conf]# cp zoo_sample.cfg zoo.cfg
[root@bigdata-1 conf]# vi zoo.cfg
//指定zookeeper数据存储目录，没有该目录则新建
dataDir=/data/zookeeper

// 增加以下配置
server.1=bigdata-1:2888:3888
server.2=bigdata-2:2888:3888
server.3=bigdata-3:2888:3888
[root@bigdata-1 conf]# echo "1" >> /data/zookeeper/myid
```
* 将配置好的zookeeper发送到另外两台节点

```
[root@bigdata-1 opt]# for i in 2 3;do scp -r zookeeper-3.4.14 bigdata-$i:$PWD;done
 [root@bigdata-1 opt]#for i in 2 3;do ssh bigdata-$i "echo '$i >> /data/zookeeper/myid' ";done
```
## 3.验证Zookeeper
* 启动3台节点中的zk服务

```
 [root@bigdata-1 opt]#  for i in 1 2 3; do ssh bigdata-$i "/opt/zookeeper-3.4.14/bin/zkServer.sh start"; done
```
* 查看进程验证

```
[root@bigdata-1 opt]# jps
9616 JournalNode
9313 NameNode
24501 QuorumPeerMain  # zk服务
9446 DataNode
9815 DFSZKFailoverController
24647 Jps
```










