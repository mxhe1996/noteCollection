---
typora-root-url: ./pic
---

# Hadoop
## 安装包解释
软件包下各个文件夹的含义
+ bin: hadoop客户端命令

+ etc/hadoop: hadoop相关的配置文件的存放

+ sbin:启动hadoop相关进程的命令

+ lib: 依赖包

安装后的配置
+  /etc/Hadoop-env : 配置Java_Home
+  /etc/Hadoop/core-site.xml: 配置默认文件系统，以及hadoop存放的临时位置
+  /etc/Hadoop/hdfs-site.xml:配置文件系统采用的副本数量

```xml
# core-site.xml
<configuration>
    <property>
	<name>fs.defaultFS</name>
	<value>hdfs://192.168.199.177:8020</value>
    </property>
    <property>
	<name>hadoop.tmp.dir</name>
	<value>/data/hadoop-3.3.3/tmp</value>
    </property>
</configuration>
```

```xml
# hdfs-site.xml
<configuration>
    <property>
        <name>dfs.http.address</name>
        <value>192.168.199.177:50070</value>
    </property>
    <property>
	<name>dfs.replication</name>
	<value>1</value>
    </property>
</configuration>
```

```xml
# mapred-site.xml
<configuration>
        <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
        </property>
</configuration>

```



如果是集群配置，3.x版本的hadoop需要在`/etc/hadoop/workers`中设置主从

```sh
node1
node2
node3
```

其中的node？在`/etc/hosts`配置

```sh
192.168.199.xx node1
192.168.199.yy node2 
```



配置完毕后在profile文件末尾配置

```sh
export JAVA_HOME=/app/jdk1.8.0_341
export PATH=$PATH:$JAVA_HOME/bin

export HADOOP_HOME=/app/hadoop-3.3.3
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```




## 启动HDFS
第一次启动集群的时候一定要格式化 hdfs namenode -format
启动集群：$HADOOP_HOME/sbin/start-dfs.sh

启动之后JPS能看到有进程，但是在网页上打不开 50070：(可能是防火墙的问题)
1. 关闭防火墙sudo systemctl stop firewalld.service

2. 查看防火墙状态： sudo firewall-cmd –state

3. 禁止防火墙开机启动：systemctl disable firewalld.service

### 单一进程启动
在hadoop中的sbin文件中执行start-dfs.sh后启动hadoop，通过jps能够出现：datanode，namenodes，secondarynamenodes(实际在hadoop2中基本用不到)共计3个进程
如果需要单独开启进程，在sbin文件夹中执行hadoop-daemon.sh start xxxx开启指定的进程即可

``` shell
bash start-dfs.sh
// 等价如下
bash hadoop-daemon.sh start datanode
bash hadoop-daemon.sh start namenode
bash hadoop-daemon.sh start secondarynamenode
```

## Hadoop命令行执行(文件系统)
`Hadoop fs -command (ls | mkdir | copyfromlocal | cat |……) /`
最后的“/”是指从根目录开始（上传文件、查看文件、查看文件目录。。。。。）
其中fs是指filesystem shell（文件系统shell） 提供一种shell-like的交互方式

### Hadoop的存储机制
Hadoop的存储位置依赖于/etc/Hadoop/core-site.xml的`<name>hadoop.tmp.dir<name>`中，并且以128M大小的block放置在该目录下的data文件夹/current/xxxxx/finalized/…中

**put/上传: 1file --》 1…n block --》 存放在不同的节点（datanode）上**
**get/获取: 查找file对应的元数据信息，按照block id进行拼接**

### Block的设置

block的大小的设置应该遵循最小的寻址开销。除了最后一个全都一致，而最后一个block如果小于`blocksize`，再hdfs中也会占用一个block块，但是不会占用整个块的空间。

**如果block大小设置过大**：

1. 从磁盘传输数据的时间会明显大于寻址时间
2. map任务通常一次只处理一个块的数据，处理速度过慢
3. 如果需要网络传输，可能增加网络传输的时间，出现卡顿/超时等现象
4. namenode容易误认为该节点异常，频繁移动副本，浪费系统资源

