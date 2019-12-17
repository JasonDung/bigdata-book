#HDFS-集群搭建
## 1. 准备3台节点
在模版机的基础上克隆以下三台节点：

1. hdp-1(namenode，datanode)
2. hdp-2(namenode，datanode)
3. hdp-3(datanode)

## 2.增加主机映射
为了方便直接使用主机名来连接主机，需要在每台节点的hosts文件中都增加以下映射关系：

```
[root@localhost ~]# vi /etc/hosts
192.168.35.31 hdp-1
192.168.35.32 hdp-2
192.168.35.33 hdp-3
```
## 3. 配置免密登录
* 第一步，在每台节点中，分别生成使用以下命令生成密钥

```
[root@localhost ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):  // 回车
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase):  //回车
Enter same passphrase again:  //回车
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:xm633db4YqTDC7sNULK6VQ4+XFoiZBw290dC3ar1Z6g root@localhost.localdomain
The key's randomart image is:
+---[RSA 2048]----+
|      + ..o...   |
|     o + . o. .  |
|      + . o ..   |
|     o . + .o    |
|      . S +o . . |
|       B X.   + o|
|      . O =. + = |
|       + o BE.= .|
|      .   +.+=.o.|
+----[SHA256]-----+

```
* 第二步，每台节点中分别使用以下命令将各自主机公钥发送给需要互信的目标节点

```
[root@localhost ~]# ssh-copy-id hdp-3
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'hdp-3 (192.168.35.33)' can't be established.
ECDSA key fingerprint is SHA256:sjzZHXIUoPTFVnh8p2/62OYxF6kgRLxjkQLLXydS6B8.
ECDSA key fingerprint is MD5:8f:91:38:a4:a7:35:6a:fd:0a:54:c0:2a:fb:3d:3e:ab.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@hdp-3's password:  //输入目标机器登录密码

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'hdp-3'"
and check to make sure that only the key(s) you wanted were added.
```

## 4.关闭防火墙

```
#停止firewall
[root@hdp-1 opt]# systemctl stop firewalld.service  
#禁止firewall开机启动
[root@hdp-1 opt]# systemctl disable firewalld.service 

```

## 5.搭建HDFS集群
### 5.1 安装JDK环境
* 第一步，解压jdk包

```
[root@localhost jdk]# tar -xvf jdk-8u141-linux-x64.tar.gz 
```
* 第二步，配置全局JDK环境变量

```
[root@localhost jdk1.8.0_141]# vi /etc/profile
JAVA_HOME=/opt/jdk/jdk1.8.0_141
export PATH=$PATH:$JAVA_HOME/bin
```
* 第三步，使JDK环境生效

```
[root@localhost jdk1.8.0_141]# source /etc/profile
[root@localhost jdk1.8.0_141]# java -version
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.141-b15, mixed mode)
```
### 4.2 安装zookeeper
* 第一步，解压下载的zookeeper安装包，并修改配置文件

```
[root@hdp-1 opt]# tar -xvf zookeeper-3.4.6.tar.gz 

//指定日志存放路径
[root@hdp-1 zookeeper-3.4.6]# vi bin/zkEnv.sh 
ZOO_LOG_DIR=/data/logs/zookeeper

// 增加核心配置
[root@hdp-1 conf]# vi zoo.cfg 
dataDir=/data/zookeeper
server.1=hdp-1:2888:3888
server.2=hdp-2:2888:3888
server.3=hdp-3:2888:3888

```
* 第二步，将配置好的zookeeper文件夹发送到各节点机器

```
[root@hdp-1 opt]# scp -r zookeeper-3.4.6 hdp-2:/opt/

//分别在各节点处建立一个文件
[root@hdp-1 ～]# echo 1 >> /data/zookeeper/myid
[root@hdp-2 ～]# echo 2 >> /data/zookeeper/myid
[root@hdp-3 ～]# echo 3 >> /data/zookeeper/myid

```
* 第三步，分别在每台节点上启动zk

```
[root@hdp-2 zookeeper-3.4.6]# sh bin/zkServer.sh start
JMX enabled by default
Using config: /opt/zookeeper-3.4.6/bin/../conf/zoo.cfg
Starting zookeeper ... 
STARTED
[root@hdp-2 zookeeper-3.4.6]# jps
7656 QuorumPeerMain
7708 Jps
```
### 4.3 安装hadoop
* 第一步，解压下载的hadoop安装包

```
[root@localhost opt]# tar -xvf hadoop-2.8.3.tar.gz 
[root@localhost opt]# cd hadoop-2.8.3
```
* 第二步，修改hadoop-env.sh

