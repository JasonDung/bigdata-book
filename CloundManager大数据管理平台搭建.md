# ClouderaManager大数据管理平台搭建
## 1. 准备3台节点
在模版机的基础上克隆以下三台节点：

1. cdh1
2. cdh2
3. cdh3

## 2.增加主机映射
为了方便直接使用主机名来连接主机，需要在每台节点的hosts文件中都增加以下映射关系：

```
[root@localhost ~]# vi /etc/hosts
192.168.35.41 cdh1
192.168.35.42 cdh2
192.168.35.43 cdh3
#分别在各个节点中修改其对应的主机名
[root@localhost ~]# vi /etc/hostname
cdh1
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
[root@localhost ~]# ssh-copy-id cdh-2
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
[root@cdh-1 opt]# systemctl stop firewalld.service  
#禁止firewall开机启动
[root@cdh-1 opt]# systemctl disable firewalld.service 

```
## 5，搭建cdh集群
官网：https://www.cloudera.com/documentation/enterprise/6/6.2/topics/configure_cm_repo.html#cm_repo

建议离线安装，由于在线安装网速等因素影响，下载依赖包特别慢！
### 5.1，在本地yum源中，增加新配置的cdh源
* 从官网下载cdh源文件

```
[root@cdh1 ~]# cd /etc/yum.repos.d/
[root@cdh1 ~]# wget https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/cloudera-manager.repo
```
### 5.2，下载安装CDH所需依赖包

https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/

从上述地址中，手动下载以下包，并将以下包复制到安装cdh节点下的/var/cache/yum/x86_64/7/cloudera-manager/packages/ 目录下

1. cloudera-manager-agent-6.2.0-968826.el7.x86_64.rpm
2. cloudera-manager-daemons-6.2.0-968826.el7.x86_64.rpm
3. cloudera-manager-server-6.2.0-968826.el7.x86_64.rpm
4. oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm

注意：/var/cache/yum/x86_64/7/cloudera-manager/packages/目录是没有创建的，进行到5.3时，就有了！

### 5.3，yum安装cloudera manager server

```
[root@cdh1 yum.repos.d]# yum install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.ustc.edu.cn
 * extras: mirror.bit.edu.cn
 * updates: mirrors.ustc.edu.cn
.....
Install  3 Packages (+70 Dependent packages)
Upgrade             ( 12 Dependent packages)

Total download size: 1.1 G
Is this ok [y/d/N]: n  ##一定要选N，避免在线安装！！
Exiting on user command
Your transaction was saved, rerun it with:
 yum load-transaction /tmp/yum_save_tx.2019-12-19.17-09.rARb45.yumtx
```
注意：执行此步骤后，将第二步中的离线下载包发送到/var/cache/yum/x86_64/7/cloudera-manager/packages/ 目录下，避免网络在线安装，因为超级慢！

### 5.4，配置cdh所需的jdk环境

```
[root@cdh1 packages]# cd /usr/java/ 
[root@cdh1 java]# tar -xvf jdk-8u141-linux-x64.tar.gz 
[root@cdh1 java]# cp jdk1.8.0_141/ 1.8

#配置环境变量
[root@cdh1 java]# vi /etc/profile
JAVA_HOME=/usr/java/1.8
export PATH=$PATH:$JAVA_HOME/bin
#使jdk环境生效
[root@cdh1 java]# source /etc/profile
```
### 5.5，安装数据库(mysql)
* 第一步，安装java连接mysql驱动包

```
[root@cdh1 opt]# wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz
[root@cdh1 opt]# tar -xvf mysql-connector-java-5.1.46.tar.gz 
[root@cdh1 opt]# mkdir -p /usr/share/java/
[root@cdh1 opt]# cd mysql-connector-java-5.1.46
[root@cdh1 mysql-connector-java-5.1.46]# cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar
```
* 第二步，mysql安装

```
[root@cdh1 opt]# wget http://repo.mysql.com/mysql-community-release-el6-5.noarch.rpm
[root@cdh1 opt]# rpm -ivh mysql-community-release-el6-5.noarch.rpm 
[root@cdh1 opt]# yum -y install mysql-server mysql mysql-devel
[root@cdh1 opt]# mysqld_safe --skip-grant-tables & mysql -u root -p
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
+--------------------+
1 row in set (0.00 sec)
```
注意：如果上述没有mysql库，则进行下面步骤

