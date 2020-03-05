

# 20_Hive

## 1.基本概念

### 1.1 什么是hive

1. Hive是由Facebook开源用于解决海量结构化日志的数据统计
2. Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射成为一张表，并提供类SQL查询功能
3. **本质是将HQL/SQL转化为MapReduce**
4. Hive处理的数据存储在HDFS
5. Hive分析数据底层的实现是MapReduce
6. 执行程序运行在yarn上

### 1.2 Hive的优缺点

1. 优点：
   1. 操作接口采用了类似SQL语法，提供了快速开发的能力
   2. 避免了去写MapReduce，减少开发人员的学习成本
   3. Hive优势在于处理大数据，**对于处理小数据没有优势，因为hive的执行延迟比较高**
   4. Hive的执行延迟比较高，**因此Hive常用于数据分析，对实时性要求不高的场合**
   5. Hive支持用户自定义函数，用户可以根据字的需求来实现自己的函数
2. 缺点：
   1. hive的HQL表达能力有限
      - 迭代式算法无法表达
      - 数据挖掘方面不擅长
   2. Hive的效率比较低
      - Hive自动生成的MapReduce作业，通常情况下不够智能化
      - Hive调优比较困难，粒度较粗

### 1.3Hive架构原理

![](img\hive\hive架构.png)

如图所示，hive通过给用户提供一系列交互接口，接收到用户的指令（SQL），使用自己的Driver，结合元数据（MetaStore），将这些指令翻译为MapReduce，提交到Hadoop中执行，最后将执行的返回结果输出到用户交互接口

1. 用户接口：client

   CLI（hive shell）、JDBC/ODBC(java访问hive)、WEBUI（浏览器访问hive）

2. 元数据：MetaStore

   元数据包括：表名、表的所属的数据库（默认是default）、表的拥有者，列/分区字段、表的类型（是否是外部表）、表的数据所在目录等；

   **默认存储在自带的derby数据库中，推荐使用MySQL存储MetaStore**

3. Hadoop

   使用HDFS进行存储，视同MapReduce进行计算

4. 驱动器：Driver

   - 解析器（SQL Parser）：将SQL字符串转换成抽象语法树AST，这一步一般用第三方工具库完成，比如antlr；对AST进行语法分析，比如表是否存在、字段是否存在、SQL语义是否有误。
   - 编译器（Physical Plan）：将AST编译生成逻辑执行计划
   - 优化器（Query Optimizer）：对逻辑执行计划进行优化
   - 执行器（Execution）：把逻辑执行计划计划转化成可以运行的物理计划。对于Hive来说，就是MR/Spark

### 1.4Hive和数据库比较

由于hive采用可类似SQL的查询语言，HQL(Hive Query Language)，因此很容易将Hive理解为数据库。其实从结构上来看，Hive和数据库除了拥有类似查询的查询语言，再无类似之处。数据库可以用在Online的应用之中，但是Hive是为数据仓库而设计的，清楚这一点，有助于从应用角度理解Hive的特性

1. 查询语言

   由于SQL被广泛的应用在数据仓库中，因此，专门针对hive的特性设计了类SQL语言的查询语言HQL。熟悉SQL的开发者可以很方便的使用Hive进行开发

2. 数据存储位置

   Hive是建立在Hadoop之上的，所有Hive的数据都是存储在HDFS中的。而数据库则可以将数据保存在块设备或者本地文件系统中

3. 数据更新

   由于Hive是针对数据仓库应用设计的，而数据仓库的内容**是读多写少的**。因此，**Hive中不支持对数据的改写和添加，所有的数据都是在加载的时候确定好的。**而数据库中的数据通常是需要经常进行修改的，因此可以使用 insert into .... values 添加数据，使用update ... set 修改数据

4. 索引

   Hive在加载数据的过程中不会对数据进行任何处理，甚至不会对数据进行扫描，因此也没有对数据中的某些key建立索引。Hive要访问数据中满足条件的特定值时，需要暴力扫描整个数据，因此即使没有索引，对于大数据量的访问，Hive仍然可以体现出优势。数据库中，通常会针对一个或几个列建立索引，因此对于少量的特定条件的数据的访问，数据库可以有很高的效率，较低的延迟。由于数据的访问延迟较高，决定了hive不适合在线数据查询。

5. 执行

   Hive中大多数查询的执行是通过Hadoop提供的MapReduce来实现的。而数据库通常有自己的执行引擎

6. 执行延迟

   Hive在查询的时候，由于没有索引，需要扫描整个表，因此延迟比较高。另一个导致Hive执行延迟高的因素是MapReduce框架。由于MapReduce本身具有较高的延迟，因此在利用MapReduce执行hive查询时，也会有较高的延迟。相对的，数据库的执行延迟较低。当然，这个低是有条件的，即数据规模比较小，当数据规模大的超过数据库的处理能力的时候，Hive的并行计算显然能体现出优势

7. 可扩展性

   由于Hive是建立在Hadoop上的，因此Hive的可扩展性和Hadooop的可扩展性是一致的（世界上最大的Hadoop集群在yahoo！，2009年的规模在4000 台节点左右）。而数据库由于ACID语义有严格限制，扩展行非常有限。目前最先进的并行数据库Oracle在理论上的扩展能里也只有100台左右。

8. 数据规模

   由于Hive建立在集群上并可以利用MapReduce进行并行计算，因此可以支持很大规模的数据，对应的，数据库可以支持的数据规模比较小



## 2.Hive安装及相关操作

**注：hive在安装之前必须先安装MySql**

Hive官网地址：

http://hive.apache.org/

文档查看地址：

https://cwiki.apache.org/confluence/display/Hive/GettingStarted

下载地址：

http://archive.apache.org/dist/hive/

github地址：

https://github.com/apache/hive

### 2.1 hive 安装

1. 上传hive安装包在/opt/software目录下

2. 解压hive安装包至/opt/module目录下面

   ```
    tar -zxvf apache-hive-1.2.1-bin.tar.gz -C /opt/module/
   ```

3. 修改目录名

   ```
   mv apache-hive-1.2.1-bin/ hive-1.2.1
   ```

4. 修改conf安装目录下的hive-env.sh.template名称为hive-env.sh

   ```
   mv hive-env.sh.template hive-env.sh
   ```

5. 配置hive-env.sh文件

   1. 配置HADOOP_HOME路径

      ```
      export HADOOP_HOME=/opt/module/hadoop-2.8.4
      ```

   2. 配置HIVE_CONF_DIR路径

      ```
      export HIVE_CONF_DIR=/opt/module/hive/conf
      ```

   注：Hive的log默认存放在/tmp/itstar/hive.log目录下（当前用户名下）

6. 修改hive的log日志存放到/opt/module/hive/logs**（可选）**

   1. 修改conf/hive-log4j.properties.template文件名称为hive-log4j.properties

      ```
      mv hive-log4j.properties.template hive-log4j.properties
      ```

   2. 在hive-log4j.properties文件中修改log存放位置

      ```
      hive.log.dir=/opt/module/hive/logs
      ```
      
   3. 在/opt/module/hive目录下创建logs文件夹

      ```
      mkdir logs
      ```

      

7. Hadoop集群配置

   1. 必须启动hdfs和yarn

      ```
      start-dfs.sh
      start-yarn.sh
      ```

   2. 在HDFS上创建/tmp和/user/hive/warehouse两个目录并修改他们的同组权限可写

      ```
      bin/hadoop fs -mkdir /tmp
      bin/hadoop fs -mkdir /user/hive/warehouse
      
      bin/hadoop fs -chmod g+w /tmp
      bin/hadoop fs -chmod g+w /user/hive/warehouse
      ```



### 2.2 hive的基本操作

1. 启动hive

   ```
   hive
   ```

2. 查看数据库

   ```
   show databases;
   ```

3. 打开默认数据库

   ```
   use default;
   ```

4. 显示default数据库中的表

   ```
   show tables;
   ```

5. 创建一张表

   ```
   create table student(id int,name string);
   ```

6. 显示数据库中有几张表

   ```
   show tables;
   ```

7. 查看表的结构

   ```
   desc student;
   ```

8. 向表中插入数据

   ```
   insert into student values(1001,"w1");
   ```

9. 查询表中数据

   ```
   select * from student;
   ```

10. 在hive 命令窗口如何查看hdfs**文件系统**

    ```
    dfs -ls /;
    ```

11. 在hive命令窗口如何查看hdfs**本地系统**

    ```
    ! ls /opt/module/datas;
    ```

12. 查看在hive中输入的所有历史命令

    - 进入到当前用户的根目录`/root`或`/home/bigdata111`

    - 查看`.hivehistory`文件

      `cat .hivehistory`

13. 退出hive

    ```
    exit;
    或
    quit;
    ```

### 2.3 Hive元数据配置到MySql

1. 驱动拷贝

   上传mysql-connector-java-5.1.27-bin.jar到/opt/module/hive/lib/

2. 配置MetaStore到MySql

   1. 在/opt/module/hive/conf目录下创建一个hive-site.xml

      ```
      touch hive-site.xml
      vim hive-site.xml
      ```

   2. 根据官方文档配置参数，拷贝数据到hive-site.xml文件中。

      https://cwiki.apache.org/confluence/display/Hive/AdminManual+MetastoreAdmin

      ```xml
      <?xml version="1.0"?>
      <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
      <configuration>
      	<property>
      	  <name>javax.jdo.option.ConnectionURL</name>
      	  <value>jdbc:mysql://bigdata111:3306/metastore?createDatabaseIfNotExist=true</value>
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
      	  <value>000000</value>
      	  <description>password to use against metastore database</description>
      	</property>
      </configuration>
      ```

   3. 配置完毕以后，如果启动hive异常，可以重新启动虚拟机

   4. 配置proflie

      vim /etc/profile

      ```
   #hive
      export HIVE_HOME=/opt/module/hive-2.1.1
      export PATH=$PATH:$HIVE_HOME/bin
      ```
   
      source /etc/profile
   
   5. 在hive的bin目录下执行 ./schematool -dbType mysql -initSchema
   
      如图显示：
   
      1.2.1版本
   
      ![](img\hive\1.2.1版本hive.png)
      
      2.3.4版本
      
      ![](img\hive\2.3.0版本hive.png)

### 2.3 Hive常见属性配置

1. Hive数据仓库位置配置

   1. Default仓库的原始位置是在hdfs上的：/user/hive/warehouse路径下

   2. **在仓库目录下，没有对默认的数据库default创建文件夹。如果某张表属于default的数据库，直接在数据仓库目录下创建一个文件夹**

   3. 修改default数据仓库原始位置，（将hive-default.xml.template如下配置信息拷贝到hive-site.xml文件中）

      ```xml
      <property>
      <name>hive.metastore.warehouse.dir</name>
      <value>/user/hive/warehouse</value>
      <description>location of default database for the warehouse</description>
      </property>
      ```

      配置同组用户拥有执行权限

      ```
      bin/hdfs dfs -chmod g+w /user/hive/warehouse
      ```

2. 查询后信息显示配置

   1. 在hive-site.xml文件中添加如下配置信息，就可以实现显示当前数据库，以及查询表的头信息配置。

      ```xml
      <property>
      	<name>hive.cli.print.header</name>
      	<value>true</value>
      </property>
      
      <property>
      	<name>hive.cli.print.current.db</name>
      	<value>true</value>
      </property>
      ```

   2. 重新启动hive，对比配置前后差异

      1. 配置前

         ![](img\hive\hive配置头信息前.png)

      2. 配置后

         ![](img\hive\hive配置头信息后.png)