```
[root@hdp-1 hadoop-2.8.3]# vi etc/hadoop/hadoop-env.sh 
export JAVA_HOME=/opt/jdk/jdk1.8.0_141/
export HADOOP_LOG_DIR=/data/logs/hadoop
export HADOOP_NAMENODE_OPTS="-Xms1024m	-Xmx1024m"
export HADOOP_DATANODE_OPTS="-Xms1024m	-Xmx1024m"
```
* 第三步，增加hdfs-site.xml和core-site.xml配置

```
//配置hdfs-site.xml 
[root@hdp-1 hadoop-2.8.3]# vi etc/hadoop/hdfs-site.xml 
<configuration>
		<property>
			<name>dfs.nameservices</name>
			<value>mycluster</value>
		</property>
		<property>
			<name>dfs.ha.namenodes.mycluster</name>
			<value>hdp-1,hdp-2</value>
		</property>
		<property>
			<name>dfs.namenode.rpc-address.mycluster.hdp-1</name>
			<value>hdp-1:8020</value>
		</property>
		<property>
			<name>dfs.namenode.rpc-address.mycluster.hdp-2</name>
			<value>hdp-2:8020</value>
		</property>
		<property>
			<name>dfs.namenode.http-address.mycluster.hdp-1</name>
			<value>hdp-1:50070</value>
		</property>
		<property>
			<name>dfs.namenode.http-address.mycluster.hdp-2</name>
			<value>hdp-2:50070</value>
		</property>
		<property>
			<name>dfs.namenode.shared.edits.dir</name>
			<value>qjournal://hdp-1:8485;hdp-2:8485;hdp-3:8485/mycluster</value>
			<description>journalnode集群访问地址</description>
		</property>
		<property>
			<name>dfs.client.failover.proxy.provider.mycluster</name>
			<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
			<description>判定namenode是否活跃</description>
		</property>
		<property>
			<name>dfs.ha.fencing.methods</name>
			<value>sshfence</value>
			<description>配置ssh的方式杀掉死掉的进程</description>
		</property>
		<property>
			<name>dfs.ha.fencing.ssh.private-key-files</name>
			<value>/home/hadoop/.ssh/id_rsa</value>
			<description>当故障转移时，hadoop会通过免密登录，kill死掉得namenode节点进程</description>
		</property>
		<property>
			<name>dfs.replication</name>
			<value>3</value>
		</property>
		<property>
			<name>dfs.namenode.name.dir</name>
			<value>file:///data/hadoop/hdfs/nn</value>
		</property>
		<property>
			<name>dfs.datanode.data.dir</name>
			<value>file:///data/hadoop/hdfs/dn</value>
		</property>
		<property>
			<name>dfs.journalnode.edits.dir</name>
			<value>/data/hadoop/hdfs/jn</value>
			<description>配置journalnode数据文件夹位置</description>
		</property>
		<property>
			<name>dfs.qjournal.start-segment.timeout.ms</name>
			<value>60000</value>
			<description>指定journal集群之间的通信超时时间</description>
		</property>
		<property>
			<name>dfs.ha.automatic-failover.enabled</name>
			<value>true</value>
			<description>开启failover机制</description>
		</property>
</configuration>
	
//配置core-site.xml
[root@hdp-1 hadoop-2.8.3]# vi etc/hadoop/core-site.xml
<configuration>
<property>
			<name>fs.defaultFS</name>
			<value>hdfs://mycluster</value>
		</property>
		<property>
			<name>ha.zookeeper.session-timeout.ms</name>
			<value>30000</value>
			<description>zkfc超过5s，就会连不上zookeper集群而导致自动退出，故设置会话超时时间30s</description>
		</property>
		<property>
			<name>ha.zookeeper.quorum</name>
			<value>hdp-1:2181,hdp-2:2181,hdp-3:2181</value>
			<description>auto failover 基于zookeeper，故指定zookeeper集群访问地址</description>
		</property>
</configuration>
```

* 第四步，修改slave文件

```
[root@hdp-1 opt]# vi hadoop-2.8.3/etc/hadoop/slaves 
hdp-1
hdp-2
hdp-3
```

* 第四步，分别在各个节点配置hadoop环境变量

```
[root@hdp-1 opt]# vi /etc/profile
export HADOOP_HOME=/opt/hadoop-2.8.3
export HADOOP_PREFIX=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMM_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$JAVA_HOME/bin
export HADOOP_INSTALL=$HADOOP_HOME

#使环境变量生效
[root@hdp-1 opt]# source /etc/profile
```
* 第四步，将配置好的hadoop文件夹分别发送另外2台节点

```
[root@hdp-1 opt]# scp -r hadoop-2.8.3 hdp-2:/opt/
[root@hdp-1 opt]# scp -r hadoop-2.8.3 hdp-3:/opt/
```
*  第五步，分别在另外两台节点创建目录