**如果block过小：**

1. 寻址时间增加，消耗大量的时间在block寻址上
2. 产生大量的block数量，需要在namenode中记录，加大namenode的内存占用
3. 存在性能问题
4. 频繁产生传输次数，浪费资源



## HDFS API编程 操作HDFS文件系统
### IDEA工具调试
HDFS的实现是通过FileSystem这个基类的get方法返回一个指定类。该get方法有3个重载实现，需要指定
1.	URI，即HDFS的具体地址，如:hdfs://hadoop000:8020
2.	Configuration类的引用变量，
3.	角色，简单理解为，得到hdfs文件对象后，执行后续操作的人员，不同的角色在该hdfs中的权限不同



#### 命令行与IDEA的区别(配置)

Configuration中的参数与**/etc/Hadoop/hdfs-site.xml**中的参数一致；如果使用fs命令行的方式创建文件，参数以**/etc/Hadoop/hdfs-site.xml**，如果是configuration中new出来的，则以代码为主。

例如，需要配置创建的副本数为1(默认dfs.replication为3)

``` java
Configuration conf = new Configuration();
conf.set("dfs.replication","1");
```



### HDFS写数据流程图解

![image-20221008213705414](/image-20221008213705414.png)

写数据时，需要先向NameNode发起请求，在得到肯定后，再次向NameNode询问block1应该存储的DataNode位置，然后对这些DataNode节点发起连接，随后传输；block2同样，先向NameNode询问block2应该存放的位置信息，然后建立连接和传输。最后将数据的元数据信息存储到NameNode中。



### HDFS读数据流程图解

![image-20221008213801321](/image-20221008213801321.png)



## Hadoop 元数据管理

元数据主要是包括HDFS的目录结构，以及每个文件block块的id、副本系数、。。。。

元数据存放的位置`${hadoop.tmp.dir}/name…`



### Checkpoint机制

![image-20221008213848250](/image-20221008213848250.png)

### Safemode

...



## MapReduce分布式处理框架

### 代码解释-InputFormat

所有的输入格式都继承于`InputFormat`，这是一个抽象类，其子类包括专门用于读取普通文件的`FileInputFormatter`，用于实现数据库读取的`DBInputFormat`等等。
`InputFormat`抽象类有2个抽象方法：

+	`public abstract List<InputSplit> getSplits(JobContext context)`
getSplits()方法是逻辑上拆分作业的输入文件集，然后将每个InputSplit分配给一个单独的Mapper执行
+	`public abstract RecordReader<K,V> createRecordReader (InputSplitsplit, TaskAttemptContext context) createRecordReader()`方法是为给定的切片创建一个记录阅读器。在切片被使用之前先调用`RecordReader.initialize(InputSplit, TaskAttemptContext)`方法。
通过InputFormat的子类实现，MapReduce框架可以做到：
1. 验证作业输入的正确性
2. 将输入的文件切割成逻辑分片(`InputSplit`),一个InputSplit将会分配给一个独立的MapTask
3. 提供`RecordReader`实现，读取`InputSplit`中的Kv对供Mapper使用。



### 代码解释-InputSplit

输入分片(Input split)：在map之前，MapReduce会根据输入文件计算输入分片(图中的splitting操作)，每个输入分配对应一个map任务，而且每个输入分片存储的并非数据本身，而是一个分片长度和记录数据位置的元组<input-file-path,start,offset>。
分片大小可以在mapred-site.xml中设置，mapred.min.split.size、mapred.max.split.size；
实际的操作中，会执行以下代码：

``` matlab
minSize=max{minSplitSize,mapred.min.split.size} 
splitSize=max{minSize,min{maxSplitSize,blockSize}}
```


其中minSplitSize默认为1B，maxSplitSize的值为Long.MAX_VALUE (9223372036854775807)
因此，在没有设置分片的范围的时候，分片的大小就是block的大小(hadoop2.x中为128MB)

