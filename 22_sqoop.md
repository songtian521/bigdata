#  22_sqoop

## 1.sqoop介绍

apache sqoop是在hadoop生态体系和RDBMS体系之间传送数据的一种工具。来自于Apache软件基金会提供。

sqoop工作机制是将导入或导出命令翻译成mapreduce程序来实现。在翻译出的mapreduce中主要是对inputformat和outputformat进行定制

- hadoop生态体系包括：HDFS、HIVE、Hbase等
- RDBMS体系包括：Mysql、Oracle、DB2等
- sqoop可以理解为：SQL到hadoop和hadoop到SQL

![](img\sqoop原理.png)

站在Apache立场看待数据流转问题，可以分为数据的导入导出：

import：数据导入。RDBMS--->Hadoop

export：数据导出。Hadoop--->RDBMS

## 2.sqoop安装

注：安装sqoop的前提是已经具备java和hadoop的环境

### 2.1 下载并解压

1.  最新版下载地址：http://mirrors.hust.edu.cn/apache/sqoop/1.4.7/

2. 上传安装包到虚拟机software目录

3. 解压sqoop压缩包到指定目录

   ```
   tar -zxvf sqoop-1.4.7.bin__hadoop-2.0.4-alpha.tar.gz -C /opt/module
   ```

### 2.2 修改配置文件

sqoop的配置文件与大多数大数据框架类似，在sqoop根目录下的conf目录中

1. 重命名配置文件

   ```
   mv sqoop-env-template.sh sqoop-env.sh
   ```

2. 修改配置文件

   ```
   #Set path to where bin/hadoop is available
   export HADOOP_COMMON_HOME=/opt/module/hadoop-2.8.4
   
   #Set path to where hadoop-*-core.jar is available
   export HADOOP_MAPRED_HOME=/opt/module/hadoop-2.8.4
   
   #set the path to where bin/hbase is available
   #export HBASE_HOME= 
   
   #Set the path to where bin/hive is available
   export HIVE_HOME=/opt/module/hive-2.3.4
   export HIVE_CONF_DIR=/opt/module/hive-2.3.4/conf
   
   #Se tthe path for where zookeper config dir is
   export ZOOKEEPER_HOME=/opt/module/zookeeper-3.4.10
   export ZOOCFGDIR=/opt/module/zookeeper-3.4.10/conf
   
   ```

3. 设置PATH

   ```
   vim /etc/profile
   
   #sqoop
   export SQOOP_HOME=/opt/module/sqoop-1.4.7
   export PATH=$PATH:$SQOOP_HOME/bin
   ```

   然后重启配置文件

   source /etc/profile

### 2.3 拷贝JDBC驱动

拷贝jdbc驱动到sqoop的lib目录下

```
$ cp -a mysql-connector-java-5.1.27-bin.jar /opt/module/sqoop-1.4.6.bin__hadoop-2.0.4-alpha/lib
```

也可以把事先准备好的驱动直接上传到sqoop的lib目录中

**注意：**sqoop-1.4.7版本需要将hive/lib/hive-exec-x.x.x.jar拷贝到sqoop的lib目录下，不然在RDBMS到Hive的过程中会报错：如下

```
19/12/27 18:04:19 ERROR hive.HiveConfig: Could not load org.apache.hadoop.hive.conf.HiveConf. Make sure HIVE_CONF_DIR is set correctly.
19/12/27 18:04:19 ERROR tool.ImportTool: Import failed: java.io.IOException: java.lang.ClassNotFoundException: org.apache.hadoop.hive.conf.HiveConf
```



### 2.4 验证sqoop

我们可以通过某一个command来验证sqoop的配置是否正确

```
[root@bigdata111 lib]# bin/sqoop help
19/12/27 13:27:40 INFO sqoop.Sqoop: Running Sqoop version: 1.4.7
usage: sqoop COMMAND [ARGS]

Available commands:
  codegen            Generate code to interact with database records
  create-hive-table  Import a table definition into Hive
  eval               Evaluate a SQL statement and display the results
  export             Export an HDFS directory to a database table
  help               List available commands
  import             Import a table from a database to HDFS
  import-all-tables  Import tables from a database to HDFS
  import-mainframe   Import datasets from a mainframe server to HDFS
  job                Work with saved jobs
  list-databases     List available databases on a server
  list-tables        List available tables in a database
  merge              Merge results of incremental imports
  metastore          Run a standalone Sqoop metastore
  version            Display version information

See 'sqoop help COMMAND' for information on a specific command.
```

**去掉警告信息**

注释掉bin目录下configure-sqoop 134行到143行的内容，如下

```
    134 ## Moved to be a runtime check in sqoop.
     135 #if [ ! -d "${HCAT_HOME}" ]; then
    136 #  echo "Warning: $HCAT_HOME does not exist! HCatalog jobs will fail."
    137 #  echo 'Please set $HCAT_HOME to the root of your HCatalog installation.'
    138 #fi
    139 #
    140 #if [ ! -d "${ACCUMULO_HOME}" ]; then
    141 #  echo "Warning: $ACCUMULO_HOME does not exist! Accumulo imports will fail."
    142 #  echo 'Please set $ACCUMULO_HOME to the root of your Accumulo installation.'
    143 #fi
```

### 2.5 测试sqoop会否能成功连接数据库

```
[root@bigdata111 lib]# sqoop list-databases \
> --connect jdbc:mysql://bigdata111:3306 \
> --username root \
> --password 000000
19/12/27 13:28:55 INFO sqoop.Sqoop: Running Sqoop version: 1.4.7
19/12/27 13:28:55 WARN tool.BaseSqoopTool: Setting your password on the command-line is insecure. Consider using -P instead.
19/12/27 13:28:55 INFO manager.MySQLManager: Preparing to use a MySQL streaming resultset.
information_schema
metastore
mysql
performance_schema
sys
```

## 3.sqoop导入

在sqoop中，导入的概念指：从非大数据集群（RDBMS）向大数据集群（HDFS,HIVE,Hbase）中传输数据，叫做：导入，即使用import关键字

**数据准备：**

1. 确定Mysql服务开启正常
2. 在Mysql中新建一张表并插入一些数据

```
mysql> create database company;
mysql> create table company.staff(id int(4) primary key not null auto_increment, name varchar(255), sex varchar(255));
mysql> insert into company.staff(name, sex) values('Thomas', 'Male');
mysql> insert into company.staff(name, sex) values('Catalina', 'FeMale');
```



### 3.1 全量导入mysql表数据到HDFS

下面命令用于从mysql数据库服务器中的staff表导入HDFS

