1	搭建Hadoop集群
操作系统环境
我们有4台机器，其中三台为SUSE10：
$ lsb_release -a
LSB Version:    core-2.0-noarch:core-3.0-noarch:core-2.0-x86_64:...
Distributor ID: SUSE LINUX
Description:    SUSE Linux Enterprise Server 10 (x86_64)
Release:        10
Codename:       n/a
其中的另外一台为ubuntu
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 12.04.3 LTS
Release:        12.04
Codename:       precise
首先，我们需要下载一些安装包，这些安装包包括Hadoop, Hive, Sqoop以及所有这些包的依赖jdk1.6。下载之后，先保存在一个目录中。
创建用户
$ sudo addgroup hadoop
$ sudo adduser -ingroup hadoop hduser
如果是SUSE:
$ groupadd hadoop
$ useradd -g hadoop -m -s /bin/bash hduser
$ passwd hduser
由于目前较为稳定的是Cloudera打过补丁的版本，因此我们的所有包都从此地下载： hadoop-2.0.0-cdh4.6.0.tar.gz, hive-0.10.0-cdh4.6.0.tar.gz, sqoop-1.4.3-cdh4.6.0.tar.gz
先用刚才创建的用户hduser登陆系统，然后创建文件夹：
$ mkdir -p yarn
$ mv hadoop-2.0.0-cdh4.6.0.tar.gz yarn/
$ mv hive-0.10.0-cdh4.6.0.tar.gz yarn/

$ cd yarn

$ tar zxvf hadoop-2.0.0-cdh4.6.0.tar.gz
$ tar zxvf hive-0.10.0-cdh4.6.0.tar.gz

$ sudo chown -R hduser:hadoop hadoop-2.0.0-cdh4.6.0
$ sudo chown -R hduser:hadoop hive-0.10.0-cdh4.6.0
这样就得到了一个干净的环境。我们需要设置一些环境变量：
环境变量设置
HADOOP_VERSION=hadoop-2.0.0-cdh4.6.0
export HADOOP_HOME=$HOME/yarn/$HADOOP_VERSION
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

export HADOOP_MAPRED_HOME=$HOME/yarn/$HADOOP_VERSION
export HADOOP_COMMON_HOME=$HOME/yarn/$HADOOP_VERSION
export HADOOP_HDFS_HOME=$HOME/yarn/$HADOOP_VERSION
export HADOOP_YARN_HOME=$HOME/yarn/$HADOOP_VERSION
export HADOOP_CONF_DIR=$HOME/yarn/$HADOOP_VERSION/etc/hadoop

export JAVA_HOME=$HOME/jdk1.7.0_55
export PATH=$JAVA_HOME/bin:$PATH

export HIVE_HOME=$HOME/yarn/hive-0.10.0-cdh4.6.0
export PATH=$HIVE_HOME/bin:$PATH

export SQOOP_HOME=$HOME/yarn/sqoop-1.4.3-cdh4.6.0
export PATH=$SQOOP_HOME/bin:$PATH
设置之后source ~/.bashrc使之生效，然后在命令行中运行
$ hadoop
应该看到诸如：
Usage: hadoop [--config confdir] COMMAND
       where COMMAND is one of:
  fs                   run a generic filesystem user client
  version              print the version
  ...
的输出。
配置集群中的机器
首先，我们需要确定主机和从机，通常一个主机会管理若干个从机，我们的环境中有四个节点。这里我们统一起见，只是用其中的三台SUSE10的机器。
首先确保集群中的机器都有一个独立的，唯一的名字：
$ cat /etc/hosts
127.0.0.1       localhost