每个分片(大小默认为128MB)可能与文件的行并不对其，虽然这并不影响程序的功能，但是本地的map操作可能会执行远程的读操作

![image-20221008214517576](/image-20221008214517576.png)



### 代码解释-RecordReader

RecordReader同样是个抽象类，负责将读入到Map的数据拆分成键值对。

``` java
public abstract void initialize(InputSplit split, TaskAttemptContext context)；
```



### Combiner

Combiner是通过`Reducer`类来定义的

Mapper执行业务操作后，将在本机上直接使用combiner操作，具体执行内容应该和reduce内容一致，这样提前聚合，可以有效减少数据量过大时候，减少网络带来的延迟影响.

![image-20221008214923728](/image-20221008214923728.png)

优点：能够减少IO，提升作业性能

局限性：平均数的计算难？



### Job中自定义的Hadoop数据类型

需要实现writable接口和实现write、readFields两个方法，定义一个默认的构造方法。

Write负责将数据写出去：`Out.writeUTF/writeLong …..()`

ReadFields负责将数据从流中读取出来：`This.xxx = in.readUTF ….. ()`

**Ps:必须保证数据写出去和读取的顺序一致**

使用的时候就可以在Mapper的

```java
<LongWritable, Text,Text, YourOwnClass>
```



### Partitioner

决定maptask输出的数据交由哪个reducetask处理

默认实现：分发的key和hash值与reduce task个数取模%

**ReduceTasks的默认数量是1**



### Mapper 实现

+ **KEYIN**: Map任务读数据的偏移量 offset 每行数据起始位置的偏移量 Long
+ **VALUEIN**:Map任务读取数据的value类型，Text
+ **KEYOUT**：map方式自定义实现输出的key类型，Text
+ **VALUEOUT**：map方式自定义实现输出的value类型

但是在指定类型的时候不能直接指定Long、String等java的基本类型， 而是应该是LongWritable这种**<u>org.apache.hadoop.io</u>**包下面的类型，他们实现了比较和(反)序列化的功能

在mapper接口中有一个setup方法，Setup方法可以在Mapper调用map方法之前将一些自定义的对象实例初始化`(xxxx = new xxxx();)`

Map的数量只与输入文件被划分成分块的数量一致



### Reducer实现

**<Mapper的key类型，Mapper的value类型，最终输出的key类型，最终输出的value类型>**

Map的输出，按照相同的key，依据partitioner的分发策略——键的哈希码被转换成一个非负数，具体由哈希值和最大的整型值做一次按位与操作，按照分区数进行取模

```java
(Key.hashCode()&Integer.MAX_VALUE)%numPartitions
```

分发到不同的reduce端。其实现方法reduce()中第二个参数：

`Iterable<XXX>`是map端输出的value值的迭代器，可以直接for循环加get方法获取具体的值，在送出后仍需要将java基本类型包装成hadoop基本类型。

<u>关于设置分区块的数量：建议每个reducer的执行时间5分钟，且产生一个HDFS块</u>



### Shuffle和排序

Map和reducer之间的过程是shuffle

![image-20221008221014749](/image-20221008221014749.png)

#### Map端

Map函数开始产生输出的时，利用缓冲的方式写到(环形)内存(默认是100mb)，并进行预排序。一旦缓冲区内容达到阈值(默认是80%)，后台的线程就会开始把内容溢出(spill)到磁盘，如果在溢出磁盘的过程中出现缓冲区被填满，那么map操作会被阻塞。

写入磁盘之前，线程按照要数据最终传入的reducer端，把数据划分成分区(partition)。每个分区内部，按照键值排序，如果有combiner函数，在sort之后执行。

每次缓冲区达到阈值，就会产生一个溢出文件(spill file)，在map任务完成最后的输出记录后，会存在多个溢出文件，在任务完成之前，溢出文件被合并成一个已经排好序的输出文件。



#### Reduce端

