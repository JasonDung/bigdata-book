# Hive配置支持Lzo格式

## 1.下载Lzo
* 安装依赖
```
[root@bigdata-3 opt]# yum -y install gcc-c++ lzo-devel zlib-devel autoconf automake libtool
```
* 下载并编译
```
[root@bigdata-3 opt]# wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.10.tar.gz
[root@bigdata-3 opt]# tar -zxvf lzo-2.10.tar.gz
[root@bigdata-3 opt]# cd lzo-2.10
[root@bigdata-3 lzo-2.10]# ./configure
[root@bigdata-3 lzo-2.10]#make && make install
```

## 2. 编译hadoop-lzo源码

* 下载hadoop-lzo的源码，下载地址：https://github.com/twitter/hadoop-lzo/archive/master.zip
*  解压之后，修改pom.xml

```
[root@bigdata-3 opt]# unzip master.zip
[root@bigdata-3 opt]# cd hadoop-lzo-master
[root@bigdata-3 hadoop-lzo-master]# vi pom.xml
<hadoop.current.version>2.8.3</hadoop.current.version>
```

* 声明环境变量

```
[root@bigdata-3 opt]# vi /etc/profile
export C_INCLUDE_PATH=/usr/local/lzo-2.10/include
export LIBRARY_PATH=/usr/local/lzo-2.10/lib
[root@bigdata-3 opt]# source /etc/profile
```

* 开始编译hadoop-lzo

```
//有点耗时，建议本地编译
[root@bigdata-3 hadoop-lzo-master]# mvn package -Dmaven.test.skip=true
```

* 同步至hadoop lib中     
     
```
[root@bigdata-3 target]# scp -r hadoop-lzo-0.4.21-SNAPSHOT.jar  /opt/hadoop-2.8.3/share/hadoop/common/
```

## 3.Hadoop配置
* 修改core-site.xml

```
<configuration>
         <property>
             <name>io.compression.codecs</name>
             <value>
             org.apache.hadoop.io.compress.GzipCodec,
             org.apache.hadoop.io.compress.DefaultCodec,
             org.apache.hadoop.io.compress.BZip2Codec,
             org.apache.hadoop.io.compress.SnappyCodec,
             com.hadoop.compression.lzo.LzoCodec,
             com.hadoop.compression.lzo.LzopCodec
             </value>
         </property>
         <property>
             <name>io.compression.codec.lzo.class</name>
             <value>com.hadoop.compression.lzo.LzoCodec</value>
         </property>
     </configuration>
```

* 重启hadoop

```
[root@bigdata-3 opt]# start-dfs.sh
```

## 4.Hive验证
* 在hive中建表log_start, 指定表格式为lzo

```
CREATE EXTERNAL TABLE log_start (line string) PARTITIONED BY (dt string)
STORED AS
INPUTFORMAT "com.hadoop.mapred.DeprecatedLzoTextInputFormat"
OUTPUTFORMAT "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat"
LOCATION '/user/hive/warehouse/game/adzl';
```
* 已经存在表，可以修改表的存储格式

```
ALTER TABLE old_log_start
SET FILEFORMAT 
INPUTFORMAT "com.hadoop.mapred.DeprecatedLzoTextInputFormat"
OUTPUTFORMAT "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat";
```

* 当需要往lzo存储格式的表新增数据时，需要加入以下两个参数：

```
SET hive.exec.compress.output=true;
SET mapred.output.compression.codec=com.hadoop.compression.lzo.LzopCodec;
```
* 在hiveserver2 web页面中进行验证