3. Hive运行日志信息配置

   1. Hive的log默认存放在/tmp/itstar/hive.log目录下（当前用户名下）。

   2. 修改hive的log存放日志到/opt/module/hive/logs

      1. 修改/opt/module/hive/conf/hive-log4j.properties.template文件名称为

         hive-log4j.properties

         ```
         mv hive-log4j.properties.template hive-log4j.properties
         ```

      2. 在hive-log4j.properties文件中修改log存放位置

         ```
         hive.log.dir=/opt/module/hive/logs
         ```

4. 多窗口启动Hive测试

   1. 先启动mysql

      ```
      mysql -uroot -p000000
      ```

   2. 查看有几个数据库

      ```
      mysql> show databases;
      +--------------------+
      | Database           |
      +--------------------+
      | information_schema |
      | mysql             |
      | performance_schema |
      | test               |
      +--------------------+
      ```

   3. 再次打开多个窗口分别启动hive

      ```
      hive
      ```

   4. 启动hive后，回到Msql窗口查看数据库，显示增加了metastore数据库

      ```
      mysql> show databases;
      +--------------------+
      | Database           |
      +--------------------+
      | information_schema |
      | metastore          |
      | mysql             |
      | performance_schema |
      | test               |
      +--------------------+
      ```


### 2.4 hive的交互方式

1. bin/hive

   ```
   cd /opt/module/hive-2.3.4/
   bin/hive
   ```

   创建一个数据库库

   ```
   create database if not exists mytest;
   ```

2. 使用sql语句或者sql脚本进行交互

   不进入客户端直接执行hive的hql语句

   ```
   cd /opt/module/hive-2.3.4/
   bin/hive -e 'create database if not exists my test;'
   ```

   或者将hql语句写成一个sql脚本然后执行

   ```
   vim hive.sql
   createdatabase if not esists mytest;
   use mytest;
   create table stu(id int,name string);
   ```

   通过hive -f 来执行sql脚本

   ```
   hive -f hive.sql
   ```



## 3.hive操作

### 3.1 数据库操作

1. 创建数据库

   ```
   create database if not exists myhive;
   use myhive;
   ```

   说明：hive的表存放位置模式是由hive-site.xml当中的一个属性执行的

   ```
   <name>hive.metastore.warehouse.dir</name>
   <value>/user/hive/warehouse</value>
   ```

2. 创建数据库并指定位置

   ```
   create database myhive locationn 'myhive';
   ```

3. 设置数据库键值对信息

   数据库可以有一些描述性的键值对信息，在创建时添加

   ```
   create database foo with dbproperties('owner' = 'song',date='201920');
   ```

   查看数据库的键值对信息

   ```
   describe database extened foo;
   ```

   修改数据库的键值对信息

   ```
   alter database foo set dbproperties('owner'='song');
   ```

4. 查看数据库更多详细信息

   ```
   desc database extend myhive;
   ```

5. 删除数据库

   删除一个空数据库库，如果数据库下面有数据表，那么就会报错

   ```
   drop database myhive;
   ```

   强制删除数据库，包含数据库下面的表一起删除

   ```
   drop database myhive cascade;
   ```

### 3.2数据库表操作

#### 3.2.1创建表的语法

```sql
create [external] table [if not exists] table_name (
col_name data_type [comment '字段描述信息']
col_name data_type [comment '字段描述信息'])
[comment '表的描述信息']
[partitioned by (col_name data_type,...)]
[clustered by (col_name,col_name,...)]
[sorted by (col_name [asc|desc],...) into num_buckets buckets]
[row format row_format]
[storted as ....]
[location '指定表的路径']
```

说明：

1. create table

   创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以使用 `if not esists`选项来忽略这个异常操作

2. external

   可以让用户创建一个外部表，在建表的同时指向一个实际数据库的路径（location），hive创建内部表时，会将数据移动到数据仓库指向的路径；若创建外部表，仅记录数据所在的路径，不对数据的位置做任何改变。在删除表的时候，内部表的元数据和数据会被一起删除

3. comment

   表示注释，默认不能使用中文

4. partitioned by

   表示使用表分区，一个表可以拥有一个或多个分区，每一个分区单独存放在一个目录下。

5. clusterered by

   对于每一个表分文件，hive可以进一步组织成桶，也就是说桶是更为细粒度的数据范围划分。hive也是针对某一列进行桶的组织

6. sorted by

   指定排序字段和排序规则

7. row format

   指定表文件字段分隔符

8. storted as

   指定表文件的存储格式，常用格式：SEQUENCEFILE, TEXTFILE, RCFILE,如果文件数据是纯文本，可以使用 STORED AS TEXTFILE。如果数据需要压缩，使用storted as SEQUENCEFILE。

9. location

   **指定表文件的存储路径**
   
10. like

    允许用户复制现有的表结构，但不复制数据

#### 3.2.2内部表

创建表时，如果没有使用external关键字，则该表为内部表（managed table）

**概念：**

默认创建的表都是所谓的管理表，有时也称为内部表。因为这种表，hive会（或多火少地）控制着数据的生命周期。hive默认情况下会将这些表的数据存储在由配置项`hive.metastore.warehouse.dir(例如，/user/hive/warehouse)`所定义的目录的子目录下。当**我们们删除一个管理表时，hive也会删除这个表的数据。**管理表不适合和其他工具共享数据

**hive建表字段类型**

| 分类     | 类型      | 描述                                            | 字面量示例                                                   |
| -------- | --------- | ----------------------------------------------- | ------------------------------------------------------------ |
| 原始类型 | boolean   | true/false                                      | ture                                                         |
|          | tinyint   | 1字节的有符号整数，-128~127                     | 1Y                                                           |
|          | smallint  | 2字节的有符号整数，-32768~32768                 | 1S                                                           |
|          | int       | 4字节的带符号整数                               | 1                                                            |
|          | bigint    | 8字节带有符号整数                               | 1L                                                           |
|          | float     | 4字节单精度浮点数                               | 1.0                                                          |
|          | double    | 8字节双精度浮点数                               | 1.0                                                          |
|          | deicimal  | 任意精度的带符号小数                            | 1.0                                                          |
|          | string    | 字符串，变长                                    | 'a','b'                                                      |
|          | varchar   | 变长字符串                                      | 'a','b'                                                      |
|          | char      | 固定长度字符串                                  | 'a','b'                                                      |
|          | binary    | 字节数组                                        | 无法表示                                                     |
|          | timestamp | 时间戳，毫秒精度值                              | 122327493795                                                 |
|          | date      | 日期                                            | ’2019-03-24‘                                                 |
|          | interval  | 时间频率间隔                                    |                                                              |
| 复杂类型 | array     | 有序的同类型集合                                | array(1,2)                                                   |
|          | map       | key-value，key必须为原始类型，value可以任意类型 | map('a',1,'b',2)                                             |
|          | struct    | 字段集合，类型可以不同                          | struct(‘1’,1,1.0), named_stract(‘col1’,’1’,’col2’,1,’clo3’,1.0) |
|          | union     | 在有限取值范围类的一个值                        | create_union(1,'a',63)                                       |

1. 建表入门  

   ```sql
   use hive;
   create table stu(id int,name string);
   insert into stu value(1,'张三');
   select * from stu;
   ```

2. 创建表并指定字段之间的分隔符

   ```sql
   create table if not exists stu2(id int,name string) row format delimited fields terminated by '\t';
   ```

3. 创建表并指定表文件的存放路径

   ```sql
   create table if not exists stu2(id int,name string) row format delimited fields terminated by '\t';
   ```

4. 根据查询结果创建表

   ```sql
   create table stu3 as select * from stu2;
   ```

5. 根据已经存在的表结构创建表

   ```sql
   create table stu4 like stu;
   ```

6. 查询表的详细信息

   ```sql
   desc formatted stu2;
   ```

7. 删除表

   ```sql
   drop table stu4;
   ```

#### 3.2.3外部表

**外部表说明：**

因为表是外部表，所以Hive并非认为其完全拥有这份数据。**删除该表并不会删除掉这份数据，不过描述表的元数据信息会被删除掉。**

**内部表和外部表的使用场景**

每天将收集到的网站日志定期流入hdfs文本文件。在外部表（原始日志表）的基础上做大量的统计分析，用到的中间表、结果表使用内部表存储，数据通过select + insert进入内部表

**操作案例**

分别创建老师与学生外部表，并向表中加载数据

1. 创建老师表

   ```sql
   create external table teacher(t_id string, t_name string) row format delimited fields terminated by '\t';
   ```

2. 创建学生表

   ```sql
   create external table student(s_id string ,s_name string,s_birth string,s_sex string) row format delimited fields terminated by '\t';
   ```

3. 加载数据

   ```sql
   load data local inpath '/opt/module/datas/student.csv' into table student;
   ```

4. 加载数据并覆盖已有的数据

   ```sql
   load data local inpath '/opt/module/datas/student.csv' overwrite into table student;
   ```

5. 从hdfs文件向表中加载数据（需要提前将数据上传到hdfs文件系统）

   ```sql
   hdfs dfs -mkdir -p /hivedatas
   hdfs dfs -put teachar.csv /hivedatas
   
   load data inpath '/hivedatas/teacher.csv' into table teacher;
   ```

#### 3.2.4分区表

**概念：**

分区表实际上就是对应一个HDFS文件系统上的独立的文件夹，该文件夹下是该分区所有的数据文件。Hive中的分区就是分目录，把一个大的数据集根据业务需要分割成小的数据集。在查询时通过WHERE子句中的表达式选择查询所需要的指定的分区，这样的查询效率会提高很多。

> 在大数据中，最常用的一种思想就是分治，我们可以把大的文件划分成一个个的小的文件，这样每次操作一个小的文件就会很容易了。同样的道理，在hive当中也是支持这种思想的，就是我们可以把大的数据，按照每月，或者天进行切分成一个个的小的文件，存放在不同的文件夹中。
>

1. 创建分区表的语法

   ```sql
   create table score(s_id string,c_id string,s_score int) partitioned by (month string) row format delimited fields terminated by '\t';
   ```

2. 创建一个表带多个分区

   ```sql
   create table score2(s_id string,c_id string,s_score int) partitioned by (year string ,month string,day string) row format delimited fields terminated by '\t';
   ```

3. 加载数据到分区表中

   ```sql
   load data local inpath '/opt/module/datas/score.csv' into table score partition  (month='201909');
   ```

4. 加载数据到多分区表中

   ```sql
   load data local inpath '/opt/module/datas/score.csv' into table score2 partition(year='201902',month='20',day='03');
   ```

5. 多分区表联合查询（使用union all)

   ```sql
   select * from score where month='201903' union all select * from score where month='201909';
   ```

6. 查看分区

   ```sql
   show partitions score;
   ```

7. 查看分区表表结构

   ```sql
   desc formatted score
   ```

8. 添加一个分区

   ```sql
   alter table score add partition(month='201902');
   ```

9. 删除分区

   删除单个分区：

   ```sql
   alter table score drop partition(month='201920');
   ```

   删除多个分区：

   ```sql
   alter table dept_partition drop partition (month='201805'), partition (month='201806');
   ```

   

#### 3.2.5分区表综合练习

**需求描述**

现在有一个文件score.csv文件，存放在集群的'/scoredatas/month=201909'目录下，这个文件每天都会生成，存放到对应的日期文件夹下面去，文件别人也需要公用，不能移动。需求：创建hive对应的表，并将数据加载到表中，进行数据统计分析，且删除表之后，数据不能删除。

1. 数据准备

   ```
   hdfs dfs -mkdir -p /scoredatas/month=201909
   hdfs dfs -put score.csv /scoredatas/month=201909
   ```

2. 创建外部分区表，并指定文件数据存放目录

   ```sql
   create external table score4(s_id string,c_id string,s_score int) partitioned by(month string) row format delimited fields terminated by '\t' loaction '/scoredatas';
   ```