Map输出的文件位于运行map任务的task tracker的本地磁盘上，而reduce的输出并不一定在本地磁盘。考虑到每个map任务的执行时间不定，每当一个map任务结束——使用心跳机制，告知application master,对于指定任务AM会知道map输出和主机位置的映射关系，reduce端的一个线程会定期询问AM，以获取map输出主机的位置，考虑到第一个reducer任务可能会失败，因此不会删除map的输出文件，直到AM告知可以删除。reducer会开始复制map的结果输出(在复制阶段)，在该阶段，会启用少量的并行线程(默认是5)，进行复制。

复制完所有map输出后，reduce任务进入排序阶段，这个阶段合并map输出，如果有50个map输出，合并因子是10，那么就会合并5次，每10个文件合成1个文件，最后有5个中间文件。

在reduce阶段，直接把数据输入reduce函数，从而省略了一次磁盘往返形成，没有把中间文件合并成1个已排序的文件。对已排序的每个键调用reduce函数，结果直接写入输出文件系统，如果是HDFS，那么本地的NM也是数据运行节点，所以第一个副本就会存在本地。



## MapReduce的特性

### 计数器

Hadoop在MapReduce过程中，可以设置枚举类型，添加计数器功能，负责记录已经处理的字节数和记录数，用以监控已处理的数据量和已产生的输出数据量。

 **内置计数器**

每个作业/任务维护若干内置计数器，例如已处理的字节数和记录数

 **用户自定义的计数器**

计数器的值可以在mapper和reducer中，以枚举类型定义。计数器是全局的，MapReduce将跨所有的map和reduce聚集这些计数器，在作业结束时，产生最终结果

### 排序

可以通过排序功能组织数据，通常都是按照key值，完成内部顺序

**<u>部分排序</u>**
		由RawComparator控制，具体规则如下：

1.	mapreduce.job.output.key.comparator.class显示定义，或者Job类的setSortComparatorClass()方法进行设置
2.	或者键必须是WritableComparable子类，并登记Comparator
3.	RawComparator将字节流反序列化成一个对象，再使用WritableComparable的compareTo()方法判断

**<u>全排序</u>**
		可以仅设置一个reduce，完成全排序

### 连接Join

#### Reduce端连接

考虑到不同key在不同map函数中

| mapper1结果key： A、B、B’ | mapper2结果key： A’、C、C’ |
| ------------------------- | -------------------------- |

保证map传来的数据能够区分来自不同的文件，例如添加标记tag，file-0/1，

但是在shuffle阶段(传输，整合相同key值的数据)，可能存在大量的传输消耗

#### Map端连接

可以将相对小的数据集，预先存放到内存中，在map中读取内存

在main函数中添加，job放入文件路径

``` java
job.addCacheFile(new URI(path));
```

在mapper中通过setup()方法，预先将文件进行读取并存放关键数据到hashmap中，再在map函数中使用

#### 多输入

利用MultipleInputs，代替InputFormat，获取多个数据源，并且分别分配Map函数，要求设置相同的分区数

#### 辅助排序

配合MultipleInputs，将不同数据源，实现Partition类，设置共同的key值，确保能够join的数据能够分到同一个分区



### 边数据分布

作业需要的额外只读数据，以辅助处理主要数据

主要是通过hadoop的分布式缓存机制。

通过在执行hadoop -jar xxxx 是设置-files、-archives、-libjars选项，实现文件的复制到分布式文件系统中。

 在任务执行前，NM将文件从HDFS复制到本地磁盘缓存上，以便于能够访问，通过以符号链接的方式指向任务的工作目录

​    而-libjars指定的文件会在任务启动前添加到任务的类路径下。



### 分布式缓存API

Job任务默认提供了将对象放入分布式缓存的方法，例如“MAP端连接”，可以存放：文件files和存档achives，具体方法：
1.	addCachexxxx
2.	setCachexxxxs
3.	addXXXToClassPath
   使用中仍然在setup方法，通过FileReader和BufferReader读取



## YARN资源调度框架

### Yarn的产生背景
Mapreduce1.x版本：
1.	JobTracker的压力过大，是单节点
2.	仅支持mapreduce作业
3.	在资源利用率上效果较差