10.144.245.202  ubuntu
10.144.245.203  nassvr
10.144.245.204  suse
10.144.245.205  taurus
我们选择了nassvr作为主机,suse和taurus为从机。
机器名称可以通过
$ hostname nassvr
来修改
无密码登录
由于主从机需要通过网络通信，而且需要通过ssh通道，因此需要配置这些机器间可以无密码登录。在Linux环境中，这一点非常容易：
$ mkdir -p ~/.ssh
$ cd ~/.ssh
$ ssh-keygen -t rsa
如果你已经有之前的口令文件，需要备份一下。一般而言，服务器上不会有这样的口令，只需要默认的确认即可。这个命令会生成两个文件，一个id_rsa以及对应的公钥id_rsa.pub。
$ ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@suse
$ ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@taurse
使用上边的命令将公钥拷贝到远程的机器上。然后就可以无密码登录到远程的服务器了。之后在远程服务器上重复上述动作，使得在从机上也可以无密码的登录到主机上。
设置配置文件
修改$HADOOP_CONF_DIR目录下的这几个文件（core-site.xml, hdfs-site.xml, mapred-site.xml, yarn-site.xml），内容分别见其他的几个gist。
应该注意的是，在文件hdfs-site.xml中的dfs.replication需要和实际的从机的个数相等，比如此处为2，是因为我们有suse和taurus作为从机。
而文件core-site.xml中的hadoop.tmp.dir的值可以自定义，只需要保证该值所对应的目录实际存在即可。
启动各个节点
$ hadoop-daemon.sh start namenode
$ hadoop-daemons.sh start datanode
$ yarn-daemon.sh start resourcemanager
$ yarn-daemons.sh start nodemanager
$ mr-jobhistory-daemon.sh start historyserver
注意此处的启动datanode和nodemanager的时候，使用的是*-daemons.sh，而不是*-daemon.sh。这样就不会在主机上启动这两个进程了。
启动之后，可以通过jps来查看进行的运行情况：
查看所有节点的状态
hdfs dfsadmin -report
应该会看到诸如：
hduser@nassvr:~> hdfs dfsadmin -report
14/05/15 01:52:35 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Configured Capacity: 171787935744 (159.99 GB)
Present Capacity: 140130291712 (130.51 GB)
DFS Remaining: 140031705088 (130.41 GB)
DFS Used: 98586624 (94.02 MB)
DFS Used%: 0.07%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Datanodes available: 2 (2 total, 0 dead)

Live datanodes:
Name: 10.144.245.205:50010 (taurus)
Hostname: taurus
Decommission Status : Normal
Configured Capacity: 85893967872 (79.99 GB)
DFS Used: 49293312 (47.01 MB)
Non DFS Used: 14761175040 (13.75 GB)
DFS Remaining: 71083499520 (66.20 GB)
DFS Used%: 0.06%
DFS Remaining%: 82.76%
Last contact: Thu May 15 01:52:36 CST 2014


Name: 10.144.245.204:50010 (suse)
Hostname: suse
Decommission Status : Normal
Configured Capacity: 85893967872 (79.99 GB)
DFS Used: 49293312 (47.01 MB)
Non DFS Used: 16896468992 (15.74 GB)
DFS Remaining: 68948205568 (64.21 GB)
DFS Used%: 0.06%
DFS Remaining%: 80.27%
Last contact: Thu May 15 01:52:35 CST 2014
使用Hadoop来进行计算
使用sqoop来导入数据:
sqoop import --hive-import --connect jdbc:oracle:thin:@10.144.167.xx:1521:orcl --username SDE --password password --verbose --table F_100WS -m 1
注意数据库的用户名和表的名称要大写!
导入完成之后，可以在Hive中运行这样的命令来进行计算
add jar
  ${env:HOME}/gis/gis-tools-for-hadoop/samples/lib/esri-geometry-api.jar
  ${env:HOME}/gis/gis-tools-for-hadoop/samples/lib/spatial-sdk-hadoop.jar;