```sql
[root@bigdata111 ~]# sqoop import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root \
--password 000000 \
--delete-target-dir \
--target-dir /company \
--table staff \
--m 1;
```

其中`--target-dir`可以用来指定导出数据存放至HDFS目录

**注：mysql jdbc url 请使用ip地址**

为了验证在HDFS导入的数据，请使用以下命令查看导入的数据

```
hdfs dfs -cat /company/part-m-00000
```

可以看出它会在HDFS上默认使用逗号，分割staff表中的数据和字段。**可以使用`--fields-terminated-by '\t'`来指定分隔符。**

```
[root@bigdata111 ~]# hdfs dfs -cat /company/part-m-00000
1,Thomas,Male
2,Catalina,FeMale
```

### 3.2 全量导入mysql表数据到HIVE

1. 方式1：

   1. 先复制表结构到hive中在导入数据

      ```sql
      sqoop create-hive-table 
      --connect jdbc:mysql://bigdata111:3306/company \
      --table staff \
      --username root \ 
      --password 000000 \ 
      --hive-table myhive.staff_hive;
      ```

      其中：

      **--table staff** 为mysql中的数据库company中的表

      **--hive-table staff** 为hive中新建表的名称

   2. 从关系数据库导入文件到hive中

      ```sql
      sqoop import  \
      --connect jdbc:mysql://bigdata111:3306/company \
      --username root \
      --password 000000 \ 
      --table staff \
      --hive-table myhive.staff_hive \
      --hive-import \
      --m 1;
      ```

2. 方式2：直接复制表结构数据到hive中

   ```sql
   sqoop import \
   --connect jdbc:mysql://bigdata111:3306/company \ 
   --username root \
   --password 000000 \ 
   --table staff \
   --hive-import \
   --m 1 \
   --hive-database myhive;
   ```

### 3.3 导入表数据子集（where过滤）

`--where` 可以指定从关系数据库导入数据时的查询条件。它执行在数据库服务器相应的SQL查询，并将结果存储在HSFS的目标目录

```
sqoop import \
> --connect jdbc:mysql://bigdata111:3306/company \
> --username root \
> --password 000000 \
> --where 'id>1' \
> --target-dir /company1 \
> --table staff \
> --m 1;
```

查看数据：

```
[root@bigdata111 conf]# hdfs dfs -cat /company1/*
2,Catalina,FeMale
```

### 3.4 导入表数据子集（query查询）

**注意事项：**

1. 使用query sql语句来进行查找不能加参数--table
2. 并且必须要添加where条件
3. 并且where条件后面必须带一个`$CONDITIONS`这个字符串
4. 并且这个sql语句必须使用单引号，不能用双引号

```sql
sqoop import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root \
--password 000000 \
--target-dir /company2 \
--query 'select id,name from staff where id < 3 and $CONDITIONS' \
--split-by id  \
--fields-terminated-by '\t' \
--m 2;
```

sqoop命令中，`--split-by id`通常配合--m 10参数使用。用于指定根据哪个字段进行划分并启动多少个maptask

### 3.5 增量导入

在实际工作当中，数据的导入，很多时候都只是需要导入增量数据即可，并不需要将表中的数据每次都全部导入到hive或者hdfs中去，这样会造成数据重复的问题。因此一般都是选用一些字段进行增量的导入，sqoop支持增量的导入数据。

**增量导入是仅导入新添加的表中的行的技术**

1. --check-column（col）

   用来指定一些列，这些列在增量导入时用来检查这些数据是否作为增量数据进行导入，和关系型数据库中的自增字段和时间戳类似。

   **注意：**这些被指定的列的类型不能使用任意字符类型，如char、varcahr等类型都是不可以的，同时--check-column可以指定多个列
   
2. --incremental（mode）

   append：追加，比如对大于last-value指定的值之后的记录进行追加导入。

   lastmodiffied：最后的修改时间，追加last-value执行的日期之后的记录

3. --last-value（value）

   指定自从上次导入后列的最大值（大于该指定的值），也可以自己设定某一值

#### 3.5.1 Append模式增量导入

执行以下指令先将之前的数据导入：

```sql
sqoop import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root --password 000000 \
--target-dir /company4 \
--table staff \
--m 1;
```