```
[root@hdp-1 opt]# mkdir -p /data/logs/hadoop
[root@hdp-1 opt]# mkdir -p /data/hadoop/hdfs/nn
[root@hdp-1 opt]# mkdir -p /data/hadoop/hdfs/dn
[root@hdp-1 opt]# mkdir -p /data/hadoop/hdfs/jn
```
* 第六步，启动hadoop集群

```
1，分别在各个节点启动journode（负责同步namenode之间的信息）
[root@hdp-1 opt]# hadoop-daemon.sh start journalnode
2，初始化namenode
[root@hdp-1 opt]# hdfs namenode -format
3，初始化成功后，将对应的namenode目录发送给另外一台namenode节点
[root@hdp-1 opt]# scp -r /data/hadoop/hdfs/nn hdp-2:/data/hadoop/hdfs/
4，分别启动两台namenode节点
[root@hdp-1 opt]# hadoop-daemon.sh --script hdfs start namenode
[root@hdp-2 opt]# hadoop-daemon.sh --script hdfs start namenode
5，初始化hadoop-ha目录
[root@hdp-1 opt]# hdfs zkfc -formatZK
6，分别在两台namenode中启动zkfc
[root@hdp-1 opt]# hadoop-daemon.sh --script hdfs start zkfc
[root@hdp-2 opt]# hadoop-daemon.sh --script hdfs start zkfc
7，分别启动3台datanode节点
[root@hdp-1 opt]# hadoop-daemon.sh --script hdfs start datanode
[root@hdp-2 opt]# hadoop-daemon.sh --script hdfs start datanode
[root@hdp-3 opt]# hadoop-daemon.sh --script hdfs start datanode
```
* 第7步，验证是否搭建成功
页面地址：192.168.35.31:50070/dfshealth.html#tab-overview 

```
Overview 'hdp-1:8020' (active)
Namespace:	mycluster
Namenode ID:	hdp-1
Started:	Mon Dec 09 20:31:59 +0800 2019
Version:	2.8.3, rb3fe56402d908019d99af1f1f4fc65cb1d1436a2
Compiled:	Tue Dec 05 11:43:00 +0800 2017 by jdu from branch-2.8.3
Cluster ID:	CID-a6dae433-aa6f-4b7e-a9c0-81f5712c5bee
Block Pool ID:	BP-1320373970-192.168.35.31-1575894650218
```
页面地址：192.168.35.32:50070/dfshealth.html#tab-overview 

```
Overview 'hdp-2:8020' (standby)
Namespace:	mycluster
Namenode ID:	hdp-2
Started:	Mon Dec 09 20:32:10 +0800 2019
Version:	2.8.3, rb3fe56402d908019d99af1f1f4fc65cb1d1436a2
Compiled:	Tue Dec 05 11:43:00 +0800 2017 by jdu from branch-2.8.3
Cluster ID:	CID-a6dae433-aa6f-4b7e-a9c0-81f5712c5bee
Block Pool ID:	BP-1320373970-192.168.35.31-1575894650218
```
第8步，测试是否会进行主备切换

1，停掉hdp-1的namenode节点之后，看hdp-2是否会切换状态为active

```
[root@hdp-1 opt]# hadoop-daemon.sh --script hdfs stop namenode
```
2，通过查看页面发现迟迟没有状态切换，查看zkfc日志发现如下报错：
```
[root@hdp-2 opt]# tail -500f /data/logs/hadoop/hadoop-root-zkfc-hdp-2.log 
java.lang.RuntimeException: Unable to fence NameNode at hdp-1/192.168.35.31:8020
        at org.apache.hadoop.ha.ZKFailoverController.doFence(ZKFailoverController.java:537)
        at org.apache.hadoop.ha.ZKFailoverController.fenceOldActive(ZKFailoverController.java:509)
        at org.apache.hadoop.ha.ZKFailoverController.access$1100(ZKFailoverController.java:61)
        at org.apache.hadoop.ha.ZKFailoverController$ElectorCallbacks.fenceOldActive(ZKFailoverController.java:895)
        at org.apache.hadoop.ha.ActiveStandbyElector.fenceOldActive(ActiveStandbyElector.java:985)
        at org.apache.hadoop.ha.ActiveStandbyElector.becomeActive(ActiveStandbyElector.java:882)
        at org.apache.hadoop.ha.ActiveStandbyElector.processResult(ActiveStandbyElector.java:467)
        at org.apache.zookeeper.ClientCnxn$EventThread.processEvent(ClientCnxn.java:599)
        at org.apache.zookeeper.ClientCnxn$EventThread.run(ClientCnxn.java:498)
```

3，根据日志得知，由于在切换状态时未找到fuster程序，导致无法进行fence，故而报错，需在namenode节点中安装psmisc程序：

```
yum install psmisc
```

安装好之后，再次查看192.168.35.32:50070/dfshealth.html#tab-overview 页面，会发现状态已经切换过来了！






   