create temporary function ST_Point as 'com.esri.hadoop.hive.ST_Point';
create temporary function ST_Contains as 'com.esri.hadoop.hive.ST_Contains';
create temporary function ST_Intersects as 'com.esri.hadoop.hive.ST_Intersects';
CREATE EXTERNAL TABLE IF NOT EXISTS roads (Name string, Layer string, Shape binary)
ROW FORMAT SERDE 'com.esri.hadoop.hive.serde.JsonSerde'
STORED AS INPUTFORMAT 'com.esri.json.hadoop.EnclosedJsonInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION '${env:HOME}/sampledata'; 
upload the roadtest.json(Esri json fromat) to path /home/hduser/roadtest.json (in HDFS)
hadoop dfs -copyFromLocal roadtest.json /home/hduser/roadtest.json
select intersects node count then.
select count(*) from f_100w
join roads
where st_intersects(st_point(f_100w.longitude, f_100w.latitude), roads.shape)=true;

core-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
 
  <property>
     <name>fs.default.name</name>
     <value>hdfs://ubuntu:9000</value>
  </property>
 
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/home/hduser/yarn/hadoop-2.0.0-cdh4.6.0/tmp</value>
  </property>
</configuration>

hdfs-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
 
<configuration>
 
 <property>
   <name>dfs.replication</name>
   <value>2</value>
 </property>
 
 <property>
   <name>dfs.permissions</name>
   <value>false</value>
 </property>
</configuration>

mapred-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
 
<configuration>
 
   <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
   </property>
</configuration>
yarn-site.xml
<?xml version="1.0"?>
<configuration>
 
  <property>
     <name>yarn.nodemanager.aux-services</name>
     <value>mapreduce_shuffle</value>
  </property>
  
  <property>
     <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
     <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  
  <property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>ubuntu:8025</value>
  </property>
  
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>ubuntu:8030</value>
  </property>
  
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>ubuntu:8040</value>
  </property>
  
</configuration>


Spark安装配置
假设
1.	已经配置好了Hadoop+Yarn的环境（hadoop-2.3.0-cdh5.0.1）
2.	已经配置好了Hive环境（hive-0.10.0-cdh4.6.0）
3.	下载spark-0.9.1-bin-hadoop2.tgz
解压缩并设置环境变量
export SPARK_HOME=$HOME/yarn/spark-0.9.1-bin-hadoop2
export PATH=$SPARK_HOME/bin:$PATH
$SPARK_HOME下的assembly/target/scala-2.10/spark-assembly_2.10-0.9.1-hadoop2.2.0.jar是最主要的包，这个包中包含了所有需要运行的文件。应该注意的是，这个包中包含了jackson的1.8.8版本，可能会与新版本的产生运行时冲突。
配置$SPARK_HOME/conf/spark-env.sh:
export JAVA_HOME=/home/hduser/jdk1.7.0_55
export SCALA_HOME=/home/hduser/scala-2.9.3
export SPARK_WORKER_MEMORY=4g
SPARK有多种运行模式，可以独立运行，可以独立+YARN模式运行。区别在于，如果是独立运行，Spark会以master-slave的模式运行，每个slave上都会运行一个Worker进程，而master上会有一个Master进程。而如果是YARN模式，则完全不用配置，Spark会把任务分发到已经运行的YARN集群中去。
独立模式
配置$SPARK/conf/slaves
# A Spark Worker will be started on each of the machines listed below.
ubuntu
taurus
suse
然后将刚才的配置同步到其他三台机器上：
#!/bin/sh

SLAVES=$HADOOP_CONF_DIR/slaves

for node in $(cat $SLAVES)
do
        echo "sync conf with " $node
        rsync -Pav $HIVE_HOME/ hduser@$node:$HIVE_HOME/
        rsync -Pav $SPARK_HOME/ hduser@$node:$SPARK_HOME/
        rsync -Pav $SHARK_HOME/ hduser@$node:$SHARK_HOME/
        echo "synced with " $node
