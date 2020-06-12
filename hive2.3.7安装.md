# hive2.3.7安装
## 1. 下载hive安装包
```
[root@bigdata-1 ~]# cd /opt/
[root@bigdata-1 opt]# wget https://mirror.bit.edu.cn/apache/hive/hive-2.3.7/apache-hive-2.3.7-bin.tar.gz
[root@bigdata-1 opt]# tar -zxf  apache-hive-2.3.7-bin.tar.gz
[root@bigdata-1 opt]# mv apache-hive-2.3.7-bin hive
```
## 2.配置hive
注意：这里hive元数据储存在mysql中的，请提前安装好mysql数据库

```
[root@bigdata-3 opt]# cd hive
// 新建hive-site.xml
[root@bigdata-3 hive]# vi conf/hive-site.xml 
<configuration>
<!-- 存放hive元数据在mysql中，以下配置连接mysql参数-->
<property>
<name>javax.jdo.option.ConnectionURL</name>
<value>jdbc:mysql://192.168.35.2:3306/hive?createDatabaseIfNotExist=true</value>
<description>JDBC connect string for a JDBC metastore</description>
</property>

<property>
<name>javax.jdo.option.ConnectionDriverName</name>
<value>com.mysql.jdbc.Driver</value>
<description>Driver class name for a JDBC metastore</description>
</property>

<property>
<name>javax.jdo.option.ConnectionUserName</name>
<value>root</value>
<description>username to use against metastore database</description>
</property>

<property>
<name>javax.jdo.option.ConnectionPassword</name>
<value>12345678</value>
<description>password to use against metastore database</description>
</property>
<!-- hive cli显示列名 -->
<property>
   <name>hive.cli.print.header</name>
   <value>true</value>
</property>
<!-- hive cli显示db -->
<property>
   <name>hive.cli.print.current.db</name>
   <value>true</value>
</property>
   <!-- 关闭元数据版本校验 -->
<property>
   <name>hive.metastore.schema.verification</name>
   <value>false</value>
</property>
</configuration>
```
## 3.检查hadoop中的配置
### 3.1 检查hdfs-site.xml
在hdfs-site.xml中检查是否有以下配置，没有则加入

```
<property>
  <name>dfs.webhdfs.enabled</name>
  <value>true</value>
</property>
```
### 3.2 检查core-site.xml
在core-site.xml中检查是否有以下配置，没有则加入!

注意：以下配置中xxx部分应为你当前节点的用户名。

```
<property>
  <name>hadoop.proxyuser.xxx.hosts</name>
  <value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.xxx.groups</name>
  <value>*</value>
</property>
```
设置完成后，记得重启hdfs集群！！！！
## 4.初始化hive元数据

```
[root@bigdata-3 hive]# bin/schematool -dbType mysql -initSchema
```
## 5.启动并验证
第一种方式,启动hive cli:

```
[root@bigdata-3 hive]# bin/hive
```
第二种方式：
* 先启动hiveserver2：

```
[root@bigdata-3 hive]# bin/hiveserver2 
```
* 打开另外一个会话窗口,使用beeline连接：

```
 [root@bigdata-3 hive]# bin/beeline -u jdbc:hive2://bigdata-3:10000/default -n root
 
Beeline version 2.3.7 by Apache Hive
0: jdbc:hive2://bigdata-3:10000/default>use default;
```

