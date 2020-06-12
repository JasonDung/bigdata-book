# hive安装TEZ执行引擎
简介：
Tez是Apache开源的支持DAG作业的计算框架，它直接源于MapReduce框架，核心思想是将Map和Reduce两个操作进一步拆分，即Map被拆分成Input、Processor、Sort、Merge和Output， Reduce被拆分成Input、Shuffle、Sort、Merge、Processor和Output等，这样，这些分解后的元操作可以任意灵活组合，产生新的操作，这些操作经过一些控制程序组装后，可形成一个大的DAG作业。

特点：Tez可以将多个有依赖的作业转换为一个作业（这样只需写一次HDFS，且中间节点较少），从而大大提升DAG作业的性能

## 1.下载tez

```
[root@bigdata-3 opt]# wget https://mirrors.bfsu.edu.cn/apache/tez/0.9.2/apache-tez-0.9.2-bin.tar.gz
[root@bigdata-3 opt]# tar -zxf apache-tez-0.9.2-bin.tar.gz
[root@bigdata-3 opt]# mv apache-tez-0.9.2-bin tez-0.9.2
```
## 2.hive配置tez
### 2.1 配置hive-env.sh

```
[root@bigdata-3 hive]# vi conf/hive-env.sh

export TEZ_HOME=/opt/tez-0.9.2/
export TEZ_JARS=""
for jar in `ls $TEZ_HOME | grep jar`;do
        export TEZ_JARS=$TEZ_JARS:$TEZ_HOME/$jar
done
for jar in `ls $TEZ_HOME/lib`;do
        export TEZ_JARS=$TEZ_JARS:$TEZ_HOME/lib/$jar
done
export HIVE_AUX_JARS_PATH=/opt/hadoop-2.8.3/share/hadoop/common/hadoop-lzo-0.4.21-SNAPSHOT.jar,$TEZ_JARS

```
### 2.1 配置hive-site.xml

```
[root@bigdata-3 hive]# vi conf/hive-site.xml
<property>
   <name>hive.execution.engine</name>
   <value>tez</value>
</property>

```
### 2.2 新增tez-site.xml

```
[root@bigdata-3 hive]# vi conf/tez-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<property>
    <name>tez.lib.uris</name>
    <value>${fs.defaultFS}/tez/tez-0.9.2,${fs.defaultFS}/tez/tez-0.9.2/lib</value>
</property>
<property>
    <name>tez.lib.uris.classpath</name>
    <value>${fs.defaultFS}/tez/tez-0.9.2,${fs.defaultFS}/tez/tez-0.9.2/lib</value>
</property>
<property>
     <name>tez.use.cluster.hadoop-libs</name>
     <value>true</value>
</property>
<property>
     <name>tez.history.logging.service.class</name>
     <value>org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService</value>
</property>
</configuration>
```
### 2.3 上传tez jar到hdfs

```
[root@bigdata-3 opt]# hdfs dfs -mkdir /tez
[root@bigdata-3 opt]# hdfs dfs -put /opt/tez-0.9.2/ /tez/
```
### 2.4 修改hdfs yarn配置

```
[root@bigdata-3 opt]# vi /opt/hadoop-2.8.3/etc/hadoop/yarn-site.xml 
<!-- 是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是true  -->
<property>
        <name>yarn.nodemanager.pmem-check-enabled</name>
        <value>false</value>
</property>
<!-- 是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是true。  -->
<property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
</property>
```
### 2.5 验证
当前窗口开启hiveserver2

```
[root@bigdata-3 hive]# bin/hiveserver2 
```
另外一个窗口开启beeline客户端

```
[root@bigdata-3 hive]# bin/beeline -u jdbc:hive2://bigdata-3:10000/default -n root

0: jdbc:hive2://bigdata-3:10000/default> use default;

```

页面访问hiveserver2 ： http://bigdata-3:10002/

在页面中查看执行的sql语句的执行引擎即可。