done
同步完成之后，启动所有节点：
$ cd $SPARK_HOME
$ sbin/start-all.sh
应该会看到如下的输出：
starting org.apache.spark.deploy.master.Master, logging to /home/hduser/yarn/spark-0.9.1-bin-hadoop2/sbin/../logs/spark-hduser-org.apache.spark.deploy.master.Master-1-nassvr.out
ubuntu: starting org.apache.spark.deploy.worker.Worker, logging to /home/hduser/yarn/spark-0.9.1-bin-hadoop2/sbin/../logs/spark-hduser-org.apache.spark.deploy.worker.Worker-1-ubuntu.out
taurus: starting org.apache.spark.deploy.worker.Worker, logging to /home/hduser/yarn/spark-0.9.1-bin-hadoop2/sbin/../logs/spark-hduser-org.apache.spark.deploy.worker.Worker-1-taurus.out
suse: starting org.apache.spark.deploy.worker.Worker, logging to /home/hduser/yarn/spark-0.9.1-bin-hadoop2/sbin/../logs/spark-hduser-org.apache.spark.deploy.worker.Worker-1-suse.out
对应的，可以使用$SPARK/sbin/stop-all.sh来停止所有的worker进程。
启动之后，可以通过JPS来查看：
hduser@ubuntu:~$ jps
9362 Worker
8581 DataNode
9844 Jps
8814 NodeManager
或者
hduser@nassvr:~/yarn> jps
18080 Master
16960 ResourceManager
19346 SharkCliDriver
16592 NameNode
889 Jps
17244 JobHistoryServer
Shark安装及配置
1.	Hive环境已经就绪
2.	Spark环境已经就绪
3.	下载Scala语言环境
配置环境变量
export SCALA_HOME=$HOME/scala-2.9.3
export PATH=$SCALA_HOME/bin:$PATH

export SHARK_HOME=$HOME/yarn/shark-0.9.1-bin-hadoop2
hduser@ubuntu:~$ cat yarn/shark-0.9.1-bin-hadoop2/conf/shark-env.sh
#!/usr/bin/env bash

export SPARK_HOME=$HOME/yarn/spark-0.9.1-bin-hadoop2
export SPARK_MEM=2g

# (Required) Set the master program's memory
export SHARK_MASTER_MEM=1g

export SHARK_HOME=$HOME/yarn/shark-0.9.1-bin-hadoop2

# Only required if run shark with spark on yarn
export SHARK_EXEC_MODE=yarn
export SPARK_ASSEMBLY_JAR=$SPARK_HOME/assembly/target/scala-2.10/spark-assembly_2.10-0.9.1-hadoop2.2.0.jar
export SHARK_ASSEMBLY_JAR=$SHARK_HOME/target/scala-2.10/shark_2.10-0.9.1.jar

# Java options
# On EC2, change the local.dir to /mnt/tmp
SPARK_JAVA_OPTS=" -Dspark.local.dir=/tmp "
SPARK_JAVA_OPTS+="-Dspark.kryoserializer.buffer.mb=10 "
SPARK_JAVA_OPTS+="-verbose:gc -XX:-PrintGCDetails -XX:+PrintGCTimeStamps "
export SPARK_JAVA_OPTS

HADOOP_VERSION=hadoop-2.3.0-cdh5.0.1
export HADOOP_HOME=$HOME/yarn/$HADOOP_VERSION
export HIVE_HOME=$HOME/yarn/hive-0.10.0-cdh4.6.0
export HIVE_CONF_DIR=$HIVE_HOME/conf
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
export MASTER=yarn-client
#export MASTER=spark://nassvr:7077

