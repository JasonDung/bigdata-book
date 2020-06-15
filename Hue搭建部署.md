# Hue搭建部署

简介：HUE=Hadoop User ExperienceHue是一个开源的Apache Hadoop UI系统，由Cloudera Desktop演化而来，最后Cloudera公司将其贡献给Apache基金会的Hadoop社区，它是基于Python Web框架Django实现的。通过使用Hue我们可以在浏览器端的Web控制台上与Hadoop集群进行交互来分析处理数据，例如操作HDFS上的数据，运行MapReduce Job，执行Hive的SQL语句，浏览HBase数据库等等。

## 1. Hue安装

* 准备环境

```
[root@bigdata-2 opt]# yum install -y ant asciidoc cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-plain gcc gcc-c++ krb5-devel libffi-devel libtidy libxml2-devel libxslt-devel make openldap-devel python-devel sqlite-devel openssl-devel gmp-devel mysql-devel python-devel
```

* 安装npm源环境

```
[root@bigdata-2 opt]# wget https://npm.taobao.org/mirrors/node/v10.14.1/node-v10.14.1-linux-x64.tar.gz
[root@bigdata-3 opt]# tar -zxf node-v10.14.1-linux-x64.tar.gz 
[root@bigdata-3 opt]# cd node-v10.14.1-linux-x64
[root@bigdata-3 node-v10.14.1-linux-x64]# vi /etc/profile
export NODE_HOME=/opt/node-v10.14.1-linux-x64
[root@bigdata-3 node-v10.14.1-linux-x64]# source /etc/profile
//配置taobao npm源
[root@bigdata-2 opt]# npm config set registry https://registry.npm.taobao.org
```

* 下载并编译

```
[root@bigdata-2 opt]# wget https://github.com/cloudera/hue/archive/cdh6.3.1-release.tar.gz
[root@bigdata-3 opt]# tar -zxf cdh6.3.2-release.tar.gz 
[root@bigdata-3 opt]# cd hue-cdh6.3.2-release/
[root@bigdata-3 hue-cdh6.3.2-release]# make apps
```

* 配置pseudo-distributed.ini文件

```
[root@bigdata-3 hue-cdh6.3.2-release]# vi desktop/conf/pseudo-distributed.ini
# hue web默认端口
http_port=8000
[[database]]
    # hue元数据存放的数据库名称
    name=hue
    engine=mysql
    host=bigdata-1
    port=3306
    user=root
    password=123456
 [[hdfs_clusters]]
    # HA support by using HttpFs

    [[[default]]]
       # Enter the filesystem uri
      fs_defaultfs=hdfs://mycluster:8020

      # NameNode logical name.
      logical_name=mycluster
      webhdfs_url=http://bigdata-1:14000/webhdfs/v1
      hadoop_conf_dir=/opt/hadoop-2.8.3/etc/hadoop/
      [[yarn_clusters]]

    [[[default]]]
      # Enter the host on which you are running the ResourceManager
      resourcemanager_host=bigdata-1
      # The port where the ResourceManager IPC listens on
      resourcemanager_port=8032
      resourcemanager_api_url=http://bigdata-1:8088
      proxy_api_url=http://bigdata-1:8088
 	history_server_api_url=http://bigdata-1:19888
    [beeswax]

  	# Host where HiveServer2 is running.
  	# If Kerberos security is enabled, use fully-qualified domain name 	(FQDN).
  	hive_server_host=bigdata-3
  	hive_server_port=10000
      hive_metastore_host=bigdata-3
      hive_metastore_port=9083
[liboozie]
	oozie_url=http://bigdata-3:11000/oozie
	remote_deployement_dir=/user/hue/oozie/deployments
[oozie]
 	oozie_jobs_count=100
	enable_cron_scheduling=true
	enable_document_action=true
	enable_oozie_backend_filtering=true
	enable_impala_action=false
```

* 初始化元数据

```
[hue@bigdata-3 hue-cdh6.3.2-release]$ build/env/bin/hue syncdb
[hue@bigdata-3 hue-cdh6.3.2-release]$ build/env/bin/hue migrate
```

* 启动&验证
由于hue不能用root用户启动，故创建hue用户

```
[root@bigdata-3 hue-cdh6.3.2-release]# useradd hue
[root@bigdata-3 hue-cdh6.3.2-release]# passwd hue
[root@bigdata-3 hue-cdh6.3.2-release]# vi /etc/sudoers

root    ALL=(ALL)       ALL
//添加hue用户权限
hue     ALL=(All)       ALL
[root@bigdata-3 hue-cdh6.3.2-release]# chown -R hue:hue ../hue-cdh6.3.2-release
[root@bigdata-3 hue-cdh6.3.2-release]# su hue
[root@bigdata-3 hue-cdh6.3.2-release]# build/env/bin/supervisor
```
页面访问：http://bigdata-3:8000/