### 概述
YARN(Yet Another Resource Negotiator)包含以下概念：
1.	Master: Resource management (RM)
   集群中同一时刻对外提供服务的只有1个，负责资源相关，处理来自客户端的请求 ；
   启动/监控AM
   监控NM
   其他资源相关
2.	Client：
   向RM提交任务和杀死任务
3.	Job schedule/monitoring----> applicaitonMaster(AM):
   每提交一个作业(application)，在container中生成一个applicationMaster(AM),
   之后AM向RM申请资源(注册)，用于在NM上的启动对应的Task
   数据切分
   为每个task向RM申请资源（在container内部启动task）
   NM的通信
   任务的监控
4.	Slave: NodeManager(NM)：
   可以有多个，
   负责干活，
   向RM发送心跳，任务的执行情况
   接受来自RM的请求来执行命令
   处理来自AM的命令
5.	Container：任务的抽象运行
   Memory、cpu。。。。
   Task是运行在container中的
   可以运行am、也可以运行map/reduce task

![image-20221008223249826](/image-20221008223249826.png)

![image-20221008223254312](/image-20221008223254312.png)

<u>个人理解</u>

（在作业运行前：RM和NM已经启动），Client提交作业给RM，由RM分配一个NM，并在其内部container中启动AM，AM向RM注册，并启动其他NM的container中的task

### 启动YARN

Yarn需要额外执行，在sbin/文件夹下执行start-yarn.sh,输入jps下会出现Resource Manager和Node Manager两个进程，如果没有出现，则需要在log文件夹下对应的.log日志文件中查询对应的启动记录。

默认8088端口可以查看yarn的图形界面

### 作业提交YARN上的步骤
1.	mvn clean package -DskipTests 项目打包，跳过测试类；
2.	将编译出的.jar文件传入服务器，然后再传入hdfs上
3.	执行作业 hadoop jar  xxx.jar 完整的(启动)类名+包名 入参args 
4.	到8088上查看作业的运行情况



## Hive数据库

### 产生背景

1.	Mapreduce的不方便
2.	RDBMS关系型数据库的需求

**hdfs上的文件没有schema的概念**

### 概述
1.	海量的结构化日志的解决方案
2.	构建在hadoop上的数据仓库（数据存储在hdfs上;可以通过mapreduce计算，并提交到yarn上运行的）
3.	提供SQL查询语言：HQL
4.	底层执行引擎有多种 (在1.x版本中主要是mr，2.x中默认为spark/tez)
5.	提供了统一的元数据管理  : SQL on Hadoop hive、spark sql、impala(在hive中创建的表，在spark sql和impala中可以直接使用)

### 体系架构

![image-20221008223638095](/image-20221008223638095.png)

+ Client：shell(登录到目标服务器)、thrift/jdbc(server/client，写jdbc代码)、webUI

+ Metastore：hive是基于表和数据库操作的，而表的这些基本信息是存储在metastore上，具体是mysql

具体执行仍然通过语法树解析，然后提交到mapreduce上执行

### Hive配置修改
1.	Hive-env.sh 中的HADOOP_HOME=具体填写，在conf中有`hive-env.sh.template`复制一份，
2.	Hive-site.xml 配置元数据的存储位置，默认为dery，需要填写mysql相关信息（第一次需要新建该xml文件），需要在配置文件中`hive.metastore.warehouse.dir`的**指定的路径创建文件夹**，然后对hadoop上该文件夹设置权限`hadoop fs -chmod 777 /...`
3.	拷贝Mysql的驱动jar包到$HIVE_HOME/lib文件夹下
4.	在配置文件中配置
5.	`schematool --dbType mysql -initSchema`初始化mysql中的hive库

```sh
<-->hive-env.sh</-->
export HADOOP_HOME=/app/hadoop-3.3.3

# Hive Configuration Directory can be controlled by:
export HIVE_CONF_DIR=/app/apache-hive-3.1.3-bin/conf

# Folder containing extra libraries required for hive compilation/execution can be controlled by:
export HIVE_AUX_JARS_PATH=/app/apache-hive-3.1.3-bin/lib

export JAVA_HOME=/app/jdk1.8.0_341
```