```
#启动mysql服务
[root@cdh1 ~]# /etc/rc.d/init.d/mysqld start
Starting mysqld (via systemctl):       [  OK  ]
[root@cdh1 ~]# ls /etc/rc.d/init.d/mysqld -i
34316433 /etc/rc.d/init.d/mysqld
#设置成开机自启
[root@cdh1 ~]# chkconfig mysqld on
#修改mysqld文件的操作权限
[root@cdh1 ~]# chmod 755 /etc/rc.d/init.d/mysqld 
#重启mysqld服务
[root@cdh1 ~]# /etc/rc.d/init.d/mysqld restart
Restarting mysqld (via systemctl):                         [  OK  ]

```
* 第三步，更改数据库密码

```
[root@cdh1 ~]# mysql
mysql> use mysql;
# 修改root用户密码为123456
mysql> update user set password=PASSWORD("123456")where user="root"
#更新权限
mysql> flush privileges;
#授权远程登录
mysql> grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;
#更新权限
mysql> flush privileges;

#测试登录
[root@cdh1 ~]# mysql -u root -p
Enter password:  123456
```
### 5.6，预创建CDH中服务所需的数据库

* 第一步，在mysql中创建对应的库并授权

```
#scm服务对应的库
mysql> create database scm default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on scm.* to 'scm'@'%' identified by 'scm';
Query OK, 0 rows affected (0.00 sec)
#metastore服务对应的库
mysql> create database amon default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on amon.* to 'amon'@'%' identified by 'amon';
Query OK, 0 rows affected (0.00 sec)
#rman服务对应的库
mysql> create database rman default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on rman.* to 'rman'@'%' identified by 'rman';
Query OK, 0 rows affected (0.00 sec)
#hue服务对应的库
mysql> create database hue default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on hue.* to 'hue'@'%' identified by 'hue';
Query OK, 0 rows affected (0.00 sec)
#metastore服务对应的库
mysql> create database metastore default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on metastore.* to 'hive'@'%' identified by 'hive';
Query OK, 0 rows affected (0.00 sec)
#sentry服务对应的库
mysql> create database sentry default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on sentry.* to 'sentry'@'%' identified by 'sentry';
Query OK, 0 rows affected (0.00 sec)
#nav服务对应的库
mysql> create database nav default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on nav.* to 'nav'@'%' identified by 'nav';
Query OK, 0 rows affected (0.00 sec)
#navms服务对应的库
mysql> create database navms default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on navms.* to 'navms'@'%' identified by 'navms';
Query OK, 0 rows affected (0.00 sec)
#oozie服务对应的库
mysql> create database oozie default character set utf8 default collate utf8_general_ci;
Query OK, 1 row affected (0.00 sec)
mysql> grant all on oozie.* to 'oozie'@'%' identified by 'oozie';
Query OK, 0 rows affected (0.00 sec)
```
* 第二步，指定CDH数据落地库

```
[root@cdh1 opt]# /opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm root
Enter SCM password: 
JAVA_HOME=/usr/java/1.8
Verifying that we can write to /etc/cloudera-scm-server
Creating SCM configuration file in /etc/cloudera-scm-server
Executing:  /usr/java/1.8/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:/opt/cloudera/cm/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
[                          main] DbCommandExecutor              INFO  Successfully connected to database.
All done, your SCM database is configured correctly!
```
### 5.7，启动cloudera manager server
由于我用root用户安装的，cdh server不能用root用户启动，故创建一个cdh用户来启动

```
#创建用户
[root@cdh1 opt]# useradd cdh
#指定密码
[root@cdh1 opt]# passwd cdh
# 给cdh目录分配非root用户的读写权限
[root@cdh1 opt]# chmod -R 777 /var/log
[root@cdh1 opt]# chmod -R 777 /etc/cloudera-scm-server 
#切换用户
[root@cdh1 opt]# su cdh
#启动后台服务
[cdh@cdh1 opt]# nohup /opt/cloudera/cm/bin/cm-server &
#查看启动日志
[cdh@cdh1 opt]# tail -500f /var/log/cloudera-scm-server/cloudera-scm-server.log 
2019-12-19 19:35:30,826 INFO WebServerImpl:org.eclipse.jetty.server.AbstractConnector: Started ServerConnector@75d614ea{HTTP/1.1,[http/1.1]}{0.0.0.0:7180}
2019-12-19 19:35:30,830 INFO WebServerImpl:org.eclipse.jetty.server.Server: Started @79975ms
2019-12-19 19:35:30,830 INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.
```
* 访问页面地址：192.168.35.41:7180 
默认用户名：admin 密码：admin

注意： cloudera manager server 的配置目录：
/etc/cloudera-scm-agent，想更改端口等配置可以在config.ini中修改