source $SPARK_HOME/conf/spark-env.sh
需要注意的是此处的MASTER的值，如果这里的值为yarn-client,则不需要启动spark的服务，只需要hadoop集群上的节点启动着即可。 如果此处的值为spark://nassvr:7077,则需要启动spark的服务。
由于spark中自带的jackson版本与ArcGIS的版本不同步，一个简单的解决方法是：
1.	将spark-assembly_2.10-0.9.1-hadoop2.2.0.jar包中的jackson部分删除，并重新打包成jar
2.	在ClassPath中添加jackson的新包
下载新版本的jackson包，并保存在$HOME/temp/下，然后拷贝到hive环境：
cp temp/jackson-*.jar yarn/hive-0.10.0-cdh4.6.0/lib/
重启所有服务，使之生效。
cp temp/jackson-*.jar yarn/hadoop-2.3.0-cdh5.0.1/share/hadoop/hdfs/lib/
cp temp/jackson-*.jar yarn/hadoop-2.3.0-cdh5.0.1/share/hadoop/yarn/lib/
cp temp/jackson-*.jar yarn/hadoop-2.3.0-cdh5.0.1/share/hadoop/tools/lib/
cp temp/jackson-*.jar yarn/hadoop-2.3.0-cdh5.0.1/share/hadoop/common/lib/
启动shark
要使用shark来做一些查询：
add jar
  ${env:HOME}/gis/gis-tools-for-hadoop/samples/lib/esri-geometry-api.jar
  ${env:HOME}/gis/gis-tools-for-hadoop/samples/lib/spatial-sdk-hadoop.jar;

create temporary function ST_Point as 'com.esri.hadoop.hive.ST_Point';
create temporary function ST_Contains as 'com.esri.hadoop.hive.ST_Contains';
create temporary function ST_Intersects as 'com.esri.hadoop.hive.ST_Intersects';


CREATE EXTERNAL TABLE IF NOT EXISTS roads (Name string, Layer string, Shape binary)
ROW FORMAT SERDE 'com.esri.hadoop.hive.serde.JsonSerde'
STORED AS INPUTFORMAT 'com.esri.json.hadoop.EnclosedJsonInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION '${env:HOME}/sampledata/roads'; 


select roads.name, count(*) from f_600w
join roads
where st_intersects(st_point(f_600w.longitude, f_600w.latitude), roads.shape)=true
group by roads.name;
110辅线 25131
X032    1184
丰善西路        142
于辛庄路        836
京包线  6244
北清路  14321
友谊路  1663
安济桥路        181
定泗路  1276
展思门路        255
工商南街        287
市场东路        5109
朱辛庄路        1074
李庄路  534
柴禾市大街     181
沙阳路  3000
满白路  71
王于路  90
环城北路        273
百沙路  1573
站前路  134
站西路  849
豆各庄路        281
路小路  125



一些应该注意的点
如何在Hive中创建表
CREATE EXTERNAL TABLE IF NOT EXISTS roads (Name string, Layer string, Shape binary)
ROW FORMAT SERDE 'com.esri.hadoop.hive.serde.JsonSerde'
STORED AS INPUTFORMAT 'com.esri.json.hadoop.EnclosedJsonInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION '${env:HOME}/sampledata/roads'; 
这里的com.esri.hadoop.hive.serde.JsonSerde定义在ArcGis提供的包中，因此需要先将对应的jar包导入到Hive环境中：
add jar /home/hduser/gis/gis-tools-for-hadoop/samples/lib/spatial-sdk-hadoop.jar /home/hduser/gis/gis-tools-for-hadoop/samples/lib/esri-geometry-api.jar;
注意要以分号结束，这样这个jar包就添加成功了。
另一方面，LOCATION中指定了一个目录，/home/hduser/sampledata/roads，注意此处的路径指定的是HDFS的路径，因此需要手工的上传该文件到HDFS：
$ hadoop fs -mkdir -p /home/hduser/sampledata/roads
$ hadoop fs -put roads.json /home/hduser/sampledata/roads/roads.json
这样，Hive会动态的获取此处的文件，并解析成为实际的数据。
参考资料
1.	How to install
2.	安装过程问题集合1
3.	安装过程问题结合2
4.	安装过程问题集合3