```xml
// Hive-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<property>
  <name>hive.metastore.warehouse.dir</name>
  <value>hdfs://192.168.199.177:8020/hive/warehouse</value>
  <description>管理表存储的位置,可以是linux中的目录,也可以是相对于fs.default.name有关的目录</description>
</property>
<property>
    <name>hive.metastore.local</name>
    <value>false</value>
</property>

    # 设置查询时候的列表名、显示当前数据库
<property>
    <name>hive.cli.print.header</name>
     <value>true</value>
     <description>Whether to print the names of the columns in query output.</description>
</property>
<property>
      <name>hive.cli.print.current.db</name>
      <value>true</value>
      <description>Whether to include the current database in the Hive prompt.</description>
</property>
<property>
    <name>hive.resultset.use.unique.column.names</name>
    <value>false</value
</property>
        
    # 基本设置
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://hadoop000:3306/hadoop_hive?createDatabaseIfNotExist=true</value>
</property>
        
<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>root</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>root</value>
</property>
</configuration>

```

```sh
# profile
export HIVE_PATH=/app/apache-hive-3.1.3-bin
export PATH=$PATH:$HIVE_PATH/bin
```





### Hive操作

Hive可以直接通过本地加载文件到表中，但是需要：
1.	在创建表的时候指定ROW FORMATE DELIMITED FIELDS TERMINATED BY ’ ’分隔符（默认的数据行之间的间隔符为“回车”）
2.	导入数据的时候：load data local inpath ‘文件路径’ overwrite into table

由于hive本质上是将sql语法解析成为yarn作业，可以通过8088端口查看yarn作业执行情况

https://hive.apache.org/

### DDL Hive data definition language
Hive数据抽象

> ​	Database:	HDFS一个目录
> >​		Table:	HDFS一个目录
> >> ​	Data 文件 (如果没有分区)
> >> ​			Partition 分区表	HDFS一个目录
> >>
> >> > ​		Data 文件 如果有分区
> >> > ​				Bucket 分桶	HDFS一个文件

### DML data manipulate language 数据管理

**加载、查询、导出:**

`Load data [local]/无 （有则本地，无则hdfs上的文件） inpath xxxxx [overwrite] into table yet`

xxx为pwd的查询地址

**插入:**

`Insert overwrite local directory ‘文件路径’ row format delimited field terminated by ‘\t’ 子查询语句（需要导出的内容）`

对应的**array**类型和**map**类型，仍然需要指定分隔符

`collection items terminated by ‘_’`和` map keys terminated by ‘:’`

创建表：

```hive
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name
　　[(col_name data_type [COMMENT col_comment], ...)]
　　[COMMENT table_comment]
　　[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
　　[CLUSTERED BY (col_name, col_name, ...)
　　　　[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
　　[ROW FORMAT row_format]
　　[STORED AS file_format]
　　[LOCATION hdfs_path]
```





### 内部表/外部表
Hive在load data后，如果是hdfs上，<u>会自动删除原来位置上的文件</u>。因为load本质上就是在hdfs上的数据转移

其中如果load进的表为内部表，那么：
1.	在删除的时候，该表的元数据和数据都会删除
2.	Ps 默认没有指明的表都是内部表
如果表为外部表，指明为external，在删除之后，只会删除元数据，hdfs中还有数据，如果出现误删，可以再创建一个表，将数据导入

**<u>注意：hive表的元数据在mysql中查询</u>** 

### 分区表

在创建的时候指定 `partitioned by (分区字段， 分区字段类型) row format 。。。。`

在load data 的时候也需要指定具体的分区 partition(分区字段=’xxxxx’)

通过hdfs查看文件的时候，能够看到吗，每个分区表对应的是一个HDFS文件系统上的一个文件夹。其中分桶对应文件，在hive中可以在where条件中指定分区字段=‘xxxx’ 查看分区表的内容。

