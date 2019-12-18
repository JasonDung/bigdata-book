# spark集群搭建
## 1，准备节点
hdp-1 (master,worker)

hdp-2(master,worker)

hdp-3(worker)
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
## 5，搭建spark集群
### 5.1，安装JDK环境
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
### 5.2 安装zookeeper
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
### 5.3，安装spark
* 第一步，从官网下载安装包后解压

```
[root@hdp-1 opt]# tar -xvf spark-2.4.4-bin-hadoop2.7.tgz
[root@hdp-1 opt]# cd spark-2.4.4-bin-hadoop2.7
```
* 第二步，修改配置

```
[root@hdp-1 spark-2.4.4-bin-hadoop2.7]# cp conf/spark-env.sh.template conf/spark-env.sh
[root@hdp-1 spark-2.4.4-bin-hadoop2.7]# vi conf/spark-env.sh
export JAVA_HOME=/opt/jdk/jdk1.8.0_141/
export SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=hdp-1:2181,hdp-2:2181,hdp-3:2181 -Dspark.deploy.zookeeper.dir=/spark"
# 从节点配置
[root@hdp-1 spark-2.4.4-bin-hadoop2.7]# cp conf/slaves.template conf/slaves
[root@hdp-1 spark-2.4.4-bin-hadoop2.7]# vi conf/slaves
hdp-1
hdp-2
hdp-3
```
* 第三步，将配置好的spark根目录，发送至另外两台节点

```
[root@hdp-1 opt]# scp -r spark-2.4.4-bin-hadoop2.7 hdp-2:/opt/
[root@hdp-1 opt]# scp -r spark-2.4.4-bin-hadoop2.7 hdp-3:/opt/
```
* 第四步,启动集群

```
[root@hdp-1 spark-2.4.4-bin-hadoop2.7]# sbin/start-all.sh 
#在另外一台master节点hdp-2处启动master服务
[root@hdp-2 spark-2.4.4-bin-hadoop2.7]# sbin/start-master.sh 
```
* 第五步，页面查看
http://192.168.35.31:8080  （alive）
http://192.168.35.32:8080   （standby）
* 验证主备切换

```
#停掉hdp-1的alive状态的master服务
[root@hdp-1 spark-2.4.4-bin-hadoop2.7]# sbin/stop-master.sh 
```
刷新hdp-2的standby状态的master节点，看是否切换为alive状态！