然后使用hdfs dfs -cat /company4/* 查看生成的数据文件，发现数据已经导入到hdfs中。

然后在使用mysql的staff中插入两条增量数据：

```
mysql> insert into staff values(3,'ASDF','ASD');
Query OK, 1 row affected (0.00 sec)

mysql> insert into staff values(4,'ASSFDS','AASD');
Query OK, 1 row affected (0.00 sec)

mysql> insert into staff values(5,'BFDS','AASD');
Query OK, 1 row affected (0.01 sec)

mysql> insert into staff values(6,'CFDS','AASD');
Query OK, 1 row affected (0.00 sec)
```

执行如下指令，实现增量的导入：

```sql
sqoop import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root \
--password 000000 \
--table staff \
--target-dir /company4 \
--incremental append \
--check-column id \
--last-value 5;
```

最后验证导入数据目录，就可以发现多了一个文件，里面就是增量数据

```
[root@bigdata111 conf]# hadoop fs -ls /company4 
Found 3 items
-rw-r--r--   3 root supergroup          0 2019-12-29 19:25 /company4/_SUCCESS
-rw-r--r--   3 root supergroup         43 2019-12-29 19:25 /company4/part-m-00000
-rw-r--r--   3 root supergroup         12 2019-12-29 19:31 /company4/part-m-00001
[root@bigdata111 conf]# hadoop fs -cat  /company4/part-m-00001
6,CFDS,AASD
```

#### 3.5.2 Lastnodified模式导入

首先创建一个customer表，指定一个时间戳字段：

```sql
create table customertest(id int,name varchar(20),last_mod 
timestamp default current_timestamp on update current_timestamp);
```

此处的时间戳设置为在数据的产生和更新时都会发生改变。

分别插入如下记录：

```sql
insert into customertest(id,name) values(1,'neil');
insert into customertest(id,name) values(2,'jack');
insert into customertest(id,name) values(3,'martin');
insert into customertest(id,name) values(4,'tony');
insert into customertest(id,name) values(5,'eric');
```

执行sqoop指令将数据全部导入hdfs中：

```sql
sqoop import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root \
--password 000000 \
--target-dir /lastmodifiedresult \
--table customertest \
--m 1;
```

查看此时导出的结果数据：

```
[root@bigdata111 conf]# hdfs dfs -cat /lastmodifiedresult/* 
1,neil,2019-12-29 19:36:40.0
2,jack,2019-12-29 19:36:40.0
3,martin,2019-12-29 19:36:40.0
4,tony,2019-12-29 19:36:40.0
5,eric,2019-12-29 19:37:11.0
```

再次插入一条数据进入 customertest 表

```sql
insert into customertest(id,name) values(6,'james');
```

使用incremental方式进行增量的导入：

```sql
sqoop import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root --password 000000 \
--table customertest --target-dir /lastmodifiedresult \
--check-column last_mod \
--incremental lastmodified \
--last-value '2019-12-29 19:36:40' \
--m 1 \
--append ;
```

再次查看数据：

```
[root@bigdata111 conf]# hdfs dfs -cat /lastmodifiedresult/*
1,neil,2019-12-29 19:36:40.0
2,jack,2019-12-29 19:36:40.0
3,martin,2019-12-29 19:36:40.0
4,tony,2019-12-29 19:36:40.0
5,eric,2019-12-29 19:37:11.0
1,neil,2019-12-29 19:36:40.0
2,jack,2019-12-29 19:36:40.0
3,martin,2019-12-29 19:36:40.0
4,tony,2019-12-29 19:36:40.0
5,eric,2019-12-29 19:37:11.0
6,james,2019-12-29 19:43:13.0
```

此时已经会导入我们最后插入的一条记录，但是我们却也发现此处插入了2条数据，这是为什么呢？

**答案：**这是因为采用lastmodified模式去处理增量时，会将大于等于last-value的值的数据当做增量插入

#### 3.5.3 Lastnodified模式：append、merge-key

使用lastmodified模式进行增量处理要指定增量数据是以append模式（附加）还是merge-key（合并）模式添加

下面演示使用merge-key的模式进行增量更新，我们去更新id为1的name字段。

```sql
update customertest set name = 'Neil' where id = 1
```

更新之后，这条数据的时间戳会更新为更新数据时的系统时间

执行如下指令，把id字段作为merge-key：

```sql
sqoop import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root \
--password 000000 \
--table customertest \
--target-dir /lastmodifiedresult \
--check-column last_mod \
--incremental lastmodified \
--last-value '2019-12-29 19:36:40' \
--m 1 \
--merge-key id ;
```

由于merge-key模式是进行了一次完整的mapreduce操作，因此最终我们在lastmodifiedresult文件夹下可以看到生成为part-r-00000 这样的文件，会发现id=1的name已经得到修改，同时新增了id=6的数据

```
[root@bigdata111 conf]# hdfs dfs -cat /lastmodifiedresult/*
1,Neil,2019-12-29 19:53:11.0
2,jack,2019-12-29 19:36:40.0
3,martin,2019-12-29 19:36:40.0
4,tony,2019-12-29 19:36:40.0
5,eric,2019-12-29 19:37:11.0
6,james,2019-12-29 19:43:13.0
```

### 3.6 RDBMS到hive

```sql
sqoop import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root \
--password 000000 \
--table staff \
--num-mappers 1 \
--hive-import \
--fields-terminated-by "\t" \
--hive-overwrite \
--hive-table staff_hive
```

1. 该过程分为两步，第一步将数据导入HDFS，第二步将导入HDFS的数据迁移到HIVE仓库
2. 从mysql到hive，本质是从MYSQL => HDFS => load To Hive

## 4.sqoop导出

在Sqoop中，“导出”概念指：从大数据集群（HDFS，HIVE，HBASE）向非大数据集群（RDBMS）中传输数据，叫做：导出，即使用export关键字。

将数据从hadoop生态体系导出到RDBMS数据库导出前，目标表必须存在于目标数据库中。

export有三种模式：

1. 默认操作是从将文件中的数据使用insert语句插入到表中
2. 更新模式：sqoop将生成updatet替换数据库现有记录的语句
3. 调用模式：sqoop将为每条记录创建一个存储过程的调用

以下是export命令语法：

```
$ sqoop export (generic-args) (export-args)
```



### 4.1 默认模式导出HDFS数据到mysql

默认情况下，sqoop export将每行输入记录转换成一条insert语句，再添加到目标数据库表中。如果数据库中的表具有约束条件（例如，其值必须唯一的主键列）并且已有数据存在，则必须注意避免插入违反这些约束条件的记录。如果insert语句失败，导出过程将会失败。**此模式主要用于将记录导出到可以接受这些结果的空表中。通常用于全表数据导出。**

导出时可以是将hive表中的全部记录或者HDFS数据（可以是全部字段也可以是部分字段）导出到mysql目标表。

1. 准备HDFS数据

   在HDFS文件系统中，‘/emp/’的目录下创建一个文件emp_data.txt:

   ```
   1201,gopal,manager,50000,TP
   1202,manisha,preader,50000,TP
   1203,kalil,php dev,30000,AC
   1204,prasanth,php dev,30000,AC
   1205,kranthi,admin,20000,TP
   1206,satishp,grpdes,20000,GR
   ```

2. 手动创建mysql中的目标表

   ```sql
    use company;
    CREATE TABLE employee ( 
    id INT NOT NULL PRIMARY KEY, 
    name VARCHAR(20), 
    deg VARCHAR(20),
    salary INT,
    dept VARCHAR(10));
   ```

3. 执行导出命令

   ```sql
   sqoop export \
   > --connect jdbc:mysql://bigdata111:3306/company \
   > --username root \
   > --password 000000 \
   > --table employee \
   > --export-dir /emp/emp_data.txt;
   ```

4. 相关配置参数

   - --input-fields-terminated-by '\t'

     指定文件中的分隔符

   - --columns

     选择列并控制他们的排序。当导出数据文件和目标字段顺序完全一致的时候可以不写。否则以逗号为间隔选择和排列各个列。没有被包含在-columns后面列名或者字段要么具备默认值，要么就允许插入空值。否则数据库会拒绝接受sqoop导出的数据，导致sqoop作业失败

   - --export-dir

     导出目录，在执行导出的时候，必须指定这个参数，同时需要具备`--table`或`--call`参数两者之一，`--table`是指导出数据库当中对应的表，`--call`是指的某个存储过程。

   - `--input-null-string ` `--input-null-non-string`

     如果没有指定第一个参数，对于字符串类型的列表来说，null这个字符串就会被翻译成空值，如果没有使用第二个参数，无论是null字符串还是说空字符串也好，对于非字符串类型的字段拉说，这两个类型的空串都会被翻译成空值。比如：

     ```
     --input-null-string "\\N" --input-null-non-string "\\N"
     ```

### 4.2 更新导出（updateonly模式）

1. 参数说明

   `--update-key`，更新标识，即根据某个字段进行更新，例如id，可以指定多个更新标识的字段，多个字段之间用逗号分隔。

   `updatemod`，指定updateonly（默认模式），仅仅更新已存在的数据记录，不会插入新纪录。

2. 准备HDFS数据

   在hdfs ’updateonly‘目录下创建一个文件uptateonly.txt

   ```
   1201,gopal,manager,50000
   1202,manisha,preader,50000
   1203,kalil,php dev,300
   ```

3. 手动创建mysql中的目标表

   ```sql
   CREATE TABLE updateonly ( 
    id INT NOT NULL PRIMARY KEY, 
    name VARCHAR(20), 
    deg VARCHAR(20),
    salary INT);
   ```

4. 先执行全部导出操作

   ```sql
   sqoop export \
   --connect jdbc:mysql://bigdata111:3306/company \
   --username root  \
   --password 000000 \
   --table updateonly \
   --export-dir /updateonly_1;
   ```

5. 此时mysql中的数据

   可以发现是全量导出，全部的数据

   ```
   mysql> select * from updateonly;
   +------+---------+---------+--------+
   | id   | name    | deg     | salary |
   +------+---------+---------+--------+
   | 1201 | gopal   | manager |  50000 |
   | 1202 | manisha | preader |  50000 |
   | 1203 | kalil   | php dev |    300 |
   +------+---------+---------+--------+
   3 rows in set (0.00 sec)
   ```

6. 新增一个文件

   updateonly_2.txt。修改了前三条数据并且新增了一条记录。上传至/updateonly2目录下

   ```
   1201,gopal,manager,1212
   1202,manisha,preader,1313
   1203,kalil,php dev,1414
   1204,allen,java,1515
   ```

7. 执行更新导出

   ```sql
   sqoop export \
   --connect jdbc:mysql://bigdata111/company \
   --username root \
   --password 000000 \
   --table updateonly \
   --export-dir /updateonly_2 \
   --update-key id \
   --update-mode updateonly;
   ```

8. 查看最终结果

   虽然导出时候的日志显示导出 4 条记录

   ```
   19/12/30 12:23:39 INFO mapreduce.ExportJobBase: Transferred 871 bytes in 28.7377 seconds (30.3086 bytes/sec)
   19/12/30 12:23:39 INFO mapreduce.ExportJobBase: Exported 4 records.
   ```

   但最终只进行了更新操作

   ```
   mysql> select * from updateonly;
   +------+---------+---------+--------+
   | id   | name    | deg     | salary |
   +------+---------+---------+--------+
   | 1201 | gopal   | manager |   1212 |
   | 1202 | manisha | preader |   1313 |
   | 1203 | kalil   | php dev |   1414 |
   +------+---------+---------+--------+
   3 rows in set (0.00 sec)
   ```

   

### 4.3 更新导出（allowinsert模式）

1. 参数说明

   `--update-key`，更新表示，即根据某个字段进行更新，例如id，可以指定多个更新标识的字段，多个字段之间用逗号分隔。

   `--updatemod`，指定allowinsert，更新已存在的数据记录，同时插入新纪录。实质上是一个insert & update的操作

2. 准备HDFS数据

   在HDFS的allowinsert_1目录下创建一个文件allowinsert_1.txt

   ```
   1201,gopal,manager,50000
   1202,manisha,preader,50000
   1203,kalil,php dev,300
   ```

3. 手动创建mysql中的目标表

   ```sql
    CREATE TABLE allowinsert ( 
    id INT NOT NULL PRIMARY KEY, 
    name VARCHAR(20), 
    deg VARCHAR(20),
    salary INT);
   ```

4. 先执行全部导出操作

   ```sql
   sqoop export \
   --connect jdbc:mysql://bigdata111:3306/company \
   --username root \
   --password 000000 \
   --table allowinsert \
   --export-dir /allowinsert_1;
   ```

5. 查看此时mysql中的数据

   可以发现是全量导出，全部的数据

   ```
   mysql> select * from allowinsert;
   +------+---------+---------+--------+
   | id   | name    | deg     | salary |
   +------+---------+---------+--------+
   | 1201 | gopal   | manager |  50000 |
   | 1202 | manisha | preader |  50000 |
   | 1203 | kalil   | php dev |  30000 |
   +------+---------+---------+--------+
   3 rows in set (0.00 sec)
   
   ```

6. 新增一个文件

   allowinsert_2.txt。修改了前三条数据并且新增了一条记录。上传至/
   allowinsert_2/目录下

   ```
   1201,gopal,manager,1212
   1202,manisha,preader,1313
   1203,kalil,php dev,1414
   1204,allen,java,15
   ```

7. 执行更新导出

   ```sql
   sqoop export \
   --connect jdbc:mysql://bigdata111:3306/company \
   --username root \
   --password 000000 \
   --table allowinsert \
   --export-dir /allowinsert_2 \
   --update-key id \
   --update-mode allowinsert ;
   ```

8. 查看最终结果

   导出时候的日志显示导出 4 条记录：

   ```
   19/12/30 11:08:21 INFO mapreduce.ExportJobBase: Transferred 872 bytes in 27.09 seconds (32.189 bytes/sec)
   19/12/30 11:08:21 INFO mapreduce.ExportJobBase: Exported 4 records.
   ```

   数据进行更新操作的同时也进行了新增的操作

   ```sql
   mysql> select * from allowinsert;
   +------+---------+---------+--------+
   | id   | name    | deg     | salary |
   +------+---------+---------+--------+
   | 1201 | gopal   | manager |   1212 |
   | 1202 | manisha | preader |   1313 |
   | 1203 | kalil   | php dev |   1414 |
   | 1204 | allen   | java    |     15 |
   +------+---------+---------+--------+
   4 rows in set (0.00 sec)
   
   ```

### 4.4 脚本打包

使用opt格式的文件打包sqoop命令，然后执行

1. 创建一个.opt文件

   ```
   $ touch job_HDFS2RDBMS.opt
   ```

2. 编写sqoop脚本

   ```
   $ vi ./job_HDFS2RDBMS.opt
   #以下命令是从staff_hive中追加导入到mysql的aca表中
   
   export
   --connect
   jdbc:mysql://bigdata113:3306/Andy
   --username
   root
   --password
   000000
   --table
   aca
   --num-mappers
   1
   --export-dir
   /user/hive/warehouse/staff_hive
   --input-fields-terminated-by
   "\t"
   ```

3. 执行该脚本

   ```
   $ bin/sqoop --options-file job_HDFS2RDBMS.opt
   ```

测试失败：

```
19/12/30 18:29:19 ERROR mapreduce.ExportJobBase: Export job failed!
19/12/30 18:29:19 ERROR tool.ExportTool: Error during export: 
Export job failed!
	at org.apache.sqoop.mapreduce.ExportJobBase.runExport(ExportJobBase.java:445)
	at org.apache.sqoop.manager.SqlManager.exportTable(SqlManager.java:931)
	at org.apache.sqoop.tool.ExportTool.exportTable(ExportTool.java:80)
	at org.apache.sqoop.tool.ExportTool.run(ExportTool.java:99)
	at org.apache.sqoop.Sqoop.run(Sqoop.java:147)
	at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:76)
	at org.apache.sqoop.Sqoop.runSqoop(Sqoop.java:183)
	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:234)
	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:243)
	at org.apache.sqoop.Sqoop.main(Sqoop.java:252)
```

已排除问题项：表结构

## 5.sqoop job作业

### 5.1 job语法

```
$ sqoop job (generic-args)
(job-args [-- [subtool-name] (subtool-args)]

$ sqoop-job (generic-args)
(job-args [-- [subtool-name] (subtool-args)]
```

### 5.2 创建job

在这里，我们创建一个名为songjob，这可以从RDDBMS表的数据导入到HDFS作业。

下面的命令用于创建一个从DB数据库的emp表导入到HDFS文件的作业

```
sqoop job --create songjob -- import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root \
--password 000000 \
--target-dir /emp \
--table updateonly \
--m 1;
```

**测试失败：**

```
Warning: /opt/module/sqoop-1.4.7/../hbase does not exist! HBase imports will fail.
Please set $HBASE_HOME to the root of your HBase installation.
19/12/30 14:14:42 INFO sqoop.Sqoop: Running Sqoop version: 1.4.7
19/12/30 14:14:42 WARN tool.BaseSqoopTool: Setting your password on the command-line is insecure. Consider using -P instead.
19/12/30 14:14:43 ERROR sqoop.Sqoop: Got exception running Sqoop: java.lang.NullPointerException
java.lang.NullPointerException
	at org.json.JSONObject.<init>(JSONObject.java:144)
	at org.apache.sqoop.util.SqoopJsonUtil.getJsonStringforMap(SqoopJsonUtil.java:43)
	at org.apache.sqoop.SqoopOptions.writeProperties(SqoopOptions.java:785)
	at org.apache.sqoop.metastore.hsqldb.HsqldbJobStorage.createInternal(HsqldbJobStorage.java:399)
	at org.apache.sqoop.metastore.hsqldb.HsqldbJobStorage.create(HsqldbJobStorage.java:379)
	at org.apache.sqoop.tool.JobTool.createJob(JobTool.java:181)
	at org.apache.sqoop.tool.JobTool.run(JobTool.java:294)
	at org.apache.sqoop.Sqoop.run(Sqoop.java:147)
	at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:76)
	at org.apache.sqoop.Sqoop.runSqoop(Sqoop.java:183)
	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:234)
	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:243)
	at org.apache.sqoop.Sqoop.main(Sqoop.java:252)
```



### 5.3 验证job

`--list`参数是用来验证保存的作业。下面的命令用来验证保存sqoop作业的列表。

```
sqoop job --list;
```

`--delete`参数用于删除job作业

```
sqoop job --delete myjob
```



### 5.4 检查

`--show`参数用来检查或验证特定的工作，及其详细信息。以下命令和样本输出用来验证一个名为songjob的作业

```
 sqoop job --show songjob
```

### 5.5 执行job

`-exec`选项用于执行保存的作业。下面的命令用于执行保存的作业songjob

```
sqoop job --exec songjob
```

### 5.6 免密执行job

sqoop在创建job时，使用--password参数，可以避免输入mysql密码，如果使用--password将出现警告，并且每次都要手动输入密码才能执行job.sqoop规定密码文件必须存放在HDFS上，并且权限必须是400.

并且检查 sqoop 的 sqoop-site.xml 是否存在如下配

```xml
<property>
 <name>sqoop.metastore.client.record.password</name>
 <value>true</value>
 <description>If true, allow saved passwords in the metastore.
 </description>
</property
```

先创建一个文件password.pwd的文件用于保存mysql密码，然后在HDFS上创建文件夹用于保存上传的密码文件，然后更改权限

```
hadoop fs -chmod 400 /sqooppassword/password.pwd
```

在创建job时，使用--password-file参数

```sql
sqoop job --create songjob \
-- import \
--connect jdbc:mysql://bigdata111:3306/company \
--username root \
--password-file /sqooppassword/password.pwd \
--target-dir /emp \
--table staff \
--m 1;
```

**注意：import前有个空格**

最后执行job查看结果

```
sqoop job -exec songjob
```

## 6.sqoop常用命令及参数

### 6.1 常用命令列举

| ***\*序号\**** | ***\*命令\****    | ***\*类\****        | ***\*说明\****                                               |
| -------------- | ----------------- | ------------------- | ------------------------------------------------------------ |
| 1              | import            | ImportTool          | 将数据导入到集群                                             |
| 2              | export            | ExportTool          | 将集群数据导出                                               |
| 3              | codegen           | CodeGenTool         | 获取数据库中某张表数据生成Java并打包Jar                      |
| 4              | create-hive-table | CreateHiveTableTool | 创建Hive表                                                   |
| 5              | eval              | EvalSqlTool         | 查看SQL执行结果                                              |
| 6              | import-all-tables | ImportAllTablesTool | 导入某个数据库下所有表到HDFS中                               |
| 7              | job               | JobTool             | 用来生成一个sqoop的任务，生成后，该任务并不执行，除非使用命令执行该任务。 |
| 8              | list-databases    | ListDatabasesTool   | 列出所有数据库名                                             |
| 9              | list-tables       | ListTablesTool      | 列出某个数据库下所有表                                       |
| 10             | merge             | MergeTool           | 将HDFS中不同目录下面的数据合在一起，并存放在指定的目录中     |
| 11             | metastore         | MetastoreTool       | 记录sqoop job的元数据信息，如果不启动metastore实例，则默认的元数据存储目录为：~/.sqoop，如果要更改存储目录，可以在配置文件sqoop-site.xml中进行更改。 |
| 12             | help              | HelpTool            | 打印sqoop帮助信息                                            |
| 13             | version           | VersionTool         | 打印sqoop版本信息                                            |

### 6.2命令参数详解

刚才列举了一些Sqoop的常用命令，对于不同的命令，有不同的参数，让我们来一一列举说明。

首先来我们来介绍一下公用的参数，所谓公用参数，就是大多数命令都支持的参数。

#### 6.2.1 公用参数：数据库连接

| ***\*序号\**** | ***\*参数\****       | ***\*说明\****         |
| -------------- | -------------------- | ---------------------- |
| 1              | --connect            | 连接关系型数据库的URL  |
| 2              | --connection-manager | 指定要使用的连接管理类 |
| 3              | --driver             | Hadoop根目录           |
| 4              | --help               | 打印帮助信息           |
| 5              | --password           | 连接数据库的密码       |
| 6              | --username           | 连接数据库的用户名     |
| 7              | --verbose            | 在控制台打印出详细信息 |

#### 6.2.2 公用参数：import

| ***\*序号\**** | ***\*参数\****                  | ***\*说明\****                                               |
| -------------- | ------------------------------- | ------------------------------------------------------------ |
| 1              | --enclosed-by <char>            | 给字段值前加上指定的字符                                     |
| 2              | --escaped-by <char>             | 对字段中的双引号加转义符                                     |
| 3              | --fields-terminated-by <char>   | 设定每个字段是以什么符号作为结束，默认为逗号                 |
| 4              | --lines-terminated-by <char>    | 设定每行记录之间的分隔符，默认是\n                           |
| 5              | --mysql-delimiters              | Mysql默认的分隔符设置，字段之间以逗号分隔，行之间以\n分隔，默认转义符是\，字段值以单引号包裹。 |
| 6              | --optionally-enclosed-by <char> | 给带有双引号或单引号的字段值前后加上指定字符。               |

#### 6.2.3 公用参数：export

| ***\*序号\**** | ***\*参数\****                        | ***\*说明\****                             |
| -------------- | ------------------------------------- | ------------------------------------------ |
| 1              | --input-enclosed-by <char>            | 对字段值前后加上指定字符                   |
| 2              | --input-escaped-by <char>             | 对含有转移符的字段做转义处理               |
| 3              | --input-fields-terminated-by <char>   | 字段之间的分隔符                           |
| 4              | --input-lines-terminated-by <char>    | 行之间的分隔符                             |
| 5              | --input-optionally-enclosed-by <char> | 给带有双引号或单引号的字段前后加上指定字符 |

#### 6.2.4 公用参数：hive

| ***\*序号\**** | ***\*参数\****                  | ***\*说明\****                                            |
| -------------- | ------------------------------- | --------------------------------------------------------- |
| 1              | --hive-delims-replacement <arg> | 用自定义的字符串替换掉数据中的\r\n和\013 \010等字符       |
| 2              | --hive-drop-import-delims       | 在导入数据到hive时，去掉数据中的\r\n\013\010这样的字符    |
| 3              | --map-column-hive <arg>         | 生成hive表时，可以更改生成字段的数据类型                  |
| 4              | --hive-partition-key            | 创建分区，后面直接跟分区名，分区字段的默认类型为string    |
| 5              | --hive-partition-value <v>      | 导入数据时，指定某个分区的值                              |
| 6              | --hive-home <dir>               | hive的安装目录，可以通过该参数覆盖之前默认配置的目录      |
| 7              | --hive-import                   | 将数据从关系数据库中导入到hive表中                        |
| 8              | --hive-overwrite                | 覆盖掉在hive表中已经存在的数据                            |
| 9              | --create-hive-table             | 默认是false，即，如果目标表已经存在了，那么创建任务失败。 |
| 10             | --hive-table                    | 后面接要创建的hive表,默认使用MySQL的表名                  |
| 11             | --table                         | 指定关系数据库的表名                                      |

公用参数介绍完之后，我们来按照命令介绍命令对应的特有参数。

#### 6.2.5 命令&参数：import

将关系型数据库中的数据导入到HDFS（包括Hive，HBase）中，如果导入的是Hive，那么当Hive中没有对应表时，则自动创建。

1. 命令：

   如：导入数据到hive中

   ```
   $ bin/sqoop import \
   --connect jdbc:mysql://bigdata113:3306/Andy \
   --username root \
   --password 000000 \
   --table access \
   --hive-import \
   --fields-terminated-by "\t"
   ```

   如：增量导入数据到hive中，mode=append

   ```
   append导入：
   $ bin/sqoop import \
   --connect jdbc:mysql://bigdata113:3306/Andy \
   --username root \
   --password 000000 \
   --table aca \
   --num-mappers 1 \
   --fields-terminated-by "\t" \
   --target-dir /user/hive/warehouse/staff_hive \
   --check-column id \
   --incremental append \
   --last-value 10
   ```

   **尖叫提示：**append不能与--hive-等参数同时使用（Append mode for hive imports is not yet supported. Please remove the parameter --append-mode）

   **注:**`--last-value 2` 的意思是标记增量的位置为第二行，也就是说，当数据再次导出的时候，从第二行开始算

   **注：**如果 --last-value N , N > MYSQL中最大行数，则HDFS会创建一个空文件。如果N<=0 , 那么就是所有数据

   

   如：增量导入数据到hdfs中，mode=lastmodified（注：卡住）

   ```
   先在mysql中建表并插入几条数据：
   mysql> create table company.staff_timestamp(id int(4), name varchar(255), sex varchar(255), last_modified timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP);
   mysql> insert into company.staff_timestamp (id, name, sex) values(1, 'AAA', 'female');
   mysql> insert into company.staff_timestamp (id, name, sex) values(2, 'BBB', 'female');
   先导入一部分数据：
   $ bin/sqoop import \
   --connect jdbc:mysql://bigdata113:3306/company \
   --username root \
   --password 000000 \
   --table staff_timestamp \
   --delete-target-dir \
   --hive-import \
   --fields-terminated-by "\t" \
   --m 1
   再增量导入一部分数据:
   mysql> insert into company.staff_timestamp (id, name, sex) values(3, 'CCC', 'female');
   $ bin/sqoop import \
   --connect jdbc:mysql://bigdata113:3306/company \
   --username root \
   --password 000000 \
   --table staff_timestamp \
   --check-column last_modified \
   --incremental lastmodified \
   --m 1 \
   --last-value "2019-05-17 09:50:12" \
   --append
   
   --last-value "2019-05-17 07:08:53" \
   ```

   **尖叫提示：**使用lastmodified方式导入数据要指定增量数据是要--append（追加）还是要--merge-key（合并）

   **尖叫提示：**在Hive中，如果不指定输出路径，可以去看以下两个目录

   1.  /user/root（此为用户名）
   2. /user/hive/warehouse  个人配置的目录

   **尖叫提示：**last-value指定的值是会包含于增量导入的数据中

   如果卡住，在yarn-site.xml中加入以下配置

   ```
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>20480</value>
    </property>
   
    <property>
       <name>yarn.scheduler.minimum-allocation-mb</name>
       <value>2048</value>
    </property>
   
    <property>
        <name>yarn.nodemanager.vmem-pmem-ratio</name>
        <value>2.1</value>
    </property>
   ```

2.  参数

   | ***\*序号\**** | ***\*参数\****                  | ***\*说明\****                                               |
   | -------------- | ------------------------------- | ------------------------------------------------------------ |
   | 1              | --append                        | 将数据追加到HDFS中已经存在的DataSet中，如果使用该参数，sqoop会把数据先导入到临时文件目录，再合并。 |
   | 2              | --as-avrodatafile               | 将数据导入到一个Avro数据文件中                               |
   | 3              | --as-sequencefile               | 将数据导入到一个sequence文件中                               |
   | 4              | --as-textfile                   | 将数据导入到一个普通文本文件中                               |
   | 5              | --boundary-query <statement>    | 边界查询，导入的数据为该参数的值（一条sql语句）所执行的结果区间内的数据。 |
   | 6              | --columns <col1, col2, col3>    | 指定要导入的字段                                             |
   | 7              | --direct                        | 直接导入模式，使用的是关系数据库自带的导入导出工具，以便加快导入导出过程。 |
   | 8              | --direct-split-size             | 在使用上面direct直接导入的基础上，对导入的流按字节分块，即达到该阈值就产生一个新的文件 |
   | 9              | --inline-lob-limit              | 设定大对象数据类型的最大值                                   |
   | 10             | --m或–num-mappers               | 启动N个map来并行导入数据，默认4个。                          |
   | 11             | --query或--e <statement>        | 将查询结果的数据导入，使用时必须伴随参--target-dir，--hive-table，如果查询中有where条件，则条件后必须加上$CONDITIONS关键字 |
   | 12             | --split-by <column-name>        | 按照某一列来切分表的工作单元，不能与--autoreset-to-one-mapper连用（请参考官方文档） |
   | 13             | --table <table-name>            | 关系数据库的表名                                             |
   | 14             | --target-dir <dir>              | 指定HDFS路径                                                 |
   | 15             | --warehouse-dir <dir>           | 与14参数不能同时使用，导入数据到HDFS时指定的目录             |
   | 16             | --where                         | 从关系数据库导入数据时的查询条件                             |
   | 17             | --z或--compress                 | 允许压缩                                                     |
   | 18             | --compression-codec             | 指定hadoop压缩编码类，默认为gzip(Use Hadoop codec default gzip) |
   | 19             | --null-string <null-string>     | string类型的列如果null，替换为指定字符串                     |
   | 20             | --null-non-string <null-string> | 非string类型的列如果null，替换为指定字符串                   |
   | 21             | --check-column <col>            | 作为增量导入判断的列名                                       |
   | 22             | --incremental <mode>            | mode：append或lastmodified                                   |
   | 23             | --last-value <value>            | 指定某一个值，用于标记增量导入的位置                         |

#### 6.2.6 命令&参数：export 

从HDFS（包括Hive和HBase）中将数据导出到关系型数据库中。

1. 命令：
   如：

   ```
   bin/sqoop export \
   --connect jdbc:mysql://bigdata113:3306/Andy \
   --username root \
   --password 000000 \
   --export-dir /user/hive/warehouse/staff_hive \
   --table aca \
   --num-mappers 1 \
   --input-fields-terminated-by "\t"
   
   ```

2. 参数：

   | ***\*序号\**** | ***\*参数\****                        | ***\*说明\****                                               |
   | -------------- | ------------------------------------- | ------------------------------------------------------------ |
   | 1              | --direct                              | 利用数据库自带的导入导出工具，以便于提高效率                 |
   | 2              | --export-dir <dir>                    | 存放数据的HDFS的源目录                                       |
   | 3              | -m或--num-mappers <n>                 | 启动N个map来并行导入数据，默认4个                            |
   | 4              | --table <table-name>                  | 指定导出到哪个RDBMS中的表                                    |
   | 5              | --update-key <col-name>               | 对某一列的字段进行更新操作                                   |
   | 6              | --update-mode <mode>                  | updateonlyallowinsert(默认)                                  |
   | 7              | --input-null-string <null-string>     | 请参考import该类似参数说明                                   |
   | 8              | --input-null-non-string <null-string> | 请参考import该类似参数说明                                   |
   | 9              | --staging-table <staging-table-name>  | 创建一张临时表，用于存放所有事务的结果，然后将所有事务结果一次性导入到目标表中，防止错误。 |
   | 10             | --clear-staging-table                 | 如果第9个参数非空，则可以在导出操作执行前，清空临时事务结果表 |

#### 6.2.7 命令&参数：codegen

将关系型数据库中的表映射为一个Java类，在该类中有各列对应的各个字段。

如：

```
$ bin/sqoop codegen \
--connect jdbc:mysql://bigdata113:3306/company \
--username root \
--password 000000 \
--table staff \
--bindir /opt/Desktop/staff \
--class-name Staff \
--fields-terminated-by "\t"
```

| ***\*序号\**** | ***\*参数\****                     | ***\*说明\****                                               |
| -------------- | ---------------------------------- | ------------------------------------------------------------ |
| 1              | --bindir <dir>                     | 指定生成的Java文件、编译成的class文件及将生成文件打包为jar的文件输出路径 |
| 2              | --class-name <name>                | 设定生成的Java文件指定的名称                                 |
| 3              | --outdir <dir>                     | 生成Java文件存放的路径                                       |
| 4              | --package-name <name>              | 包名，如com.z，就会生成com和z两级目录                        |
| 5              | --input-null-non-string <null-str> | 在生成的Java文件中，可以将null字符串或者不存在的字符串设置为想要设定的值（例如空字符串） |
| 6              | --input-null-string <null-str>     | 将null字符串替换成想要替换的值（一般与5同时使用）            |
| 7              | --map-column-java <arg>            | 数据库字段在生成的Java文件中会映射成各种属性，且默认的数据类型与数据库类型保持对应关系。该参数可以改变默认类型，例如：--map-column-java id=long, name=String |
| 8              | --null-non-string <null-str>       | 在生成Java文件时，可以将不存在或者null的字符串设置为其他值   |
| 9              | --null-string <null-str>           | 在生成Java文件时，将null字符串设置为其他值（一般与8同时使用） |
| 10             | --table <table-name>               | 对应关系数据库中的表名，生成的Java文件中的各个属性与该表的各个字段一一对应 |

#### 6.2.8 命令&参数：create-hive-table

生成与关系数据库表结构对应的hive表结构。

**命令：**

如：仅建表

```
$ bin/sqoop create-hive-table \
--connect jdbc:mysql://bigdata113:3306/company \
--username root \
--password 000000 \
--table staff \
--hive-table hive_staff1
```

参数：

| ***\*序号\**** | ***\*参数\****      | ***\*说明\****                                        |
| -------------- | ------------------- | ----------------------------------------------------- |
| 1              | --hive-home <dir>   | Hive的安装目录，可以通过该参数覆盖掉默认的Hive目录    |
| 2              | --hive-overwrite    | 覆盖掉在Hive表中已经存在的数据                        |
| 3              | --create-hive-table | 默认是false，如果目标表已经存在了，那么创建任务会失败 |
| 4              | --hive-table        | 后面接要创建的hive表                                  |
| 5              | --table             | 指定关系数据库的表名                                  |

#### 6.2.9 命令&参数：eval

可以快速的使用SQL语句对关系型数据库进行操作，经常用于在import数据之前，了解一下SQL语句是否正确，数据是否正常，并可以将结果显示在控制台。

**命令：**

```
$ bin/sqoop eval \
--connect jdbc:mysql://bigdata113:3306/company \
--username root \
--password 000000 \
--query "SELECT * FROM staff"
```

参数：

| ***\*序号\**** | ***\*参数\**** | ***\*说明\****    |
| -------------- | -------------- | ----------------- |
| 1              | --query或--e   | 后跟查询的SQL语句 |

#### 6.1.10 命令&参数：import-all-tables

可以将RDBMS中的所有表导入到HDFS中，每一个表都对应一个HDFS目录

**命令：**

如：注意：(卡住)

```
$ bin/sqoop import-all-tables \
--connect jdbc:mysql://bigdata113:3306/company \
--username root \
--password 000000 \
--hive-import \
--fields-terminated-by "\t"
```

参数：

| ***\*序号\**** | ***\*参数\****          | ***\*说明\****                         |
| -------------- | ----------------------- | -------------------------------------- |
| 1              | --as-avrodatafile       | 这些参数的含义均和import对应的含义一致 |
| 2              | --as-sequencefile       |                                        |
| 3              | --as-textfile           |                                        |
| 4              | --direct                |                                        |
| 5              | --direct-split-size <n> |                                        |
| 6              | --inline-lob-limit <n>  |                                        |
| 7              | --m或—num-mappers <n>   |                                        |
| 8              | --warehouse-dir <dir>   |                                        |
| 9              | -z或--compress          |                                        |
| 10             | --compression-codec     |                                        |

#### 6.2.11 命令&参数：job

用来生成一个sqoop任务，生成后不会立即执行，需要手动执行。

**命令：***

如：

```
$ bin/sqoop job \
 --create myjob -- import-all-tables \
 --connect jdbc:mysql://bigdata111:3306/company \
 --username root \
 --password 000000
$ bin/sqoop job \
--list
$ bin/sqoop job \
--exec myjob
```

**尖叫提示：**注意import-all-tables和它左边的--之间有一个空格

**尖叫提示：**如果需要连接metastore，则--meta-connect 

执行的结果在HDFS：/user/root/ 目录中，即导出所有表到/user/root中

参数：

| ***\*序号\**** | ***\*参数\****            | ***\*说明\****           |
| -------------- | ------------------------- | ------------------------ |
| 1              | --create <job-id>         | 创建job参数              |
| 2              | --delete <job-id>         | 删除一个job              |
| 3              | --exec <job-id>           | 执行一个job              |
| 4              | --help                    | 显示job帮助              |
| 5              | --list                    | 显示job列表              |
| 6              | --meta-connect <jdbc-uri> | 用来连接metastore服务    |
| 7              | --show <job-id>           | 显示一个job的信息        |
| 8              | --verbose                 | 打印命令运行时的详细信息 |

**尖叫提示：**在执行一个job时，如果需要手动输入数据库密码，可以做如下优化

```
<property>
	<name>sqoop.metastore.client.record.password</name>
	<value>true</value>
	<description>If true, allow saved passwords in the metastore.</description>
</property>
```

#### 6.2.12 命令&参数：list-databases

**命令：**

如：

```
$ bin/sqoop list-databases \
--connect jdbc:mysql://bigdata113:3306/ \
--username root \
--password 000000
```

**参数：**与公用参数一样

#### 6.2.13 命令&参数：list-tables

**命令：**

如：

```
$ bin/sqoop list-tables \
--connect jdbc:mysql://bigdata113:3306/company \
--username root \
--password 000000
```

**参数：**与公用参数一样

#### 6.2.14 命令&参数：merge

将HDFS中不同目录下面的数据合并在一起并放入指定目录中

数据环境：注意:以下数据自己手动改成\t

```
new_staff
1	AAA	male
2	BBB	male
3	CCC	male
4	DDD	male

old_staff
1	AAA	female
2	CCC	female
3	BBB	female
6	DDD	female
```

**尖叫提示：**上边数据的列之间的分隔符应该为\t，行与行之间的分割符为\n，如果直接复制，请检查之。

**命令：**

如：

```
创建JavaBean：
$ bin/sqoop codegen \
--connect jdbc:mysql://bigdata113:3306/company \
--username root \
--password 000000 \
--table staff \
--bindir /opt/Desktop/staff \
--class-name Staff \
--fields-terminated-by "\t"

开始合并：注：是hdfs路径
$ bin/sqoop merge \
--new-data /test/new/ \
--onto /test/old/ \
--target-dir /test/merged \
--jar-file /opt/Desktop/staff/Staff.jar \
--class-name Staff \
--merge-key id
结果：
1	AAA	MALE
2	BBB	MALE
3	CCC	MALE
4	DDD	MALE
6	DDD	FEMALE
```

参数：

| ***\*序号\**** | ***\*参数\****       | ***\*说明\****                                         |
| -------------- | -------------------- | ------------------------------------------------------ |
| 1              | --new-data <path>    | HDFS 待合并的数据目录，合并后在新的数据集中保留        |
| 2              | --onto <path>        | HDFS合并后，重复的部分在新的数据集中被覆盖             |
| 3              | --merge-key <col>    | 合并键，一般是主键ID                                   |
| 4              | --jar-file <file>    | 合并时引入的jar包，该jar包是通过Codegen工具生成的jar包 |
| 5              | --class-name <class> | 对应的表名或对象名，该class类是包含在jar包中的         |
| 6              | --target-dir <path>  | 合并后的数据在HDFS里存放的目录                         |

#### 6.2.15 命令&参数：metastore

记录了Sqoop job的元数据信息，如果不启动该服务，那么默认job元数据的存储目录为~/.sqoop，可在sqoop-site.xml中修改。

**命令：**

如：启动sqoop的metastore服务

```
$ bin/sqoop metastore
```

参数：

| ***\*序号\**** | ***\*参数\**** | ***\*说明\**** |
| -------------- | -------------- | -------------- |
| 1              | --shutdown     | 关闭metastore  |