<u>可以通过sqoop将数据导出到RDBMS中</u>

整体步骤：

1.	ETL  （利用yarn 执行hadoop jar 。。。。）
2.	将结果加载到分区表中(load …)
3.	可以将分区表中的数据按照维度统计到维度表中(insert …)
4.	可以将数据利用sqoop导出

`partitioned by`子句中的列定义是表中正式的列——**分区列**，但是数据文件不会包括这些列的值，因为他们来自于目录名。

如果查询不存在的分区表，那么还是会**返回分区的列和值！！！！！**

### 查询数据

#### 排序

+ **Sort by**可以指代**order by**， 并且避免全排列；

+ 如果需要控制特定的行分配到哪个reducer，使用**distribute by**
+ **Sort by** 和**distribute by**同时使用时，可以直接用**cluster by**代替

#### MapReduce脚本

在Hive中可以直接使用python脚本(操作逻辑)，通过**hive> ADD File**命令，**配合using“脚本”AS输出**。

```shel
> From 某表
> Select transform（a,b,c）
> Using ‘脚本.py’
> As a, b

```

其中的脚本.py内容如下

```pytho
For line in sys.stdin:
	_ = line.strip().split()
# 操作逻辑

```

如果嵌套查询，**Map、Reduce**关键词也能获取和select transform 一致的效果

#### 链接

(left/right) Join on

Hive自动以智能的方式减少执行MapReduce作业数，如果在多个连接的连接条件中使用相同的列，那么就会少MapReduce

Left join 和 join 不会执行reduce操作

而right 和full outer join，由于需要检测非空值，汇总join结果需要reduce

#### 子查询

#### 视图

### 与传统数据库相比

Hive数据库采用的是**读时模式**，对Hive来说，数据依托于HDFS，其加载过程也是在HDFS上的文件复制或者移动，所以数据的加载过程非常迅速。

传统数据库采用的是**写时模式**，在加载数据的时候，需要对数据进行“解析”，再进行序列化并以数据库内部格式存入磁盘。因为其表的模式是数据在加载过程中强制确定的，在数据记载中，如果出现数据不符合模式，就会拒绝加载数据。

**写时模式**，有利于提升查询性能，因为数据库可以对列进行索引。

HDFS不同就地文件更新，插入、更新、删除等操作导致的变化会存储在一个较小的增量文件中。



### sql常用功能

#### 列转行

```hive
select course
from 
xxx
lateral view # 配合split、explode将一行数据拆分成多行
explode(split(courses,"正则或者分隔符")) course_tmp as course; 
#explode 将数组拆分，split将字符分为数组 都是udf，其中explode是udtf
```



### 窗口函数

窗口：函数运行时计算的数据集的范围

函数：运行时执行的函数

（<u>聚合函数，只能满足返回的结果是一行</u>）

```hive
# 0.范围
over(partition by xx order by xx rows between unbounded preceding and current row)
unbounded 表示从头开始，搭配 preceding 和 following 分别表示最开始和最后
n preceding 表示前面n行
n following 表示后面n行

# 1.求和
sum() over(partition by ...)

# 2.找值
LEAD(col, n, default) 	# 朝下去找
LAG(col, n, default)	# 朝上去找
col: 要找哪几个字段
n：几条
default：没有找到就是default

# 3.最开始值和最后值
first_value 表示当前分组中的第一个值
last_value 表示当前分组中的最后一个值(注意：根据语法可能是指截止当前行的最后一个)

# 4.切片
ntile(n) # 表示对当前分组的数据进行切分

# 5.排序编号
row_number() 对不同分组按照顺序排序， “分组和顺序”的排序方式 需要在 over()中指明
rank() 分组生成编号，排序相同就会重复编号，但是下一个会保持编号的上位总数 例如 1,2,3,3,5
dense_rank()  编号不会间断 例如 1,2,2,3

# 6.
cume_dist() 小于等于当前行值的行数 / 分组内的总行数
```