3. 进行表的修复（建立表与数据文件之间的一个关系映射）

   ```sql
   msck repair table score4;
   ```









#### 3.2.6 分桶表

**分区针对的是数据的存储路径；分桶针对的是数据文件。**

分区提供一个隔离数据和优化查询的便利方式。不过，并非所有的数据集都可形成合理的分区，特别是之前所提到过的要确定合适的划分大小这个疑虑。

分桶是将数据集分解成更容易管理的若干部分的另一个技术。

分桶，就是将数据按照指定的字段进行划分到多个文件当中去，分桶就是MapReduce当中的分区。

**开启hive的分桶功能**

```
set hive.enforce.bucketing=true;
```

**设置Reduce个数**

```
set mapreduce.job.reduces=3;
```

**创建分桶表**

```sql
create table course(c_id string,c_name string,t_id string)clustered by(c_id)into 3 buckets row format delimited fields terminated by '\t';
```

分桶表的数据加载，由于通标的数据加载通过hdfs dfs -put 文件或者通过load data均不好使，只能通过insert overwrite

创建普通表，并通过insert overwriter的方式将普通表的数据通过查询的方式加载到分桶表中去

1. 创建普通表

   ```sql
   create table course_common(c_id string,c_name string,t_id string)row format delimited fields terminated by '\t';
   ```

2. 普通表中加载数据

   ```sql
   load data local inpath '/opt/module/datas/course.csv' into table course_common;
   ```

3. 通过insert overwriter 给桶表中加载数据

   ```sql
   insert overwirter table course select * from course_common cluster by(c_id);
   ```


#### 3.2.7分桶表抽样调查

对于非常大的数据集，有时用户需要使用的是一个具有代表性的查询结果而不是全部结果。Hive可以通过对表进行抽样来满足这个需求。

查询表stu中的数据

```sql
select * from stu_buck tablesample(bucket 1 out of 4 on id);
```

**注：**tablesample是抽样语句，语法：TABLESAMPLE(BUCKET x OUT OF y) 。

```sql
select * from stu_buck tablesample(bucket 1 out of 3 on id);
```

**不是桶数的倍数或者因子也可以，但是不推荐。**

y必须是table总bucket数的倍数或者因子。hive根据y的大小，决定抽样的比例。例如，table总共分了4份，当y=2时，抽取(4/2=)2个bucket的数据，当y=8时，抽取(4/8=)1/2个bucket的数据。

```sql
hive (default)> select * from stu_buck tablesample(bucket 1 out of 2 on id);
```

**x表示从哪个bucket开始抽取。**例如，table总bucket数为4，tablesample(bucket 4 out of 4)，表示总共抽取（4/4=）1个bucket的数据，抽取第4个bucket的数据。

```sql
select * from stu_buck tablesample(bucket 1 out of 8 on id);
```

**注意：x的值必须小于等于y的值**，否则

FAILED: SemanticException [Error 10061]: Numerator should not be bigger than denominator in sample clause for table stu_buck

#### 3.2.8 数据块抽样

Hive提供了另外一种按照百分比进行抽样的方式，这种是基于行数的，按照输入路径下的数据块百分比进行的抽样。

```sql
hive (default)> select * from stu tablesample(0.1 percent) ;
```

提示：这种抽样方式不一定适用于所有的文件格式。另外，这种抽样的最小抽样单元是一个HDFS数据块。因此，如果表的数据大小小于普通的块大小128M的话，那么将会返回所有行。

### 3.3 修改表结构

1. 重命名

   ```sql
   alter table old_table_name rename to new_table_name
   ```

   把表socre4修改成score5

   ```sql
   alter table score4 rename to score5
   ```

2. 增加/修改列信息

   - 查询表结构

     ```sql
     desc score5;
     ```

   - 添加列

     ```sql
     alter table score5 add colums (mycol string,mysco int);
     ```

   - 更新列

     ```sql
     alter table score5 change column mysco mysconew int;
     ```

   - 删除表

     ```sql
     drop table score5;
     ```

   

## 4.DML数据操作

**语法**

```sql
hive>load data [local] inpath '/opt/module/datas/student.txt' [overwrite] into table student [partition (partcol1=val1,…)];
```

1. load data:表示加载数据
2. local:表示从本地加载数据到hive表；否则从HDFS加载数据到hive表
3. inpath:表示加载数据的路径
4. overwrite:表示覆盖表中已有数据，否则表示追加
5. into table:表示加载到哪张表
6. student:表示具体的表名
7. partition:表示上传到指定分区

### 4.1 数据导入

#### 4.1.1直接向分区表中加载数据

```sql
create table score3 like score;
insert into table score3 partition(month='201909')values('001','002','003')
```

#### 4.1.2通过查询插入数据

- 通过load方式加载数据

  ```sql
  load data local inpath '/opt/module/datas/score.csv' overwriter into table score partition(month='201909');
  ```

- 通过查询方式加载数据

  ```sql
  create table score4 like score;
  insert overwrite table score4 partition(month='201909') select s_id,c_id,s_score from score;
  ```

#### 4.1.3查询语句中创建表并加载数据（As Select）

```sql
create table if not exists student3
as select id, name from student;
```

#### 4.1.4创建表时通过Location指定加载数据路径

1. 创建表，并指定在hdfs上的位置

   ```sql
   create table if not exists student5(
                 id int, name string
                 )
                 row format delimited fields terminated by '\t'
                 location '/user/hive/warehouse/student5';
   ```

2. 上传数据到hdfs上

   ```sql
    dfs -put /opt/module/datas/student.txt  /user/hive/warehouse/student5;
   ```

3. 查询数据

   ```sql
    select * from student5;
   ```

#### 4.1.5  Import数据到指定Hive表中

> 注意：先用export导出后，再将数据导入。同在HDFS上是Copy级操作

```
export table default.student to '/user/hive/warehouse/export/student';
```

```
import table student2 partition(month='201809') from '/user/hive/warehouse/export/student';
```

### 4.2 数据导出

#### 4.2.1 insert 导出

1. 将查询的结果导出到本地,数据之间无间隔

   ```sql
    insert overwrite local directory '/opt/module/datas/export/student'
               select * from student;
   ```

2. 将查询的结果格式化导出到本地,数据之间"\t"间隔

   ```sql
    insert overwrite local directory '/opt/module/datas/export/student1'
                ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'             select * from student;
   ```

3. 将查询的结果导出到HDFS上(没有local)

   ```sql
   insert overwrite directory '/user/itstar/student2'
                ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' 
                select * from student;
   ```

   **注：**虽然同是HDFS，但不是copy操作

#### 4.2.2 hadoop命令导出

```sql
dfs -get /user/hive/warehouse/student/month=201809/000000_0  /opt/module/datas/export/student3.txt;
```

#### 4.2.3hive shell命令导出

基本语法：（hive -f/-e 执行语句或者脚本 > file）

```sql
bin/hive -e 'select * from default.student;' > /opt/module/datas/export/student4.txt;
```

#### 4.2.4  Export导出到HDFS上

```sql
 export table default.student to '/user/hive/warehouse/export/student';
```

#### 4.2.5  Sqoop导出

后续学习

#### 4.2.6清除表中数据（Truncate）

注意：Truncate只能删除管理表，不能删除外部表中数据

```sql
truncate table student;
```



## 5.hive查询语法

