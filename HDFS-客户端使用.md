# HDFS-客户端使用

## 1，Linux终端操作HDFS
* 如何使用hdfs命令，通过-help来查看

```
[root@hdp-2 opt]# hdfs dfs -help
Usage: hadoop fs [generic options]
        [-appendToFile <localsrc> ... <dst>]
        [-cat [-ignoreCrc] <src> ...]
        [-checksum <src> ...]
        [-chgrp [-R] GROUP PATH...]
        [-chmod [-R] <MODE[,MODE]... | OCTALMODE> PATH...]
        [-chown [-R] [OWNER][:[GROUP]] PATH...]
        [-copyFromLocal [-f] [-p] [-l] [-d] <localsrc> ... <dst>]
        [-copyToLocal [-f] [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
        [-count [-q] [-h] [-v] [-t [<storage type>]] [-u] [-x] <path> ...]
        [-cp [-f] [-p | -p[topax]] [-d] <src> ... <dst>]
        [-createSnapshot <snapshotDir> [<snapshotName>]]
        [-deleteSnapshot <snapshotDir> <snapshotName>]
        [-df [-h] [<path> ...]]
        [-du [-s] [-h] [-x] <path> ...]
        [-expunge]
        [-find <path> ... <expression> ...]
        [-get [-f] [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
        [-getfacl [-R] <path>]
        [-getfattr [-R] {-n name | -d} [-e en] <path>]
        [-getmerge [-nl] [-skip-empty-file] <src> <localdst>]
        [-help [cmd ...]]
        [-ls [-C] [-d] [-h] [-q] [-R] [-t] [-S] [-r] [-u] [<path> ...]]
        [-mkdir [-p] <path> ...]
        [-moveFromLocal <localsrc> ... <dst>]
        [-moveToLocal <src> <localdst>]
        [-mv <src> ... <dst>]
        [-put [-f] [-p] [-l] [-d] <localsrc> ... <dst>]
        [-renameSnapshot <snapshotDir> <oldName> <newName>]
        [-rm [-f] [-r|-R] [-skipTrash] [-safely] <src> ...]
        [-rmdir [--ignore-fail-on-non-empty] <dir> ...]
        [-setfacl [-R] [{-b|-k} {-m|-x <acl_spec>} <path>]|[--set <acl_spec> <path>]]
        [-setfattr {-n name [-v value] | -x name} <path>]
        [-setrep [-R] [-w] <rep> <path> ...]
        [-stat [format] <path> ...]
        [-tail [-f] <file>]
        [-test -[defsz] <path>]
        [-text [-ignoreCrc] <src> ...]
        [-touchz <path> ...]
        [-truncate [-w] <length> <path> ...]
        [-usage [cmd ...]]

```
*  常用的hdfs命令

```
#查看hdfs根目录
[root@hdp-2 opt]# hdfs dfs -ls  /
#创建一个名为hdfsdemo的目录
[root@hdp-2 opt]# hdfs dfs -mkdir /hdfsdemo
[root@hdp-2 opt]# hdfs dfs -ls  /
Found 1 items
drwxr-xr-x   - root supergroup          0 2019-12-09 23:53 /hdfsdemo
#上传一个文件
[root@hdp-2 opt]# hdfs dfs -put /opt/test.txt /hdfsdemo/
[root@hdp-2 opt]# hdfs dfs -ls /hdfsdemo/
Found 1 items
-rw-r--r--   3 root supergroup          0 2019-12-09 23:54 /hdfsdemo/test.txt
#将hdfs文件下载到本地
[root@hdp-2 opt]# hdfs dfs -get /hdfsdemo/test.txt /opt/
[root@hdp-2 opt]# ll
total 1
-rw-r--r--.  1 root root    0 Dec  9 23:56 test.txt
#删除目录
[root@hdp-2 opt]# hdfs dfs -rm -r /hdfsdemo
Deleted /hdfsdemo
[root@hdp-2 opt]# hdfs dfs -ls /
```
## 2，JAVA操作HDFS
* 第一步，创建一个maven项目
* 第二步，引入操作HDFS所需的依赖

```
	<dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.8.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.8.3</version>
        </dependency>
```
* 第三步，java客户端连接HDFS集群

```
public class HdfsDemo {

    public static void main(String[] args) throws URISyntaxException, IOException {
    	 #指定操作HADOOP的用户名
        System.setProperty("HADOOP_USER_NAME", "root");
        #获取HDFS连接
        FileSystem fileSystem = FileSystem.get(new URI("hdfs://192.168.35.32:8020"), new Configuration());
        try {
            // 创建文件目录
            fileSystem.mkdirs(new Path("/hdfsdemo/"));
            // 创建文件
            fileSystem.createNewFile(new Path("/hdfsdemo/a.txt"));
            //将本地文件拷贝到hdfs
            fileSystem.copyFromLocalFile(new Path("/Users/b.txt"), new Path("/hdfsdemo/"));
            // 将hdfs文件拷贝到本地
            fileSystem.copyToLocalFile(new Path("/hdfsdemo/账号.txt"), new Path("/Users/"));

            // 遍历hdfs根目录
            RemoteIterator<LocatedFileStatus> fileList = fileSystem.listFiles(new Path("/"), true);
            while (fileList.hasNext()) {
                LocatedFileStatus file = fileList.next();
                System.out.println(file.getPath());
            }

            // 删除文件夹
            fileSystem.delete(new Path("/hdfsdemo/"), true);
        } finally {
            // 关闭链接
            fileSystem.close();
        }

    }
}
```
关于如何通过客户端操作HDFS，本篇到这就结束了！
