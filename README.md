gishadoop
=========

一、数据导入
1、HDFS
    Hadoop实现了一个分布式文件系统（Hadoop Distributed File System），简称HDFS。
HDFS有着高容错性的特点，并且设计用来部署在低廉的硬件上。而且它提供高传输率来访问应用程序的数据，适合那些有着超大数据集的应用程序。
（1）创建文件夹
hadoop dfs -mkdir /home/hadoop/input
（2）本地文件存储到HDFS
hadoop dfs -put F_10w.txt /home/hadoop/input/.
2、HIVE    
    HIVE是基于Hadoop的一个数据仓库工具，可以将结构化得数据文件映射为一张数据库表，并提供类型sql查询功能，可以将sql语句转换为MapReduce任务进行运行。
（1）创建表
CREATE TABLE IF NOT EXISTS F_10w(objectid INT, latitude DOUBLE,longitude DOUBLE,value1 DOUBLE) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TEXTFILE;
（2）导入数据
  1) 从本地加载数据：
LOAD DATA LOCAL INPATH '/home/hadoop/hive/input/mr-data/F_10w.txt' OVERWRITE INTO TABLE F_10w;
  2) 导出数据到本地
INSERT OVERWRITE LOCAL DIRECTORY '/home/hadoop/hive/output/mr-data' SELECT t.* FROM F_10w t;
  3) 从HDFS加载数据：
LOAD DATA INPATH '/home/hadoop/hdfs/input/mr-data/F_10w.txt' OVERWRITE INTO TABLE F_10w;
  4) 导出数据到HDFS
INSERT OVERWRITE LOCAL DIRECTORY '/home/hadoop/hdfs/output/mr-data' SELECT t.* FROM F_10w t;
（3）元数据存储
    Hive 中的元数据包括表的名字，表的列和分区及其属性，表的属性(是否为外部表等)，表的数据所在目录等。
  1) 默认本地创建Derby数据库存放元数据；
  2) 配置通过网络连接到一个数据库存放元数据，如Oracle或者MySQL嵌入式数据库；  
3、SQOOP
    Sqoop是一个用来将Hadoop和关系型数据库中的数据相互转移的工具，可以将一个关系型数据库中的数据导进到Hadoop的HDFS中，也可以将HDFS的数据导进到关系型数据库中。
sqoop import --connect jdbc:oracle://10.144.167.24:1521/sde --username sde --password sde --table F_10w --hive-import
二、使用hadoop做相交统计分析（gis-tools-for-hadoop）
    道路面数据(空间数据)和MR坐标点数据（关系数据），道路数据一般以空间数据类型存在需要将其转换成json（Features To JSON）。
（1）加载jar包
add jar    /home/hadoop/tools/gis-tools-for-hadoop/samples/esri-geometry-api.jar    /home/hadoop/tools/gis-tools-for-hadoop/samples/spatial-sdk-hadoop.jar; 
（2）创建临时函数
create temporary function ST_Point as 'com.esri.hadoop.hive.ST_Point'; 
create temporary function ST_Intersects as 'com.esri.hadoop.hive.ST_Intersects';
（3）创建外部表
CREATE EXTERNAL TABLE IF NOT EXISTS F_10w (objectid INT, latitude DOUBLE,longitude DOUBLE,value1 DOUBLE) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' LOCATION '/home/hadoop/hive/input/mr-data'; 

CREATE EXTERNAL TABLE IF NOT EXISTS ROAD_POLYGON (objectid INT,BoundaryShape binary) ROW FORMAT SERDE 'com.esri.hadoop.hive.serde.JsonSerde' STORED AS INPUTFORMAT 'com.esri.json.hadoop.EnclosedJsonInputFormat' OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat' LOCATION '/home/hadoop/hive/input/road-data'; 
（4）查询测试
SELECT * FROM F_10w;
（5）相交统计
select ROAD_POLYGON.objectid,count(*) from f_10w join ROAD_POLYGON where st_intersects(st_point(f_10w.longitude,f_10w.latitude),ROAD_POLYGON.shape)=true group by ROAD_POLYGON.objectid;
三、hadoop结合Geometry API进行开发
    从hadoop的hadfs文件系统中获取数据，然后使用这些Geometry API将数据转化为Esri的几何对象，或者要素den个，当有了这些空间数据之后，就可以进行空间分析等。
（1）安装Eclipse hadoop插件
    其实在Eclipse下安装插件对Java开发者来说司空见惯，很平常，插件有助于我们快速开发，为了方便hadoop的开发，我们也可以安装一个插件，去网上下载：hadoop-eclipse-plugin-1.0.4.jar这个jar包，然后扔到eclipse的plugins目录下，重启就可以看到了。
（2）配置hadoop
 在window的Preference下找到Hadoop Map/Reduce,选择Hadoop 的安装目录，如下图：
 
 
打开Map/Reduce视图，如下：
 
在Map/Reduce Locations中设置Hadoop的相关参数，如下：
 
在Project Explorer可以看到hdfs中的文件目录和数据，可以通过右键给hdfs上上传文件和下载文件等操作。
 
（3）开发
新建一个Map/Reduce 工程，如下 
引入Esri提供的Geometry API jar包，我这里直接引入源码，如下：
 
通过向导，新建Mapper/Reduc类，如下：
 

四、Hadoop Yarn 
    从 0.23.0 版本开始，Hadoop 的 MapReduce 框架完全重构，发生了根本的变化。新的 Hadoop MapReduce 框架命名为 MapReduceV2 或者叫 Yarn。
Hadoop1.0和2.0比较如下：
 
（1）Shark
Shark即Hive on Spark，本质上是通过Hive的HQL解析，把HQL翻译成Spark上的RDD操作，然后通过Hive的metadata获取数据库里的表信息，实际HDFS上的数据和文件，会由Shark获取并放到Spark上运算。(具体使用参考Hive)
（2）Spark
一个开源的基于内存的大数据迭代运算项目（目前支持scala、python、java编程）
Spark对于资源管理与作业调度可以使用Standalone、Apache Mesos及Hadoop YARN来实现
（3）构建Spark集成开发环境：
1) 安装Scala2.9.3
2) 安装Eclipse Scala IDE插件
下载Eclipse Scala IDE插件，将Eclipse Scala IDE插件中features和plugins两个目录下的所有文件拷贝到Eclipse解压后对应的目录中。
3) 使用scala/python/java语言开发Spark程序
根据向导创建Scala工程，导入相应开发语言依赖库。