### 5.1 select

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
FROM table_reference
[WHERE where_condition]
[GROUP BY col_list [HAVING condition]]
[CLUSTER BY col_list
| [DISTRIBUTE BY col_list] [SORT BY| ORDER BY col_list]
]
[LIMIT number]
```

1. order by会对输入做一个全局排序，因此只有一个Reducer，会导致当输入规模较大时，需要较长的计算时间
2. sort by不是全局排序，其在数据进入reducer前完成排序。因此，如果用sort by进行排序，并且设置mapred.reduce.task>1，则sort by只保证每个reducer的输出有序，不保证全局有序
3. disribute by(字段)根据指定的字段将数据分到不同的reducer，且分发算法是hash散列。
4. cluster by(字段)除了具有distribute by功能外，还会对该字段进行排序

因此，如果distribute和sort字段都是同一个时，此时，cluster by = distribute by + sort by

**注意：**

（1）SQL 语言**大小写不敏感。** 

（2）SQL 可以写在一行或者多行

（3）**关键字不能被缩写也不能分行**

（4）各子句一般要分行写。

（5）使用缩进提高语句的可读性。

### 5.2 查询语法

**全表查询**

```sql
select * from score;
```

**选择特定列**

```sql
select s_id,c_id from score;
```

**列别名**

1. 重命名一个列
2. 便于计算
3. 紧跟列名，也可以在列名和别名之间加入关键字as

```sql
select s_id as myid,c_id from score;
```

### 5.3 常用函数

1. 求总行数（count） 

   ```sql
   select count(1) from score;
   ```

2. 求分数的最大值（max）

   ```sql
   select max(s_score) from score;
   ```

3. 求分数最小值（min)

   ```sql
   select min(s_score) from score;
   ```

4. 求分数的总和（sum）

   ```sql
   select sum(s_score) from score;
   ```

5. 求分数的平均值

   ```sql
   select  avg(s_score) from score;
   ```

### 5.4 limit语句

典型的查询会返回多行数据。limit子句用于限制返回的行数

```sql
select * from score limit 3;
```

### 5.5where语句

1. 使用where子句，将不满足条件的行过滤掉

2. where子句紧随from子句

3. 案例操作：查询分数大于60的数据

   ```sql
   select * from score where s_score > 60;
   ```

**比较运算符**

下面表中描述了谓词操作符，这些操作符同样可以用于JOIN…ON和HAVING语句中。

| 操作符                  | 支持的数据类型 | 描述                                                         |
| ----------------------- | -------------- | ------------------------------------------------------------ |
| A=B                     | 基本数据类型   | 如果A等于B则返回TRUE，反之返回FALSE                          |
| A<=>B                   | 基本数据类型   | 如果A和B都为NULL，则返回TRUE，其他的和等号（=）操作符的结果一致，如果任一为NULL则结果为NULL |
| A<>B, A!=B              | 基本数据类型   | A或者B为NULL则返回NULL；如果A不等于B，则返回TRUE，反之返回FALSE |
| A<B                     | 基本数据类型   | A或者B为NULL，则返回NULL；如果A小于B，则返回TRUE，反之返回FALSE |
| A<=B                    | 基本数据类型   | A或者B为NULL，则返回NULL；如果A小于等于B，则返回TRUE，反之返回FALSE |
| A>B                     | 基本数据类型   | A或者B为NULL，则返回NULL；如果A大于B，则返回TRUE，反之返回FALSE |
| A>=B                    | 基本数据类型   | A或者B为NULL，则返回NULL；如果A大于等于B，则返回TRUE，反之返回FALSE |
| A [NOT] BETWEEN B AND C | 基本数据类型   | 如果A，B或者C任一为NULL，则结果为NULL。如果A的值大于等于B而且小于或等于C，则结果为TRUE，反之为FALSE。如果使用NOT关键字则可达到相反的效果。 |
| A IS NULL               | 所有数据类型   | 如果A等于NULL，则返回TRUE，反之返回FALSE                     |
| A IS NOT NULL           | 所有数据类型   | 如果A不等于NULL，则返回TRUE，反之返回FALSE                   |
| IN(数值1, 数值2)        | 所有数据类型   | 使用 IN运算显示列表中的值                                    |
| A [NOT] LIKE B          | STRING 类型    | B是一个SQL下的简单正则表达式，如果A与其匹配的话，则返回TRUE；反之返回FALSE。B的表达式说明如下：‘x%’表示A必须以字母‘x’开头，‘%x’表示A必须以字母’x’结尾，而‘%x%’表示A包含有字母’x’,可以位于开头，结尾或者字符串中间。如果使用NOT关键字则可达到相反的效果。 |
| A RLIKE B, A REGEXP B   | STRING 类型    | B是一个正则表达式，如果A与其匹配，则返回TRUE；反之返回FALSE。匹配使用的是JDK中的正则表达式接口实现的，因为正则也依据其中的规则。例如，正则表达式必须和整个字符串A相匹配，而不是只需与其字符串匹配。 |

- 查询分数等于80的所有数据

  ```sql
  select * from score where s_score = 80;
  ```

- 查询分数在80到100的所有数据

  ```sql
  select * from score where s_score between 80 and 100;
  ```

- 查询成绩为空的所有数据

  ```sql
  select * from score where s_score is null;
  ```

- 查询成绩是80和90的数据

  ```sql
  select * from score where s_score in(80,90);
  ```

### 5.6 like 和rlike

1. 使用like运算选择类似的值
2. 选择条件可以包含字符或数字

```
% 代表零个或多个字符（任意个字符）
_ 代表一个字符
```

1. rlike子句是hive中这个功能的一个扩展，其实可用通过Java的正则表达式这个更强大的语言来自指定匹配条件

2. 案例实操：

   - 查找以8开头的所有成绩

     ```sql
     select * from score where s_score like '8%';
     ```

   - 查找第二个数值为9的所有成绩数据

     ```sql
     select * from score where s_score like '_9%';
     ```

   - 查找s_id中含1的数据

     ```sql
     select  * from score where s_id rlike '[1]'; # like '%1%'
     ```

### 5.7 逻辑运算符

| 操作符 | 含义   |
| ------ | ------ |
| AND    | 逻辑并 |
| OR     | 逻辑或 |
| NOT    | 逻辑否 |

1. 查询成绩大于80，并且s_id是01的数据

   ```sql
   select * from score where s_score > 80 and s_id = '01';
   ```

2. 查询成绩大于80，或s_id是01的数据

   ```sql
   select * from score where s_id > 80 or s_id = '01';
   ```

3. 查询s_id不是01和02的学生

   ```sql
   select * from score where s_id not in('01','02');
   ```

### 5.8 分组

1. group by分组

   group by 语句通常会和聚合函数一起使用，按照一个或多个队列结果进行分组，然后对每个组执行聚合操作。

   - 计算每个学生的平均分数

     ```sql
     select s_id,avg(s_score) from score group by s_id;
     ```

   - 计算每个学生最高成绩

     ```sql
     select s_id,max(s_score) from score group by s_id;
     ```

2. having语句

   1. having与where不同点：

      - where针对表中的列发挥作用，查询数据；having针对查询结果中的列发挥作用，筛选数据
      - where后面不能写分组函数，而having后面可以使用分组函数
      - having值用于group by 分组统计语句

   2. 案例实操：

      - 求每个学生的平均分数

        ```sql
        select s_id,ang(s_score) from score group by s_id;
        ```

      - 求每个学生平均分数大于85的人

        ```sql
        select s_id,avg(s_score) as avgscore from score group by s_id having avgscore > 85;
        ```

### 5.9 join语句

1. 等值join

   hive支持通常的sql join语句，但是只支持等值连接，不支持非等值连接

   案例操作：查询分数对应的姓名

   ```sql
   select s.s_id,s.s_score,stu.s_name,stu.s_birth from score as s join student as stu on s.s_id = stu.s_id;
   ```

2. 表的别名

   - 好处

     - 使用别名可以简化查询
     - 使用表名前缀可以提高执行效率

   - 案例实操：

     合并老师与课程表

     ```sql
     select  * from teacher t join course on t.t_id = c.t_id;
     ```

3. 内连接

   内连接：只有进行连接 的两个表中都存在与连接条件相匹配的数据才会被保留下来

   ```sql
   select * from teacher t inner join course c on t.t_id = c.t_id;
   ```

4. 左外连接

   左外连接：join操作符左边表中符合where子句的所有记录都会被 返回。查询老师对应的课程

   ```sql
   select * from teacher t elft jion course c on t.t_id = c.t_id;
   ```

5. 右外连接

   右外连接：join操作符右边表中符合where子句的所有记录都将会被返回

   ```sql
   select * from teacher t right join course c on t.t_id = c.t_id;
   ```

6. 多表连接

   注意：连接n个表，至少需要n-1个连接条件。例如：连接三个表，至少需要两个连接条件

   多表连接查询，查询老师对应的课程，以及对应的分数，对应的学生

   ```sql
   select * from teacher t
   left join course c on t.t_id = c.t_id
   left join score s on s.c_id = c.c_id 
   left join student stu on s.s_id = stu.s_id;
   ```

   大多数情况下，hive会对每对join连接对象启动一个MapReduce任务。本例中会首先启动一个MapReduce job对表teacher和表course进行连接操作，然后会在启动一个MapReduce job 将第一个MapReduce job的输出和表score进行连接操作
   
   **注意：**为什么不是表d和表l先进行连接操作呢？这是因为Hive总是按照从左到右的顺序执行的。
   
7. 连接谓词中不支持or

   ```sql
   hive (default)> select e.empno, e.ename, d.deptno from emp e join dept d on e.deptno = d.deptno or e.ename=d.ename;   错误的
   ```

   

### 5.10 排序

1. 全局排序

   order by 全局排序，一个Reduce

   1. 使用order by子句排序（ASC）升序（默认），DESC：降序

   2. order by 子句在select 语句的结尾

   3. 案例实操：

      - 查询学生的成绩，并按照分数降序排列

        ```sql
        select * from student s lift join score sco on s.s_id = sco.s_id order by sco.s_score desc;
        ```

      - 查询学生的成绩，并按照分数升序排列

        ```sql
        select * from student s left join score sco on s.s_id = sco.s_id order by sco.s_score asc;
        ```

2. 按照别名排序

   按照分数的平均值排序

   ```sql
   select s_id,avg(s_score) avg from score group by s_id order by avg;
   ```

3. 多个列排序

   按照学生id和平均成绩进行排序

   ```sql
   select * from,avg(s_score) avg from score group by s_id order by s_id,avg;
   ```

4. 每个MapReduce内部排序（sort by）局部排序

   Sort By：每个MapReduce内部进行排序，分区规则按照key的hash来运算，（区内排序）对全局结果集来说不是排序。

   1. 设置reduce个数

      ```sql
      set mapreduce.job.reduces=3;
      ```

   2. 查看设置reduce个数

      ```sql
      set mapreduce.job.reduces;
      ```

   3. 查询成绩按照成绩降序排列

      ```sql
      select * from score sort by s_score;
      ```

   4. 将查询结果导入到文件中（按照成绩降序排列）

      ```sql
      insert overwrite local directory '/opt/moudule/datas/sort' select * from score sort by s_score;
      ```

5. 分区排序

   distribute by ：类似MR中partition，进行分区，结合sort by 使用

   **注意：hive要求hdfs要求distribute by语句要写在sort by 语句之前**

   对distribute by 进行测试，一定要分配多reduce进行处理，否则无法看到distribute by的效果

   案例实操：先按照学生id进行分区，再按照学生成绩进行排序。

   1. 设置reduce的个数，将我们对应的s_id划分到对应的reduce当中去

      ```sql
      set mapreduce.job.reduces= 7;
      ```

   2. 通过distribute by 进行数据分区

      ```sql
      insert overwrite local directory '/opt/module/datas/sort' select * from score distribute by s_id sort by s_score;
      ```

6. cluster by

   当distribute by 和sort by 字段相同时，可以使用cluster by的方式

   cluster by 除了具有distribute by的功能外还兼具sort by的功能。但是排序**只能是倒序排序**，不能指定排序规则为ASC或者DESC。

   以下两种写法等价：

   ```sql
   select * from score cluster by s_id;
   select * from score distribute by s_id sort by s_id;
   ```



## 6.Hive数据类型

### 6.1 基本数据类型

| Hive数据类型 | Java数据类型 | 长度                                                 | 例子                                 |
| ------------ | ------------ | ---------------------------------------------------- | ------------------------------------ |
| TINYINT      | byte         | 1byte有符号整数                                      | 20                                   |
| SMALINT      | short        | 2byte有符号整数                                      | 20                                   |
| INT          | int          | 4byte有符号整数                                      | 20                                   |
| BIGINT       | long         | 8byte有符号整数                                      | 20                                   |
| BOOLEAN      | boolean      | 布尔类型，true或者false                              | TRUE  FALSE                          |
| FLOAT        | float        | 单精度浮点数                                         | 3.14159                              |
| DOUBLE       | double       | 双精度浮点数                                         | 3.14159                              |
| STRING       | string       | 字符系列。可以指定字符集。可以使用单引号或者双引号。 | ‘now is the time’ “for all good men” |
| TIMESTAMP    |              | 时间类型                                             |                                      |
| BINARY       |              | 字节数组                                             |                                      |

对于Hive的String类型相当于数据库的varchar类型，该类型是一个可变的字符串，不过它不能 其中最多能存储多少个字符，理论上它可以存储2GB的字符数。

### 6.2 集合数据类型

| 数据类型 | 描述                                                         | 语法示例 |
| -------- | ------------------------------------------------------------ | -------- |
| STRUCT   | 和c语言中的struct类似，都可以通过“点”符号访问元素内容。例如，如果某个列的数据类型是STRUCT{first STRING, last STRING},那么第1个元素可以通过字段.first来引用。 | struct() |
| MAP      | MAP是一组键-值对元组集合，使用数组表示法可以访问数据。例如，如果某个列的数据类型是MAP，其中键->值对是’first’->’John’和’last’->’Doe’，那么可以通过字段名[‘last’]获取最后一个元素 | map()    |
| ARRAY    | 数组是一组具有相同类型和名称的变量的集合。这些变量称为数组的元素，每个数组元素都有一个编号，编号从零开始。例如，数组值为[‘John’, ‘Doe’]，那么第2个元素可以通过数组名[1]进行引用。 | Array()  |

Hive有三种复杂数据类型ARRAY、MAP 和 STRUCT。ARRAY和MAP与Java中的Array和Map类似，而STRUCT与C语言中的Struct类似，它封装了一个命名字段集合，复杂数据类型允许任意层次的嵌套。

**案例实操：**

1. 假设某表有如下一行，我们采用JSON格式来表示其数据结构。在hive下访问的格式为：

   ```json
   {
       "name": "songsong",
       "friends": ["bingbing" , "lili"] ,       //列表Array, 
       "children": {                      //键值Map,
           "xiao song": 18 ,
           "xiaoxiao song": 19
       }
       "address": {                      //结构Struct,
           "street": "hai dian qu" ,
           "city": "beijing" 
       }
   }
   ```

2. 基于上述数据结构，我们在hive里创建对应的表，并导入数据

   创建本地测试文件test.txt

   ```
   songsong,bingbing_lili,xiao song:18_xiaoxiao song:19,hai dian qu_beijing
   yangyang,caicai_susu,xiao yang:18_xiaoxiao yang:19,chao yang_beijing
   ```

   注意，MAP，STRUCT和ARRAY里的元素间关系都可以用同一个字符表示，这里用“_”。

3. hive上创建测试表test

   ```sql
   create table test(
   name string,
   friends array<string>,
   children map<string, int>,
   address struct<street:string, city:string>
   )
   row format delimited fields terminated by ','
   collection items terminated by '_'
   map keys terminated by ':'
   lines terminated by '\n';
   ```

   字段解释：

   - row format delimited fields terminated by ','  -- 列分隔符

   - collection items terminated by '_'  --MAP STRUCT 和 ARRAY 的分隔符(数据分割符号)

   - map keys terminated by ':'				-- MAP中的key与value的分隔符

   - lines terminated by '\n';					-- 行分隔符

4. 导入文本数据到测试表

   ```
   load data local inpath '/opt/module/datas/test.txt' into table test;
   ```

5. 访问三种集合列里的数据，以下分别是ARRAY,MAP,STRUCT的访问方式

   ```
   hive (myhive)> select friends[1],children['xiao song'],address.city from test where name="songsong";
   OK
   _c0	_c1	city
   lili	18	beijing
   Time taken: 1.847 seconds, Fetched: 1 row(s)
   ```

   如果起别名：`select friends[1] as pengyou from test where name="songsong";`

### 6.3 类型转换

hive的原子数据类型是可以进行隐式转换的，类似于java的类型转化，例如某表达式使用int类型，tinyint会自动转化为int类型，但是hive不会进行反向转化，例如，某表达式使用tinyint类型，int类型不会自动转化为tinyint类型，他会返回错误，，除非使用cast操作

1. 隐式类型转换规则如下：
   - 任何整数类型都可以隐式转化为一个范围更广的类型，如tinyint转化为int，int可以转化为bigint
   - 所有整数类型，float和strint类型都可以隐士转化为double
   - tinyint，smallint，int都可以转化为float
   - boolean类型不可以转化为任何其他的类型
2. 可以使用cast操作显示进行数据类型转换，例如CAST('1' AS INT)将把字符串1转化成整数1；如果强制类型转换失败，如执行CAST('X' AS INT) ，表达式返回空值null

## 7.hive shell 参数

### 7.1 hive 命令行

语法结构

```
bin/hive [-hiveconf x=y]* [<-i filename>]* [<-f filename>|<-e query-string>] [-S]
```

说明：

1. -i 从文件初始化HQL
2. -e 从命令行执行指定的HQL
3. -f 执行HQL脚本
4. -v 输出指定的HQL语句到控制台
5. -p  connect to Hive Server on port number
6.  -hiveconf x=y Use this to set hive/hadoop configuration variables. 设置hive运行时候的参数配置

### 7.2 hive参数配置方式

开发hive应用时，不可避免地需要设定hive的参数。设定hive的参数可以调优HQL代码的执行效率，或帮助定位问题。

**对于一半参数，有以下三种设定方式：**

- 配置文件
- 命令行参数
- 参数声明

**配置文件：**

- 默认配置文件：hive-default.xml 

- 用户自定义配置文件：hive-site.xml

用户自定义配置会覆盖默认配置。另外，Hive也会读入Hadoop的配置，因为Hive是作为Hadoop的客户端启动的，Hive的配置会覆盖Hadoop的配置。配置文件的设定对本机启动的所有Hive进程都有效。

配置文件的设定对本机启动的所有hive进程都有效

**命令行参数：**启动hive（客户端或server方式）时，可以在命令行添加-hiveconf param=value来设定参数，例如：

```
bin/hive -hiveconf hive.root.logger=INFO,console
```

**这一设定对本次启动的seesion（对于server方式启动，则是所有请求的seesions）有效**

**参数声明：**可以在HQL中使用set关键字设定参数，例如：

```
set mapred.reduce.tasks=100;
```

这一设定的作用域也是seesion级的。

上述三种设定方式的优先级依次递增。即参数声明覆盖命令行参数，命令行参数覆盖配置文件设定。注意某些系统级的参数，例如log4j相关的设定，必须用前两种方式设定，因为那些参数的读取在seesion建立以前已经完成了

**参数声明 > 命令行参数 > 配置文件参数（hive）**

## 8.Hive函数

### 8.1内置函数

内容较多，见《hive官方文档》

https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF

1. 查看系统自带的函数

   ```
   show functions;
   ```

2. 显示自带函数的用法

   ```
   desc function upper;
   ```

3. 详细显示自带的函数的用法

   ```
   desc function extend upper;
   ```

4. 常用内置函数

   ```sql
   # 字符串连接函数：concat
   select concat('asa','assd','asdas');
   # 带分隔符字符串连接函数：concat_ws
   select concat_ws(',','asd','asdf','fdf');
   # cast类型转换
   select cast(1.5 as int);
   # get_json_object(json 解析函数，用来处理json，必须是json格式)
    select get_json_object('{"name":"jack","age":"20"}','$.name'
   # URL解析函数
     select parse_url('http://facebook.com/path1/p.php?k1=v1&k2=v2#Ref1', 'HOST');
   # explode:把map集合中每个键值对或数组中的每个元素都单独生成一行的形式
   ```

### 8.2 自定义函数

1. hive自带了一些函数，比如：max/min等，当hive提供的内置函数无法满足你的业务处理需要时，此时就可以考虑视同用户自定义函数（UDF）

2. 根据用户自定义函数类别分为以下三种：

   1. UDF：一进一出
   2. UDAF：聚集函数，多进一出，类似于：count/max/min
   3. UDTF：一进多出，，如：lateral view explore()

3. 编程步骤：

   1. 继承org.apache.hadoop.hive.ql.UDF

   2. 需要实现evaluate函数；evaluate函数支持重载

   3. 在hive的命令行窗口创建函数

      - 添加jar

        ```
        add jar linux_jar_path
        ```

      - 创建function

        ```
        create [temporary] function [dbname.]function_name AS class_name;
        ```

   4. 在hive的命令行窗口删除函数

      ```
      Drop [temporary] function [if exists] [dbname.]function_name;
      ```

4. 注意事项：

   1. UDF必须要有返回类型，可以返回null，但是返回类型不能为void
   2. UDF中常用Text/LongWritable等类型，不推荐使用java类型

5. 自定义UDF函数开发案例

   1. 创建Maven工程，并导入pom依赖

      ```xml
   <?xml version="1.0" encoding="UTF-8"?>
      <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
          <modelVersion>4.0.0</modelVersion>
      
          <groupId>hive</groupId>
          <artifactId>hive2.3.4</artifactId>
          <version>1.0-SNAPSHOT</version>
      
      
          <packaging>jar</packaging>
          <dependencies>
              <!-- https://mvnrepository.com/artifact/org.apache.hive/hive-exec -->
              <dependency>
                  <groupId>org.apache.hive</groupId>
                  <artifactId>hive-exec</artifactId>
                  <version>2.3.4</version>
              </dependency>
      
      
              <!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-common -->
              <dependency>
                  <groupId>org.apache.hadoop</groupId>
                  <artifactId>hadoop-common</artifactId>
                  <version>2.8.4</version>
              </dependency>
          </dependencies>
      
      
          <build>
              <plugins>
                  <plugin>
                   <groupId>org.apache.maven.plugins</groupId>
                      <artifactId>maven-compiler-plugin</artifactId>
                   <version>2.5.1</version>
                      <configuration>
                       <encoding>UTF-8</encoding>
                          <source>1.8</source>
                          <target>1.8</target>
                      </configuration>
                  </plugin>
                  <!-- 编译插件 -->
                  <plugin>
                      <groupId>org.apache.maven.plugins</groupId>
                      <artifactId>maven-compiler-plugin</artifactId>
                      <configuration>
                          <source>1.8</source>
                          <target>1.8</target>
                          <encoding>utf-8</encoding>
                      </configuration>
                  </plugin>
              </plugins>
          </build>
   </project>
      ```

   4. 创建一个类

      ```java
      package myudf;
      import org.apache.hadoop.hive.ql.exec.UDF;
   
      public class udf extends UDF {
   
      	public String evaluate (final String s) {
      		
      		if (s == null) {
   			return null;
      		}
   		
      		return s.toString().toLowerCase();
      	}
      }
      ```
   ```
   
   5. 打成jar包上传到服务器/opt/module/jars/udf.jar
   
   6. 将jar包添加到hive的classpath
   
   ```
      hive (default)> add jar /opt/module/jars/udf.jar;
      ```
   
   7. 创建临时函数与开发好的java class关联**(全类名)**
   
      ```
      hive (default)> create temporary function udf_lower as "myudf.udf";
      ```
   
   8. 即可在hql中使用自定义的函数
   
      ```
      hive (default)> select ename, udf_lower(ename) from student;
      ```
   
   
   原表内容：
   
      ```
   hive (myhive)> select * from student2;
   OK
   student2.id	student2.name
   1	aaa
   1	AAA
   Time taken: 0.118 seconds, Fetched: 2 row(s)
   ```
   
   使用自定义udf函数查询后
   
   ```
   hive (myhive)> select udf_lower(name) from student2;
   OK
   _c0
   aaa
   aaa
   Time taken: 0.329 seconds, Fetched: 2 row(s)
   ```
   
   
   ```


## 9.Hive的数据压缩

**概念：**在实际工作当中，hive当中处理的数据，一般都需要经过压缩，前期在学习hadoop的时候，已经配置过hadoop的压缩，hive也是一样的可以使用压缩来节省我们的MR处理的网络带宽

### 9.1 MR支持的压缩编码

| 压缩格式 | 工具  | 算法    | 文件扩展名 | 是否可切分 |
| -------- | ----- | ------- | ---------- | ---------- |
| DEFAULT  | 无    | DEFAULT | .deflate   | 否         |
| Gzip     | gzip  | DEFAULT | .gz        | 否         |
| bzip2    | bzip2 | bzip2   | .bz2       | 是         |
| LZO      | lzop  | LZO     | .lzo       | 否         |
| LZ4      | 无    | LZ4     | .lz4       | 否         |
| Snappy   | 无    | Snappy  | .snappy    | 否         |

为了支持多种压缩/解压缩算法，Hadoop引入了编码/解码器，如下所示：

| 压缩格式 | 对应的编码/解码器                          |
| -------- | ------------------------------------------ |
| DEFLATE  | org.apache.hadoop.io.compress.DefaultCodec |
| gzip     | org.apache.hadoop.io.compress.GzipCodec    |
| bzip2    | org.apache.hadoop.io.compress.BZip2Codec   |
| LZO      | com.hadoop.compression.lzo.LzopCodec       |
| LZ4      | org.apache.hadoop.io.compress.Lz4Codec     |
| Snappy   | org.apache.hadoop.io.compress.SnappyCodec  |

压缩性能的比较

| 压缩算法 | 原始文件大小 | 压缩文件大小 | 压缩速度 | 解压速度 |
| -------- | ------------ | ------------ | -------- | -------- |
| gzip     | 8.3GB        | 1.8GB        | 17.5MB/s | 58MB/s   |
| bzip2    | 8.3GB        | 1.1GB        | 2.4MB/s  | 9.5MB/s  |
| LZO      | 8.3GB        | 2.9GB        | 49.3MB/s | 74.6MB/s |

在64位模式下的Core i7处理器的单核上，Snappy的压缩速度约为250MB /秒或更高，而解压缩的速度约为500 MB /秒。

### 9.2 压缩配置参数

要在Hadoop中启用压缩，可以配置如下参数（mapred-site.xml文件中）：

| 参数                                              | 默认值                                                       | 阶段        | 建议                                         |
| ------------------------------------------------- | ------------------------------------------------------------ | ----------- | -------------------------------------------- |
| io.compression.codecs   （在core-site.xml中配置） | org.apache.hadoop.io.compress.DefaultCodec, org.apache.hadoop.io.compress.GzipCodec, org.apache.hadoop.io.compress.BZip2Codec,org.apache.hadoop.io.compress.Lz4Codec | 输入压缩    | Hadoop使用文件扩展名判断是否支持某种编解码器 |
| mapreduce.map.output.compress                     | false                                                        | mapper输出  | 这个参数设为true启用压缩                     |
| mapreduce.map.output.compress.codec               | org.apache.hadoop.io.compress.DefaultCodec                   | mapper输出  | 使用LZO、LZ4或snappy编解码器在此阶段压缩数据 |
| mapreduce.output.fileoutputformat.compress        | false                                                        | reducer输出 | 这个参数设为true启用压缩                     |
| mapreduce.output.fileoutputformat.compress.codec  | org.apache.hadoop.io.compress. DefaultCodec                  | reducer输出 | 使用标准工具或者编解码器，如gzip和bzip2      |
| mapreduce.output.fileoutputformat.compress.type   | RECORD                                                       | reducer输出 | SequenceFile输出使用的压缩类型：NONE和BLOCK  |

### 9.3 开启Map输出阶段压缩

开启map输出阶段压缩可以减少job中map和Reduce task间的数据压缩。具体配置如下：

**案例实操：**

1. 开启hive中间传输数据压缩功能，默认为false

   ```
   set hive.exec.compress.intermediate=true;
   ```

2. 开启mapreduce中map输压缩方式，默认为false

   ```
   set mapreduce.map.output.compress=true;
   ```

3. 设置mapreduce中map输出数据的压缩方式，默认为false

   ```
   set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec;
   ```

4. 执行查询语句

   ```sql
   set count(1) from score;
   ```

### 9.4 开启reduce输出阶段压缩

当hive将输出写入到表中时，输出内容同样可以进行压缩。属性`hive.exec.compress.output`控制着这个功能。用户可能需要保持默认设置文件中的默认值false，这样默认的输出就是非压缩的纯文本文件了。用户可以通过在查询语句或执行脚本中设置这个值为true，来开启输出结果压缩功能。

**案例实操：**

1. 开启hive最终输出数据压缩功能，默认为false

   ```
   set hive.exec.compress.output=true;
   ```

2. 开启mapreduce最终输出数据压缩，默认为false

   ```
   set mapreduce.output.fileputputformat.compress=true;
   ```

3. 设置mapreduce最终数据输出压缩方式

   ```
   set mapreduce.output.fileoutputformat.compress.codec=org.apache.hadoop.io.compress.SnappyCodec;
   ```

4. 设置mapreduce最终输出压缩为块压缩

   ```
   set mapreduce.output.fileoutputformat.compress.type=BLOCK;
   ```

5. 测试一下输出结果是不是压缩文件

   ```sql
   insert overwrite local directory '/opt/module/datas/snappy' select * from score distribute by s_id sort by s_id desc;
   ```

## 10.hive的数据存储格式

hive支持的存储数据的格式主要有：TEXTFILE（行式存储） 、SEQUENCEFILE(行式存储)、ORC（列式存储）、PARQUET（列式存储）。

行储存：textFile 、 sequencefile 

列储存：orc 、parquet

### 10.1列式存储和行式存储

![](img\hive\wps1.jpg)

上图左边为逻辑表，右边第一个为行式存储，第二个为列式存储。

**行存储的特点：**

查询满足条件的一整行数据的时候，列存储则需要去每个聚集的字段找到对应的每个列的值，行存储只需要找到其中一个值，其余的值都在相邻的地方，所以此时行存储查询的速度更快。

**列存储的特点：**

因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算法

> TEXTFILE和SEQUENCEFILE的存储格式都是基于行存储的；
> ORC和PARQUET是基于列式存储的。

### 10.2 常用数据存储格式

**TEXTFILE格式：**

默认格式，数据不做压缩，磁盘开销大，数据解析开销大。可结合Gzip，Bzip2使用。

**ORC格式：**

> Orc (Optimized Row Columnar)是hive 0.11版里引入的新的存储格式。

可以看到每个ORC文件由1个或多个stripe组成，每个stripe为250MB大小，每个stripe里面三部分组成，分别是Index Data,Row Data,Stripe Footer：

- indexData ：某些列的索引数据，一个轻量级的index，**默认是每隔1W行做一个索引。**这里做的索引应该只是记录某行的各字段在Row Data中的offset。
- rowData：存的是具体的数据，先取部分行，然后对这些行按列进行存储。对每个列进行了编码，分成多个Stream来存储。
- stripFooter：存的是各个Stream的类型，长度等信息。

![](img\hive\wps2.jpg)

每个文件有一个File Footer，这里面存的是每个Stripe的行数，每个Column的数据类型信息等；每个文件的尾部是一个PostScript，这里面记录了整个文件的压缩类型以及FileFooter的长度信息等。在读取文件时，会seek到文件尾部读PostScript，从里面解析到File Footer长度，再读FileFooter，从里面解析到各个Stripe信息，再读各个Stripe，即从后往前读。

**PARQUET格式**

> parquet是面向分析型业务的列式存储格式，由Twitter和Cloudera合作开发

parquet文件是以二进制方式存储的，所以是不可以直接读取的，文件中包括该文件数据和元数据，因此parquet格式文件是自解析的。

通常情况下，在存储parquet数据的时候会按照Block大小设置行组的大小，由于一般情况下每一个MapReduce任务处理数据的最小单位是一个Block，这样可以把每一个行组由一个Mapper任务处理，增大任务执行的并行度。

parquet文件的格式如下图所示：

![](img\hive\wps3.jpg)

上图展示了一个Parquet文件的内容，一个文件中可以存储多个行组，文件的首位都是该文件的Magic Code，用于校验它是否是一个Parquet文件，Footer length记录了文件元数据的大小，通过该值和文件长度可以计算出元数据的偏移量，文件的元数据中包括每一个行组的元数据信息和该文件存储数据的Schema信息。除了文件中每一个行组的元数据，每一页的开始都会存储该页的元数据，在Parquet中，有三种类型的页：数据页、字典页和索引页。数据页用于存储当前行组中该列的值，字典页存储该列值的编码字典，每一个列块中最多包含一个字典页，索引页用来存储当前行组下该列的索引，目前Parquet中还不支持索引页。

## 11.文件存储格式与数据压缩结合

### 11.1 压缩比和查询速度对比

#### 11.1.1TextFile

1. 创建表，存储数据格式为TEXTFILE

   ```sql
   create table log_text (
   track_time string,
   url string,
   session_id string,
   referer string,
   ip string,
   end_user_id string,
   city_id string
   )
   ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
   STORED AS TEXTFILE ;
   ```

2. 向表中加载数据

   ```sql
   load data local inpath '/opt/module/datas/log.data' into table log_text;
   ```

3. 查看表中数据大小

   ```
   dfs -du -h /user/hive/warehouse/myhive.db/log_text;
   ```
   
   ```
   [root@bigdata111 datas]# hadoop dfs -du -h /user/hive/warehouse/myhive.db/log_text;
   DEPRECATED: Use of this script to execute hdfs command is deprecated.
   Instead use the hdfs command for it.
   
   18.1 M  /user/hive/warehouse/myhive.db/log_text/log.data
   ```
   
   注：log.data原数据大小为18.1MB



#### 11.1.2 ORC

1. 创建表，存储格式为ORC

   ```sql
   create table log_orc(
   track_time string,
   url string,
   session_id string,
   referer string,
   ip string,
   end_user_id string,
   city_id string
   )
   ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
   STORED AS orc ;
   ```

2. 向表中加载数据

   ```sql
   insert into table log_orc select * from log_text;
   ```

3. 查看表中数据大小

   ```
   dfs -du -h /user/hive/warehouse/myhive.db/log_orc;
   ```
   
   ```
   [root@bigdata111 datas]# hadoop dfs -du -h /user/hive/warehouse/myhive.db/log_orc;
   DEPRECATED: Use of this script to execute hdfs command is deprecated.
   Instead use the hdfs command for it.
   
   2.8 M  /user/hive/warehouse/myhive.db/log_orc/000000_0
   ```
   
   

#### 11.1.3 Parquet

1. 创建表，存储数据格式为parquet

   ```sql
   create table log_parquet(
   track_time string,
   url string,
   session_id string,
   referer string,
   ip string,
   end_user_id string,
   city_id string
   )
   ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
   STORED AS PARQUET ;
   ```

2. 向表中加载数据

   ```sql
   insert into table log_parquet select * from log_text ;
   ```

3. 查看表中数据大小

   ```
   dfs -du -h /user/hive/warehouse/myhive.db/log_parquet;
   ```
   
   ```
   hive (myhive)> dfs -du -h  /user/hive/warehouse/myhive.db/log_parquet;
   13.1 M  /user/hive/warehouse/myhive.db/log_parquet/000000_0
   ```
   
   

#### 11.1.4 存储文件的压缩比总结：

ORC > Parquet > textFile

#### 11.1.5 存储文件的查询速度测试

1. TextFile

   ```
   Stage-Stage-1: Map: 1  Reduce: 1   Cumulative CPU: 3.05 sec   HDFS Read: 19023315 HDFS Write: 106 SUCCESS
   Total MapReduce CPU Time Spent: 3 seconds 50 msec
   OK
   _c0
   100000
   Time taken: 22.722 seconds, Fetched: 1 row(s)
   ```

2. ORC

   ```
   hive (default)> select count(*) from log_orc;
   Time taken: 20.867 seconds, Fetched: 1 row(s)
   ```

3. Parquet

   ```
   hive (default)> select count(*) from log_parquet;
   Time taken: 22.922 seconds, Fetched: 1 row(s)
   ```

#### 11.1.6 存储文件的查询速度总结

ORC > TextFile > Parquet

### 11.2 ORC存储指定压缩方式

官网：https://cwiki.apache.org/confluence/display/Hive/LanguageManual+O

ORC存储方式的压缩：

| Key                      | Default    | Notes                                                        |
| ------------------------ | ---------- | ------------------------------------------------------------ |
| orc.compress             | `ZLIB`     | high level compression (one of NONE, ZLIB, SNAPPY)           |
| orc.compress.size        | 262,144    | number of bytes in each compression chunk                    |
| orc.stripe.size          | 67,108,864 | number of bytes in each stripe                               |
| orc.row.index.stride     | 10,000     | number of rows between index entries (must be >= 1000)       |
| orc.create.index         | true       | whether to create row indexes                                |
| orc.bloom.filter.columns | ""         | comma separated list of column names for which bloom filter should be created |
| orc.bloom.filter.fpp     | 0.05       | false positive probability for bloom filter (must >0.0 and <1.0) |

1. 创建一个非压缩的ORC存储方式

   1. 建表

      ```sql
      create table log_orc_none(
      track_time string,
      url string,
      session_id string,
      referer string,
      ip string,
      end_user_id string,
      city_id string
      )
      ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
      STORED AS orc tblproperties ("orc.compress"="NONE");
      ```

   2. 插入数据

      ```sql
      insert into table log_orc_none select * from log_text
      ```

   3. 查看插入后数据

      ```
      dfs -du -h /user/hive/warehouse/myhive.db/log_orc_none;
      ```
      
      ```
      hive (myhive)> dfs -du -h /user/hive/warehouse/myhive.db/log_orc_none;
      7.7 M  /user/hive/warehouse/myhive.db/log_orc_none/000000_0
      ```
      
      

2. 创建一个SNAPPT压缩的ORC存储方式

   1. 建表

      ```sql
      create table log_orc_snappy(
      track_time string,
      url string,
      session_id string,
      referer string,
      ip string,
      end_user_id string,
      city_id string
      )
      ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
      STORED AS orc tblproperties ("orc.compress"="SNAPPY");
      ```

   2. 插入数据

      ```sql
      insert into table log_orc_snappy select * from log_text ;
      ```

   3. 查看插入后数据

      ```
      dfs -du -h /user/hive/warehouse/myhive.db/log_orc_snappy ;
      ```
      
      ```
      hive (myhive)> dfs -du -h /user/hive/warehouse/myhive.db/log_orc_snappy ;
      3.7 M  /user/hive/warehouse/myhive.db/log_orc_snappy/000000_0
      ```
      
      

### 11.3存储方式和压缩总结

在实际的项目开发中，hive表的数据存储格式一般选择：ORC或Parquet。压缩方式一般选择Snappy

## 12.hive调优

### 12.1 Fetch抓取

Fetch抓取是指，**hive中对某些情况的 查询可以不必使用MapReduce计算。**例如：`select * from score;`在这种情况下，hive可以简单地读取score对应的存储目录下的文件，然后输出查询结果到控制台，通过设置`hive.fetch.task.conversion`参数，可以控制查询语句是否走MapReduce。

在hive-default.xml.template文件中hive.fetch.task.conversion默认是more，**老版本hive默认是minimal，该属性修改为more以后，在全局查找、字段查找、limit查找等都不走mapreduce。**

```xml
<property>
    <name>hive.fetch.task.conversion</name>
    <value>more</value>
    <description>
      Expects one of [none, minimal, more].
      Some select queries can be converted to single FETCH task minimizing latency.
      Currently the query should be single sourced not having any subquery and should not have
      any aggregations or distincts (which incurs RS), lateral views and joins.
      0. none : disable hive.fetch.task.conversion
      1. minimal : SELECT STAR, FILTER on partition columns, LIMIT only
      2. more  : SELECT, FILTER, LIMIT only (support TABLESAMPLE and virtual columns)
    </description>
  </property>
```

**案例实操：**

1. 把hive.fetch.task.conversion设置为none，然后执行查询语句，都会执行mapreduce程序

   ```sql
   set hive.fetch.task.conversion=none;
   
   select * from score;
   select s_score from score;
   select s_score from score limit 3;
   ```

2. 把hive.fetchtask.conversion设置为more，然后执行查询语句，如下查询方式都不会执行MapReduce程序。

   ```sql
   set hive.fetch.task.conversion=more;
   
   select * from score;
   select s_score from score;
   select s_score from score limit 3;
   ```

### 12.2 本地模式

大多数的Hadoop Job是需要Hadoop提供完整的可扩展性来处理大数据集的。不过，有时Hive的输入数据量是非常小的。在这种情况下，为查询触发执行任务时消耗可能会比实际job的执行时间要多的多。对于大多数这种情况，**hive可以通过本地模式在单台机器上处理所有的任务。对于小数据集，执行时间可以明显被缩短。**

用户可以通过设置hive.exec.mode.local.auto的值为true，来让hive在适当的时候启动这个优化。

**案例实操：**

1. 开启本地模式，并执行查询语句（快）(注意重启Hive)

   ```sql
   set hive.exec.mode.local.auto=true;
   select * from score cluster by s_id;
   ```

2. 关闭本地模式，并执行查询语句（慢）

   ```sql
   set hive.exec.mode.local.auto=false;
   select * from score cluster by s_id;
   ```

### 12.3 表的优化

#### 12.3.1 mapjoin

如果不指定MapJoin或者不符合MapJoin的条件，那么Hive解析器会将Join操作转换成Common Join，即：在Reduce阶段完成join。容易发生数据倾斜。可以用MapJoin把小表全部加载到内存在map端进行join，避免reducer处理。

**开启MapJoini参数设置：**

1. 设置自动选择MapJoin，默认为true

   ```sql
   set hive.auto.convert.join=true;
   ```

2. 大表小表的阈值设置（默认25M以下认为是小表）

   ```
   set hive.mapjoin.smalltable.filesieze=25123456;
   ```

**MapJoin工作机制**

案例操作：

1. 开启mapjoin功能

   ```sql
   set hive.auto.convert.join = true; 默认为true
   ```

2. 执行小表JOIN大表语句

   ```sql
   insert overwrite table jointable
   select b.id, b.time, b.uid, b.keyword, b.url_rank, b.click_num, b.click_url
   from smalltable s
   join bigtable  b
   on s.id = b.id;
   ```

   Time taken: 24.594 seconds

3. 执行大表JOIN小表语句

   ```sql
   insert overwrite table jointable
   select b.id, b.time, b.uid, b.keyword, b.url_rank, b.click_num, b.click_url
   from bigtable  b
   join smalltable  s
   on s.id = b.id;
   ```

   Time taken: 24.315 seconds

#### 12.3.2小表、大表Join

将key相对分散，并且数据量小的表放在join的左边，这样可以有效减少内存溢出错误发生的几率；再进一步，可以使用Group变小的维度表（1000条以下的记录条数）先进内存。在map端完成reduce（预聚合）。

**实际测试发现：新版的hive已经对小表JOIN大表和大表JOIN小表进行了优化。小表放在左边和右边已经没有明显区别。**

需求：测试大表JOIN小表和小表JOIN大表的效率

1. 建大表、小表和JOIN后表的语句

   ```sql
   // 创建大表
   create table bigtable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';
   // 创建小表
   create table smalltable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';
   // 创建join后表的语句
   create table jointable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';
   ```

2. 分别向大表和小表中导入数据

   ```sql
   hive (default)> load data local inpath '/opt/module/datas/bigtable' into table bigtable;
   hive (default)>load data local inpath '/opt/module/datas/smalltable' into table smalltable;
   ```

3. 关闭mapjoin功能（默认是打开的）

   ```
   set hive.auto.convert.join = false;
   ```

4. 执行小表JOIN大表语句

   ```
   insert overwrite table jointable
   select b.id, b.time, b.uid, b.keyword, b.url_rank, b.click_num, b.click_url
   from smalltable s
   left join bigtable  b
   on b.id = s.id;
   ```

   Time taken: 35.921 seconds

5. 执行大表JOIN小表语句

   ```
   insert overwrite table jointable
   select b.id, b.time, b.uid, b.keyword, b.url_rank, b.click_num, b.click_url
   from bigtable  b
   left join smalltable  s
   on s.id = b.id;
   ```

   Time taken: 34.196 seconds

#### 12.3.3 大表join小表

**空KEY过滤**

有时join超时是因为某些key对应的数据太多，而相同key对应的数据都会发送到相同的reducer上，从而导致内存不够。此时我们应该仔细分析这些异常的key，很多情况下，这些key对应的数据是异常数据，我们需要在SQL语句中进行过滤。例如key对应的字段为空，操作如下：

1. 配置历史服务器

   配置mapred-site.xml

   ```xml
   <property>
   <name>mapreduce.jobhistory.address</name>
   <value>bigdata111:10020</value>
   </property>
   <property>
       <name>mapreduce.jobhistory.webapp.address</name>
       <value>bigdata111:19888</value>
   </property>
   ```

   启动历史服务器

   sbin/mr-jobhistory-daemon.sh start historyserver

   查看jobhistory

   http://192.168.1.102:19888/jobhistory

2. 创建原始数据表、空id表、合并后数据表

   ```sql
   // 创建原始表
   create table ori(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';
   // 创建空id表
   create table nullidtable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';
   // 创建join后表的语句
   create table jointable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';
   ```

3. 分别加载原始数据和空id数据到对应表中

   ```sql
   hive (default)> load data local inpath '/opt/module/datas/ori' into table ori;
   hive (default)> load data local inpath '/opt/module/datas/nullid' into table nullidtable;
   ```

4. 测试不过滤空id

   ```sql
   hive (default)> insert overwrite table jointable 
   select n.* from nullidtable n left join ori o on n.id = o.id;
   Time taken: 42.038 seconds
   Time taken: 37.284 seconds
   Time taken: 97.281 seconds
   ```

5. 测试过滤空id

   ```
   hive (default)> insert overwrite table jointable 
   select n.* from (select * from nullidtable where id is not null ) n  left join ori o on n.id = o.id;
   Time taken: 31.725 seconds
   Time taken: 28.876 seconds
   ```

**空key转换**

有时虽然某个key为空对应的数据很多，但是相应的数据不是异常数据，必须要包含在join的结果中，此时我们可以表a中key为空的字段赋一个随机的值，使得数据随机均匀地分不到不同的reducer上。例如：

**案例实操：不随机分布空null值：**

1. 设置5个reduce个数

   ```
   set mapreduce.job.reduces = 5;
   ```

2. join两张表

   ```sql
   insert overwrite table jointable
   select n.* from nullidtable n left join ori b on n.id = b.id;
   ```

   结果：可以看出来，出现了数据倾斜，某些reducer的资源消耗远大于其他reducer。

   ![](img\hive\join.png)

**随机分布空null值**

1. 设置5个reduce个数

   ```
   set mapreduce.job.reduces = 5;
   ```

2. join两张表

   ```sql
   insert overwrite table jointable
   select n.* from nullidtable n full join ori o on 
   case when n.id is null then concat('hive', rand()) else n.id end = o.id;
   ```

   结果：可以看出来，消除了数据倾斜，负载均衡reducer的资源消耗

   ![](img\hive\join2.png)

### 12.4Group By

默认情况下，Map阶段同一key数据分发给一个reduce，当一个key数据过大时就倾斜了。并不是所有的聚合操作都需要在Reduce端完成，很多聚合操作都可以先在Map端进行部分聚合，最后在Reduce端得出最终结果。

**开启Map端聚合参数设置：**

1. 是否在map端进行聚合，默认为true；

   ```sql
   set hive.map.aggr = true;
   ```

2. 在map端进行聚合操作的条目数据

   ```sql
   set hive.groupby.mapaggr.checkinterval = 100000;
   ```

3. 有数据倾斜的时候进行负载均衡（默认是false）

   ```sql
   set hive.groupby.skewindata = true;
   ```

> **当选项设定为true，生成的查询计划会有两个MR Job。**
>
> 第一个MR Job中，map的输出结果会随机分不到reudce中，每个reduce做部分聚合操作，并输出结果，这样处理的结果是**相同的Group by key有可能被分发到不同的reduce中**，从而达到负载均衡的目的；
>
> 第二个MR Job再根据预处理的数据结果按照Group by key 分布到reduce中（这个过程可以保证相同的Group by key 被分布到同一个reduce中），最后完成最终的聚合操作。

### 12.5 Count（distinct）去重统计

数据量小的时候无所谓，数据量大的情况下，由于 count distinct操作需要用一个reduce task来完成，这一个reduce需要处理的数据量太大，就会导致整个job很难完成，一般count distinct使用先group by 在count的方式替换

```sql
select count(distinct s_id) from score;

select count(s_id) from (select id from score group by s_id) a;
```

虽然会多用一个job来完成，但在数据量大的情况下，这个绝对是值得的。

### 12.6 笛卡尔积

1. 笛卡尔集会在下面条件下产生:

   - 省略连接条件
   - 连接条件无效
   - 所有表中的所有行互相连接

2. 案例实操

   ```sql
   hive (default)> select empno, deptno from emp, dept;
   
   FAILED: SemanticException Column deptno Found in more than One Tables/Subqueries
   ```

尽量避免笛卡尔积，即避免join的时候不加on条件，或者无效的on条件，hive只能使用1个reducer来完成笛卡尔积。

### 12.7  行列过滤

列处理：在SELECT中，只拿需要的列，如果有，尽量使用分区过滤，少用SELECT *。

行处理：在分区剪裁中，当使用外关联时，如果将副表的过滤条件写在Where后面，那么就会先全表关联，之后再过滤，比如：

**案例实操：**

1. 测试先关联两张表，再用where条件过滤

   ```sql
   hive (default)> select o.id from bigtable b
   join ori o on o.id = b.id
   where o.id <= 10;
   Time taken: 34.406 seconds, Fetched: 100 row(s)
   ```

2. 通过子查询后，再关联表

   ```sql
   hive (default)> select b.id from bigtable b
   join (select id from ori where id <= 10 ) o on b.id = o.id;
   Time taken: 30.058 seconds, Fetched: 100 row(s)
   ```

### 12.7 动态分区调整

关系型数据库中，对分区表Insert数据时候，数据库自动会根据分区字段的值，将数据插入到相应的分区中，Hive中也提供了类似的机制，即动态分区(Dynamic Partition)，只不过，使用Hive的动态分区，需要进行相应的配置。

hive的动态分区是以第一个表的分区规则，来对应第二个表的分区规则，将第一个表的所有分区，全部拷贝到第二个表中来，第二个表在加载数据的时候，不需要指定分区了，直接用第一个表的分区即可

#### 12.7.1 开启动态分区参数设置

1. 开启动态分区功能（默认true，开启）

   ```sql
   set hive.exec.dynamic.partition=true;
   ```

2. 设置为非严格模式，（动态分区的模式，默认strict，表示必须执行至少一个分区为静态分区，nonstrict模式表示允许所有的分区字段都可以使用动态分区）。

   ```sql
   set hive.exec.dynamic.partition.mode=nonstrict;
   ```

3. 在所有执行MR的节点上，最大一共可以创建多少个动态分区。

   ```sql
   set hive.exec.max.dynamic.partitions = 100;
   ```

4. **在每个执行MR的节点上 ，最大可以创建多少个动态分区。**该参数需要根据实际的数据来设定。该参数需要根据实际的数据来设定。比如：源数据中包含了一年的数据，即day字段有365个值，那么该参数就需要设置成大于365，如果使用默认值100，则会报错。

   ```sql
   set hive.exec.max.dynamic.partitions.pernode=100;
   ```

5. 整个MR Job中，最大可以创建多少个HDFS文件。

   > 在linux系统当中，每个linux用户最多可以开启1024个进程，每一个进程最多可以打开2048个文件，即持有2048个文件句柄，下面这个值越大，就可以打开文件句柄越大。

   ```sql
   set hive.exec.max.created.files=100000;
   ```

6. 当有空分区生成时，是否抛出异常。一般情况下不需要设置

   ```sql
   set hive.error.on.empty.partition = false;
   ```

#### 12.7.2 案例操作

**需求：**将ori中的数据按照时间（如20111231234568），插入到目标表ori_partitioned的相应分区中。

1. 准备数据原表

   ```sql
   create table ori_partitioned(id bigint, time bigint, uid string, keyword 
   string, url_rank int, click_num int, click_url string) 
   PARTITIONED BY (p_time bigint) 
   row format delimited fields terminated by '\t';
   
   load data local inpath '/export/servers/hivedatas/small_data' into  table
   ori_partitioned partition (p_time='20111230000010');
   
   load data local inpath '/export/servers/hivedatas/small_data' into  table
   ori_partitioned partition (p_time='20111230000011');
   ```

2. 创建目标分区表

   ```sql
   create table ori_partitioned_target(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) PARTITIONED BY (p_time STRING) row format delimited fields terminated by '\t'
   ```

3. 向目标分区表中加载数据

   > 如果按照之前介绍的往指定一个分区中insert数据，那么这个需求很不容易实现。这时候就需要使用动态分区来实现

   ```sql
   INSERT overwrite TABLE ori_partitioned_target PARTITION (p_time)
   SELECT id, time, uid, keyword, url_rank, click_num, click_url, p_time
   FROM ori_partitioned;
   ```

   **注意：**在select子句的最后几个字段，必须对应前面PARTITION (p_time)中指定的分区字段，包括顺序。

4. 查看分区

   ```sql
   show partitons ori_partitioned_target;
   ```

### 12.8 数据倾斜

#### 12.8.1合理设置Map数

1. 通常情况下，作业会通过input的目录产生一个或者多个map任务。

   主要的决定因素有：input的文件总个数，input的文件大小，集群设置的文件块大小。

2. 是不是map数越多越好？

   **答案是否定的**。如果一个任务有很多小文件（远远小于块大小128m），则每个小文件也会被当做一个块，用一个map任务来完成，而一个map任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的map数是受限的。

3. 是不是保证每个map处理接近128m的文件块，就高枕无忧了？

   答案也是不一定。比如有一个127m的文件，正常会用一个map去完成，但这个文件只有一个或者两个小字段，却有几千万的记录，如果map处理的逻辑比较复杂，用一个map任务去做，肯定也比较耗时。

   针对上面的问题2和3，我们需要采取两种方式来解决：即减少map数和增加map数；

#### 12.8.2 小文件进行合并

在map执行前合并小文件，减少map数：CombineHiveInputFormat具有对小文件进行合并的功能（系统默认的格式）。HiveInputFormat没有对小文件合并功能。

```
set hive.input.format= org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

#### 12.8.3 复杂文件增加Map数

当input的文件都很大，任务逻辑复杂，map执行非常慢的时候，可以考虑增加Map数，来使得每个map处理的数据量减少，从而提高任务的执行效率。

增加map的方法为：根据`computeSliteSize(Math.max(minSize,Math.min(maxSize,blocksize)))=blocksize=128M`公式，调整maxSize最大值。让maxSize最大值低于blocksize就可以增加map的个数。

**案例实操：**

1. 执行查询

   ```
   select count(*) from emp;
   
   Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 1
   ```

2. 设置最大切片值为100个字节

   ```
   hive (default)> set mapreduce.input.fileinputformat.split.maxsize=100;
   hive (default)> select count(*) from emp;
   
   Hadoop job information for Stage-1: number of mappers: 6; number of reducers: 1
   ```

#### 12.8.4 合理设置Reduce数

1. 调整reduce个数方法一

   - 每个Reduce处理的数据量默认是256MB

     ```
     hive.exec.reducers.bytes.per.reducer=256000000
     ```

   - 每个任务最大的reduce数，默认为1009

     ```
     hive.exec.reducers.max=1009
     ```

   - 计算reducer数的公式

     N=min(参数2，总输入数据量/参数1)

2. 调整reduce个数方法二

   在hadoop的mapred-default.xml文件中修改

   设置每个job的Reduce个数

   `set mapreduce.job.reduces = 15;`

3. reduce个数并不是越多越好

   - 过多的启动和初始化reduce也会消耗时间和资源；
   - 另外，有多少个reduce，就会有多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；
   - 在设置reduce个数的时候也需要考虑这两个原则：**处理大数据量利用合适的reduce数；使单个reduce任务处理数据量大小要合适；**

### 12.9 并行执行

Hive会将一个查询转化成一个或多个阶段。这样的阶段可以是MapReduce阶段、抽样阶段、合并阶段、limit阶段。或者hive执行过程中可能需要的其他阶段。默认情况下，hive一次只会执行一个阶段。不过，某个特点的job可能包含众多的阶段，而这些阶段可能并非完全相互依赖的，也就是说，有些阶段是可以并行执行的，这样可能时整个job的执行时间缩短。不过，如果有更多的阶段可以并行执行，那么job可能就越快完成。

通过设置参数`hive.exec.parallel`的值为true，就可以开启并发执行，不过，在共享集群中，需要注意下，如果job中并行阶段增多，那么集群利用率就会增加

```sql
set hive.exec.parallel=true;       //打开任务并行执行
set hive.exec.parallel.thread.number=16;  //同一个sql允许最大并行度，默认为8。
```

当然，得是在系统资源比较空闲的时候才有优势，否则，没资源，也并行不起来

### 12.10 严格模式

Hive提供了一个严格模式，可以防止用户执行那些可能意向不到的不好的影响的查询。

通过设置属性hive.mapred.mode值为默认是非严格模式nonstrict 。开启严格模式需要修改`hive.mapred.mode`值为strict，开启严格模式可以禁止3种类型的查询。

```sql
set hive.mapred.mode = strict; #开启严格模式
set hive.mapred.mode = nostrict; #开启非严格模式
```

1. **对于分区表，在where语句中必须含有分区字段作为过滤条件来限制范围，否则不允许执行 。**换句话说，就是用户不允许扫描所有分区。进行这个限制的原因是，通常分区表都拥有非常大的数据集，而且数据增加迅速。没有进行分区限制的查询可能会消耗令人不可接受的巨大资源来处理这个表
2. **对于使用了order by语句的查询，要求必须使用limit语句 。**因为order by为了执行排序过程会将所有的结果数据分发到同一个Reducer中进行处理，强制要求用户增加这个LIMIT语句可以防止Reducer额外执行很长一段时间
3.  **限制笛卡尔积的查询 。**对关系型数据库非常了解的用户可能期望在执行JOIN查询的时候不使用ON语句而是使用where语句，这样关系数据库的执行优化器就可以高效地将WHERE语句转化成那个ON语句。不幸的是，Hive并不会执行这种优化，因此，如果表足够大，那么这个查询就会出现不可控的情况。



### 12.11 JVM重用

JVM重用是Hadoop调优参数的内容，其对Hive的性能具有非常大的影响，特别是对于很难避免**小文件的场景或task特别多的场景，这类场景大多数执行时间都很短**

Hadoop的默认配置通常是使用派生JVM来执行map和Reduce任务的。这时JVM的启动过程可能会造成相大的开销，尤其是执行的job包含有成百上千task任务的情况。**JVM重用可以使得JVM实例在同一个job中重新使用N次。**N的值可以在Hadoop的mapred-site.xml文件中进行配置。通常在10-20之间，具体多少需要根据具体业务场景测试得出。

我们也可以在hive当中通过

```sql
set mapred.job.reuse.jvm.num.tasks=10;
```

这个设置来设置我们的jvm重用

> 这个功能的缺点是，开启JVM重用将一直占用使用到的task插槽，以便进行重用，直到任务完成后才能释放。如果某个“不平衡的”job中有某几个reduce task执行的时间要比其他Reducetask消耗的时间多的多的话，那么保留的插槽就会一直空闲着却无法被其他的job使用，直到所有的task都结束了才会释放。

### 12.12 推测执行

在分布式集群环境下，因为程序Bug（包括Hadoop本身的bug），负载不均衡或者资源分布不均等原因，会造成同一个作业的多个任务之间运行速度不一致，有些任务的运行速度可能明显慢于其他任务（比如一个作业的某个任务进度只有50%，而其他所有任务已经运行完毕），则这些任务会拖慢作业的整体执行进度。为了避免这种情况发生，Hadoop采用了推测执行（Speculative Execution）机制 ，**它根据一定的法则推测出“拖后腿”的任务，并为这样的任务启动一个备份任务，让该任务与原始任务同时处理同一份数据，并最终选用最先成功运行完成任务的计算结果作为最终结果**

设置开启推测执行参数：

```sql
set mapred.map.tasks.speculative.execution=true
set mapred.reduce.tasks.speculative.execution=true
set hive.mapred.reduce.tasks.speculative.execution=true;
```

关于调优这些推测执行变量，还很难给一个具体的建议。如果用户对于运行时的偏差非常敏感的话，那么可以将这些功能关闭掉。如果用户因为输入数据量很大而需要执行长时间的map或者Reduce task的话，那么启动推测执行造成的浪费是非常巨大大

### 12.13执行计划（Explain）

1. 基本语法

   ```
   EXPLAIN [EXTENDED | DEPENDENCY | AUTHORIZATION] query
   ```

2. 案例实操

   1. 查看下面这条语句的执行计划

      ```sql
      hive (default)> explain select * from emp;
      hive (default)> explain select deptno, avg(sal) avg_sal from emp group by deptno;
      ```

   2. 查看详细执行计划

      ```sql
      hive (default)> explain extended select * from emp;
      hive (default)> explain extended select deptno, avg(sal) avg_sal from emp group by deptno;
      ```

      