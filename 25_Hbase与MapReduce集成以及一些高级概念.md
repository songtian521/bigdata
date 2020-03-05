# 25_Hbase与MR集成以及一些高级概念

Hbase当中的数据最终都是存储在HDFS上面的，HBase天生的支持MR的操作，我们可以通过MR直接处理HBase当中的数据，并且MR可以将处理后的结果直接存储到HBase当中去

注意：我们可以使用TableMapper与TableReducer来实现从HBase当中读取与写入数据

http://hbase.apache.org/2.0/book.html#mapreduce

# 1.案例1

读取myuser这张表当中的数据写入到HBase的另外一张表当中去

这里我们将myuser这张表当中f1列族的name和age字段写入到myuser2这张表的f1列族当中去

## 1.1 创建myuser2这张表

注意：列簇的名字要与myuser表的列簇名字相同

```
create 'myuser2','f1'
```

## 1.2 创建maven工程，导入jar包

```xml
<dependencies>
        <!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-hdfs -->
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.8.4</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-common -->
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.8.4</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-client -->
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.8.4</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.hbase/hbase-mapreduce -->
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-mapreduce</artifactId>
            <version>2.0.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.hbase/hbase-server -->
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-server</artifactId>
            <version>2.0.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.hbase/hbase-client -->
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>2.0.0</version>
        </dependency>


        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>6.14.3</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                    <!--    <verbal>true</verbal>-->
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <minimizeJar>true</minimizeJar>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

        </plugins>
    </build>


```

## 1.3 开发MR程序

### 1.3.1 定义mapper类

```java
package MR;

import org.apache.hadoop.hbase.Cell;
import org.apache.hadoop.hbase.CellUtil;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.TableMapper;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.hadoop.io.Text;

import java.io.IOException;
import java.util.List;

/**
 * @Class:Hbase.MR.mapper
 * @Descript:
 * @Author:宋天
 * @Date:2020/1/6
 */

/**
 * 负责读取myuser表中的数据
 * 如果Mapper类需要读取hbase表数据，那么我们mapper类需要继承TableMapper类
 * 将key2 value2定义为text和put类型
 *
 * text里面装rowkey
 * put装需要插入的数据
 */
public class mapper extends TableMapper<Text, Put> {
    /**
     *
     * @param key rowkey
     * @param value result对象，封装了我们一条条的数据
     * @param context 上下文对象
     * @throws IOException
     * @throws InterruptedException
     *
     * 需求：读取myuser当中f1列簇相面的name和age列
     */
    @Override
    protected void map(ImmutableBytesWritable key, Result value, Context context) throws IOException, InterruptedException {
       //获取rowkey的字节数组
        byte[] bytes = key.get();
        String rowkey = Bytes.toString(bytes);
        Put put = new Put(bytes);

        //获取到所有的cell
        List<Cell> cells = value.listCells();
        for ( Cell cell : cells ){
            //获取cell对应的列簇
            byte[] familyBytes = CellUtil.cloneFamily(cell);
            //获取对应的列
            byte[] qualifierBytes = CellUtil.cloneQualifier(cell);
            //判断：只需要f1列簇下面name和age两个列
            if (Bytes.toString(familyBytes).equals("f1") && Bytes.toString(qualifierBytes).equals("name") || Bytes.toString(qualifierBytes).equals("age")){
                put.add(cell);
            }
        }
        //将数据写出去
        if (!put.isEmpty()){
            context.write(new Text(rowkey),put);
        }
    }
}

```

### 1.3.2 定义reducer类

```java
package MR;

import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.TableReducer;
import org.apache.hadoop.io.Text;

import java.io.IOException;

/**
 * @Class:Hbase.MR.reducer
 * @Descript:
 * @Author:宋天
 * @Date:2020/1/6
 */

/**
 * 负责将数据写入到myuser2
 */
public class reducer extends TableReducer<Text, Put, ImmutableBytesWritable> {
    @Override
    protected void reduce(Text key, Iterable<Put> values, Context context) throws IOException, InterruptedException {
        for (Put put :values){
            context.write(new ImmutableBytesWritable(key.toString().getBytes()),put);
        }

    }
}

```

### 1.3.3 定义程序运行的main方法

```java
package MR;


import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.client.Scan;
import org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;


/**
 * @Class:Hbase.MR.main
 * @Descript:
 * @Author:宋天
 * @Date:2020/1/6
 */
public class HbaseMain extends Configured implements Tool {

    @Override
    public int run(String[] strings) throws Exception {
        Job job = Job.getInstance(super.getConf(), "hbaseMR");
        //打包运行必须设置main方法所在的主类
        job.setJarByClass(HbaseMain.class);

        Scan scan = new Scan();
        //定义mapper类和reducer类
        /*
             String table, Scan scan,
             Class<? extends TableMapper> mapper,
             Class<?> outputKeyClass,
             Class<?> outputValueClass, Job job
         */
        TableMapReduceUtil.initTableMapperJob("myuser",scan,mapper.class, Text.class, Put.class,job,false);

        //使用工具类初始化reducer类
        /**
         * String table,
         Class<? extends TableReducer> reducer, Job job
         */
        TableMapReduceUtil.initTableReducerJob("myuser2",reducer.class,job);

        boolean b = job.waitForCompletion(true);

        return b?0:1;
    }

    //程序入口类
    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum","bigdata111:2181,bigdata222:2181,bigdata333:2181");

        int run = ToolRunner.run(conf,new HbaseMain(),args);
        System.exit(run);
    }

}

```

### 1.3.4 运行

1. 方式1：本地运行

   直接在idea中运行即可

2. 方式2：打包集群运行

   注意，我们需要使用打包插件，将HBase的依赖jar包都打入到工程jar包里面去

   ```xml
   <plugin>
               <groupId>org.apache.maven.plugins</groupId>
               <artifactId>maven-shade-plugin</artifactId>
               <version>2.4.3</version>
               <executions>
                   <execution>
                       <phase>package</phase>
                       <goals>
                           <goal>shade</goal>
                       </goals>
                       <configuration>
                           <minimizeJar>true</minimizeJar>
                       </configuration>
                   </execution>
               </executions>
           </plugin>
   ```

   main代码当中添加

   ```
   job.setJarByClass(HBaseMain.class);
   ```

   idea中点击package打包，并在target文件夹下找到两个大小不一的jar包

   两个jar包的运行方式如下：

   1. 使用命令运行hbase开头jar包

      ```
      yarn jar HBase-1.0-SNAPSHOT.jar MR.HbaseMain
      ```

   2. 配置环境变量运行original这个比较小的jar包

      ```
      export HADOOP_HOME=/opt/module/hadoop-2.8.4/
      export HBASE_HOME=/opt/module/hbase-2.0.0/
      export HADOOP_CLASSPATH=`${HBASE_HOME}/bin/hbase mapredcp
      ```

      soucre /etc/profile

      ```
      yarn jar original-hbaseStudy-1.0-SNAPSHOT.jar   MR.HbaseMain
      ```

# 2. 案例2

读取HDFS文件，写入到HBase表当中去

读取hdfs路径hbase_input/user.txt，然后将数据写入到myuser2这张表当中去

## 2.1 准备数据文件

准备数据文件，并将数据文件上传到HDFS上面去

```
hdfs dfs -mkdir /hbase_input
```

```
cd /opt/file
vim user.txt


0007    zhangsan        18
0008    lisi    25
0009    wangwu  20
```

注：自行检查分隔符是否为`\t`

上传

```
hdfs dfs -put user.txt /habse_input
```

## 2.2 开发mapper类

### 2.2.1 定义mapper类

```java
package MR2;

/**
 * @Class:HBase_MR.MR2.HDFSMapper
 * @Descript:
 * @Author:宋天
 * @Date:2020/1/7
 */

import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;

/**
 * 通过mapper读取hdfs上面的文件，然后进行处理
 */
public class HDFSMapper extends Mapper<LongWritable, Text,Text, NullWritable> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        //读取数据之后不做任何处理，直接将数据写入到reducer中处理
        context.write(value,NullWritable.get());
    }
}

```

### 2.2.2 定义reducer类

```java
package MR2;

import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.TableReducer;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;

import java.io.IOException;

/**
 * @Class:HBase_MR.MR2.HDFSReducer
 * @Descript:
 * @Author:宋天
 * @Date:2020/1/7
 */
public class HDFSReducer extends TableReducer<Text, NullWritable, ImmutableBytesWritable> {
    @Override
    protected void reduce(Text key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {

        String[] split = key.toString().split("\t");
        Put put = new Put(split[0].getBytes());
        put.addColumn("f1".getBytes(),"name".getBytes(),split[1].getBytes());
        put.addColumn("f1".getBytes(),"age".getBytes(),split[2].getBytes());
        //将数据写出去，key3是ImmutableBytesWritable，这个里面装的是rowke
        //然后将写出去的数据封装到put对象中
        context.write(new ImmutableBytesWritable(split[0].getBytes()),put);
    }
}

```

### 2.2.3 定义程序运行main方法

```java
package MR2;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

/**
 * @Class:HBase_MR.MR2.HDFSMain
 * @Descript:
 * @Author:宋天
 * @Date:2020/1/7
 */
public class HDFSMain extends Configured implements Tool {
    @Override
    public int run(String[] strings) throws Exception {
//        获取job对象
        Job job = Job.getInstance(super.getConf(), "HDFSMR");
//        第一步：读取文件，解析成key，value对
        job.setInputFormatClass(TextInputFormat.class);
        TextInputFormat.addInputPath(job,new Path("hdfs://bigdata111:9000/hbase_input"));
//        第二步：自定义map逻辑，接收k1，v1并转换为k2，v2进行输出
        job.setMapperClass(HDFSMapper.class);
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(NullWritable.class);
//        分区排序规约
//        第七步：设置reduce类
        TableMapReduceUtil.initTableReducerJob("myuser2",HDFSReducer.class,job);
        boolean b = job.waitForCompletion(true);


        return b?0:1;
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum","bigdata111:2181,bigdata222:2181,bigdata333:2181");
        int run = ToolRunner.run(conf, new HDFSMain(), args);
        System.exit(run);

    }
}

```

### 2.2.4 运行

运行方式和案例1相同，这里不再重复

# 3.案例3

**通过bulkload的方式批量加载数据到HBase当中去**

## 3.1 概念

> 加载数据到HBase当中去的方式多种多样，我们可以使用HBase的javaAPI或者使用sqoop将我们的数据写入或者导入到HBase当中去，但是这些方式不是慢就是在导入的过程的占用Region资源导致效率低下，我们也可以通过MR的程序，将我们的数据直接转换成HBase的最终存储格式HFile，然后直接load数据到HBase当中去即可
>
> HBase中每张Table在根目录（/HBase）下用一个文件夹存储，Table名为文件夹名，在Table文件夹下每个Region同样用一个文件夹存储，每个Region文件夹下的每个列族也用文件夹存储，而每个列族下存储的就是一些HFile文件，HFile就是HBase数据在HFDS下存储格式，所以HBase存储文件最终在hdfs上面的表现形式就是HFile，如果我们可以直接将数据转换为HFile的格式，那么我们的HBase就可以直接读取加载HFile格式的文件，就可以直接读取了

优点：

1. 导入过程不占用region资源
2. 能快速导入海量数据
3. 节省内存

![](img\HBase数据读写流程.png)

使用bulkload的方式将我们的数据直接生成HFile格式，然后直接加载到HBase的表当中去

![](img\bulkload方式加载数据到Hbase.png)

**需求：**将我们hdfs上面的这个路径/hbase_input/user.txt的数据文件，转换成HFile格式，然后load到myuser2这张表里面去

## 3.2 定义mapper类

```java
package MR3;

import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;

/**
 * @Class:HBase_MR.MR3.HDFSReadMapper
 * @Descript:
 * @Author:宋天
 * @Date:2020/1/7
 */
public class HDFSReadMapper extends Mapper<LongWritable, Text, ImmutableBytesWritable, Put> {

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

        String[] split = value.toString().split("\t");
        Put put = new Put(split[0].getBytes());
        put.addColumn("f1".getBytes(),"name".getBytes(),split[1].getBytes());
        put.addColumn("f1".getBytes(),"age".getBytes(),split[2].getBytes());

        context.write(new ImmutableBytesWritable(split[0].getBytes()),put);

    }
}

```

## 3.3定义main程序入口

```java
package MR3;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.client.Table;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.HFileOutputFormat2;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

/**
 * @Class:HBase_MR.MR3.HDFSReadMain
 * @Descript: 将txt文件转换为HFile文件，并上传至HDFS
 * @Author:宋天
 * @Date:2020/1/7
 */
public class HDFSReadMain extends Configured implements Tool {
    @Override
    public int run(String[] strings) throws Exception {
        Configuration conf = super.getConf();
        //获取job对象
        Job job = Job.getInstance(conf, "HDFSMR2");
        Connection connection = ConnectionFactory.createConnection(conf);
        Table table = connection.getTable(TableName.valueOf("myuser2"));
        //读取文件
        job.setInputFormatClass(TextInputFormat.class);
        TextInputFormat.addInputPath(job,new Path("hdfs://bigdata111:9000/hbase_input"));

        job.setMapperClass(HDFSReadMapper.class);
        job.setMapOutputKeyClass(ImmutableBytesWritable.class);
        job.setMapOutputValueClass(Put.class);

        //将数据输出为Hfile格式
//        Job job, Table table, RegionLocator regionLocator
//        配置增量的添加数据
        HFileOutputFormat2.configureIncrementalLoad(job,table,connection.getRegionLocator(TableName.valueOf("myuser2")));
//        设置输出class，决定输出格式
        job.setOutputFormatClass(HFileOutputFormat2.class);

//        输出路径
        HFileOutputFormat2.setOutputPath(job,new Path("hdfs://bigdata111:9000/hbase_output"));
        boolean b = job.waitForCompletion(true);
        return b?0:1;
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum","bigdata111:2181,bigdata222:2181,bigdata333:2181");
        int run = ToolRunner.run(conf, new HDFSReadMain(), args);
        System.exit(run);

    }
}

```

## 3.4 运行

如上

## 3.5 开发代码，加载数据

```java
package MR3;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Admin;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Table;
import org.apache.hadoop.hbase.mapreduce.LoadIncrementalHFiles;

/**
 * @Class:HBase_MR.MR3.LoadData
 * @Descript:将HDFS上的HFile文件加载到HBase中去
 * @Author:宋天
 * @Date:2020/1/7
 */
public class LoadData {
    public static void main(String[] args) throws Exception {
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.property.clientPort", "2181");
        configuration.set("hbase.zookeeper.quorum", "bigdata111,bigdata222,bigdata333");

        Connection connection =  ConnectionFactory.createConnection(configuration);
        Admin admin = connection.getAdmin();
        Table table = connection.getTable(TableName.valueOf("myuser2"));
        LoadIncrementalHFiles load = new LoadIncrementalHFiles(configuration);
        load.doBulkLoad(new Path("hdfs://bigdata111:9000/hbase_output"), admin,table,connection.getRegionLocator(TableName.valueOf("myuser2")));
    }

}
```

或者我们也可以通过命令行来进行加载数据

先将hbase的jar包添加到hadoop的classpath路径下

```
export HBASE_HOME=/opt/module/hbase-2.0.0/
export HADOOP_HOME=/opt/module/hadoop-2.8.4/
export HADOOP_CLASSPATH=`${HBASE_HOME}/bin/hbase mapredcp`
```

然后执行以下命令，将hbase的HFile直接导入到表myuser2当中来

```
yarn jar /opt/module/hbase-2.0.0/lib/hbase-server-1.2.0.jar completebulkload /hbase_output myuser2
```



# 4.Hbase与hive的对比

1. hive

   - 数据仓库工具

     hive的本质其实就相当于将HDFS中已经存储的文件在mysql中做了一个双射关系，以方便使用HQL去管理查询

   - 用于数据分析、清洗

     hive适用于离线的数据分析和清洗，延迟较高

   - 基于HDFS、MapReduce

     hive存储的海事局依旧在DataNode上，编写的HQL语句终将是转换为MapReduce代码执行

2. hbase

   - nosql数据库

     是一种面向列存储的非关系型数据库

   - 用于存储结构化和非结构化的数据

     适用于表单非关系型数据的存储，不适合做关联查询，类似join等操作

   - 基于HDFS

     数据持久化存储的体现形式是Hfile，存放在DataNode中，被ResionServer以region的形式进行管理

   - 延迟较低，接入在线业务使用

     面对大量的企业数据，Hbase可以直线单表大量数据的存储，同时提供了高效的数据访问速度

**总结：**hive与hbase

hive和Hbase是两种基于Hadoop的不同技术，Hive是一种类sql的引擎，并且运行MapReduce任务，Hbase 是一种在Hadoop之上的NOSQL的key/value数据库。这两种工具是可以同时使用的。就像用google来搜索，用facebook进行社交一样，hive可以用来进行统计查询，hbase可以用来进行实时查询，数据也可以从hive写到hbase，或者从hbase写会hive

# 5.hive与Hbase的整合

hive与我们的hbase各有千秋，各自有着不同的功能，但是归根结底，hive与hbase的数据最红都是存储在hdfs上面的，一般的我们为了存储磁盘的空间，不会将一份数据存储到多个地方，导致磁盘空间的浪费，我们可以直接将数据存入hbase，然后通过hive整合hbase直接使用sql语句分析hbase里面的数据即可，非常方便。

## 5.1 将hive分析结果的数据，保存到HBase当中去

1. 拷贝hbase的五个依赖jar包到hive的lib目录下

   将我们HBase的五个jar包拷贝到hive的lib目录下

   hbase的jar包都在/opt/module/hbase-2.0.0/lib

   ```
   hbase-client-2.0.0.jar         	  
   hbase-hadoop2-compat-2.0.0.jar
   hbase-hadoop-compat-2.0.0.jar
   hbase-it-2.0.0.jar    
   hbase-server-2.0.0.jar
   ```

   也可以通过创建软连接的方式来进行jar包的依赖

   ```
   ln -s /opt/module/hbase-2.0.0/lib/hbase-client-2.0.0.jar /opt/module/apache-hive-2.1.0-bin/lib/hbase-client-2.0.0.jar
   ln -s /opt/module/hbase-2.0.0/lib/hbase-hadoop2-compat-2.0.0.jar /opt/module/apache-hive-2.1.0-bin/lib/hbase-hadoop2-compat-2.0.0.jar
   ln -s /opt/module/hbase-2.0.0/lib/hbase-hadoop-compat-2.0.0.jar /opt/module/apache-hive-2.1.0-bin/lib/hbase-hadoop-compat-2.0.0.jar
   ln -s /opt/module/hbase-2.0.0/lib/hbase-it-2.0.0.jar /opt/module/apache-hive-2.1.0-bin/lib/hbase-it-2.0.0.jar
   ln -s /opt/module/hbase-2.0.0/lib/hbase-server-2.0.0.jar /opt/module/apache-hive-2.1.0-bin/lib/hbase-server-2.0.0.jar  
   ```

2. 修改hive的配置文件

   hive的配置文件hive-site.xml添加以下两行配置

   ```xml
   	<property>
                   <name>hive.zookeeper.quorum</name>
                   <value>bigdata111,bigdata222,bigdata333</value>
           </property>
   
            <property>
                   <name>hbase.zookeeper.quorum</name>
                   <value>bigdata111,bigdata222,bigdata333</value>
           </property>
   ```

3. 修改hive-env.sh配置文件添加以下配置

   ```
   cd /opt/module/hive-2.1.0/conf
   vim hive-env.sh
   ```

   ```
   export HADOOP_HOME=/opt/module/hadoop-2.8.4
   export HBASE_HOME=/opt/module/hbase-2.0.0
   export HIVE_CONF_DIR=/opt/module/hive-2.1.0/conf
   ```

4. hive当中建表，并加载一下数据

   ```
   cd /opt/module/hive-2.1.0/
   bin/hive
   ```

   创建hive数据库与hive对应的数据库表

   ```sql
   create database course;
   use course;
   create external table if not exists course.score(id int,cname string,score int) row format delimited fields terminated by '\t' stored as textfile ;
   ```

   准备数据如下：

   ```
   cd /opt/file
   vim hive-hbase.txt
   ```

   ```
   1       zhangsan        80
   2       lisi    60
   3       wangwu  30
   4       zhaoliu 70
   ```

   注：以`\t`分割

   进行加载数据

   进入hive客户端进行加载数据

   ```
   hive (course)> load data local inpath '/export/hive-hbase.txt' into table score;
   hive (course)> select * from score;
   ```

5. 创建hive管理表与HBase进行映射

   我们可以创建一个hive的管理表与hbase当中的表进行映射，hive管理表当中的数据，都会存储到hbase上面去

   hive当中创建内部表

   ```sql
   create table course.hbase_score(id int,cname string,score int)  
   stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'  
   with serdeproperties("hbase.columns.mapping" = "cf:name,cf:score") 
   tblproperties("hbase.table.name" = "hbase_score");
   ```

   通过insert  overwrite select  插入数据

   ```sql
   insert overwrite table course.hbase_score select id,cname,score from course.score;
   ```

6. hbase当中查看表hbase_score

   进入hbase的客户端查看表hbase_score，并查看当中的数据

   ```
   list
   ```

## 5.2 创建hive外部表，映射HBase当中已有的表模型

1. HBase当中创建表并手动插入加载一些数据

   进入HBase的shell客户端，手动创建一张表，并插入加载一些数据进去

   ```
   create 'hbase_hive_score',{ NAME =>'cf'}
   put 'hbase_hive_score','1','cf:name','zhangsan'
   put 'hbase_hive_score','1','cf:score', '95'
   put 'hbase_hive_score','2','cf:name','lisi'
   put 'hbase_hive_score','2','cf:score', '96'
   put 'hbase_hive_score','3','cf:name','wangwu'
   put 'hbase_hive_score','3','cf:score', '97'
   ```

2. 建立hive的外部表，映射HBase当中的表以及字段
   在hive当中建立外部表，
   进入hive客户端，然后执行以下命令进行创建hive外部表，就可以实现映射HBase当中的表数据

   ```sql
   CREATE external TABLE course.hbase2hive(id int, name string, score int) STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,cf:name,cf:score") TBLPROPERTIES("hbase.table.name" ="hbase_hive_score");
   ```



# 6.HBase的预分区

## 6.1 为什么要预分区

- 增加数据读写效率
- 负载均衡，防止数据倾斜
- 方便几区容灾调度region
- 优化map数量

## 6.2 如何预分区

每一个region维护着startRow与endRowkey，如果加入的数据符合某个region维护的rowkey范围，则该数据交给这个region维护

## 6.3 如何设定预分区

1. 直接设定

   ```
    create 'staff','info','partition1',SPLITS => ['1000','2000','3000','4000']
   ```

2. 使用16进制算法生成预分区

   ```
   create 'staff2','info','partition2',{NUMREGIONS => 15, SPLITALGO => 'HexStringSplit'}
   ```

3. 使用JavaAPI创建预分区

   ```java
   /**
        * 通过javaAPI进行HBase的表的创建以及预分区操作
        */
       @Test
       public void hbaseSplit() throws IOException {
           //获取连接
           Configuration configuration = HBaseConfiguration.create();
           configuration.set("hbase.zookeeper.quorum", "node01:2181,node02:2181,node03:2181");
           Connection connection = ConnectionFactory.createConnection(configuration);
           Admin admin = connection.getAdmin();
           //自定义算法，产生一系列Hash散列值存储在二维数组中
           byte[][] splitKeys = {{1,2,3,4,5},{'a','b','c','d','e'}};
   
   
           //通过HTableDescriptor来实现我们表的参数设置，包括表名，列族等等
           HTableDescriptor hTableDescriptor = new HTableDescriptor(TableName.valueOf("staff3"));
           //添加列族
           hTableDescriptor.addFamily(new HColumnDescriptor("f1"));
           //添加列族
           hTableDescriptor.addFamily(new HColumnDescriptor("f2"));
           admin.createTable(hTableDescriptor,splitKeys);
           admin.close();
   
       }
   ```

# 7.HBase的rowkey设计技巧

HBase是三维有序存储的，通过rowkey（行键），columnkey（column family和qualifier）和TimeStamp（时间戳）这三个维度可以对Hbase中的数据进行快速定位。

HBase中row可以可以唯一标识一行记录，在HBase查询的时候，有以下几种方式：

1. 通过get方式，指定rowkey获取唯一一条记录
2.  通过scan方式，设置startRow和stopRow参数进行范围匹配
3.  全表扫描，即直接扫描整张表中所有行记录

## 7.1 rowkey长度原则

rowkey是一个二进制码流，可以是任意字符串，最大长度64kb，实际应用中一般为10-100bytes，以byte[]形式保存，一般设计成定长。

**建议越短越好，不要超过16个字节**，原因如下：

1. 数据的持久化文件HFile中是按照KeyValue存储的，如果rowkey过长，比如超过100字节，1000w行数据，光rowkey就要占用100*1000w=10亿个字节，将近1G数据，这样会极大影响HFile的存储效率；
2. MemStore将缓存部分数据到内存，如果rowkey字段过长，内存的有效利用率就会降低，系统不能缓存更多的数据，这样会降低检索效率。

## 7.2 rowkey散列原则

如果rowkey按照时间戳的方式递增，不要将时间放在二进制码的前面，建议将rowkey的高位作为散列字段，由程序随机生成，低位放时间字段，这样将提高数据均衡分布在每个RegionServer，以实现负载均衡的几率。如果没有散列字段，首字段直接是时间信息，所有的数据都会集中在一个RegionServer上，这样在数据检索的时候负载会集中在个别的RegionServer上，造成热点问题，会降低查询效率。

## 7.3 rowkey唯一原则

必须在设计上保证其唯一性，rowkey是按照字典顺序排序存储的，因此，设计rowkey的时候，要充分利用这个排序的特点，将经常读取的数据存储到一块，将最近可能会被访问的数据放到一块。

## 7.4 什么是热点

HBase中的行是按照rowkey的字典顺序排序的，这种设计优化了scan操作，可以将相关的行以及会被一起读取的行存取在临近位置，便于scan。然而糟糕的rowkey设计是热点的源头。 

热点发生在大量的client直接访问集群的一个或极少数个节点（访问可能是读，写或者其他操作）。大量访问会使热点region所在的单个机器超出自身承受能力，引起性能下降甚至region不可用，这也会影响同一个RegionServer上的其他region，由于主机无法服务其他region的请求。 

设计良好的数据访问模式以使集群被充分，均衡的利用。为了避免写热点，设计rowkey使得不同行在同一个region，但是在更多数据情况下，数据应该被写入集群的多个region，而不是一个。下面是一些常见的避免热点的方法以及它们的优缺点：

1. 加盐

   这里所说的加盐不是密码学中的加盐，而是在rowkey的前面增加随机数，具体就是给rowkey分配一个随机前缀以使得它和之前的rowkey的开头不同。分配的前缀种类数量应该和你想使用数据分散到不同的region的数量一致。加盐之后的rowkey就会根据随机生成的前缀分散到各个region上，以避免热点。

2. 哈希

   哈希会使同一行永远用一个前缀加盐。哈希也可以使负载分散到整个集群，但是读却是可以预测的。使用确定的哈希可以让客户端重构完整的rowkey，可以使用get操作准确获取某一个行数据。

3. 反转

   第三种防止热点的方法时反转固定长度或者数字格式的rowkey。这样可以使得rowkey中经常改变的部分（最没有意义的部分）放在前面。这样可以有效的随机rowkey，但是牺牲了rowkey的有序性。

   反转rowkey的例子以手机号为rowkey，可以将手机号反转后的字符串作为rowkey，这样的就避免了以手机号那样比较固定开头导致热点问题

4. 时间戳反转

   一个常见的数据处理问题是快速获取数据的最近版本，使用反转的时间戳作为rowkey的一部分对这个问题十分有用，可以用 Long.Max_Value - timestamp 追加到key的末尾，例如  key reverse_timestamp  , [key] 的最新值可以通过scan [key]获得[key]的第一条记录，因为HBase中rowkey是有序的，第一条记录是最后录入的数据。

   **其他一些建议：**

   **尽量减少行键和列族的大小在HBase中，value永远和它的key一起传输的。**当具体的值在系统间传输时，它的rowkey，列名，时间戳也会一起传输。如果你的rowkey和列名很大，这个时候它们将会占用大量的存储空间。

   **列族尽可能越短越好，最好是一个字符。**

   **冗长的属性名虽然可读性好，但是更短的属性名存储在HBase中会更好。**

# 8.HBase的协处理器

http://hbase.apache.org/book.html#cp

## 8.1 起源

Hbase 作为列族数据库最经常被人诟病的特性包括：无法轻易建立“二级索引”，难以执 行求和、计数、排序等操作。比如，在旧版本的(<0.92)Hbase 中，统计数据表的总行数，需 要使用 Counter 方法，执行一次 MapReduce Job 才能得到。

虽然 HBase 在数据存储层中集成了 MapReduce，能够有效用于数据表的分布式计算。然而在很多情况下，做一些简单的相 加或者聚合计算的时候， 如果直接将计算过程放置在 server 端，能够减少通讯开销，从而获 得很好的性能提升。

于是， HBase 在 0.92 之后引入了协处理器(coprocessors)，实现一些激动人心的新特性：能够轻易建立二次索引、复杂过滤器(谓词下推)以及访问控制等。

## 8.2 协处理器有两种： observer 和 endpoint

**observer **

Observer 类似于传统数据库中的触发器，当发生某些事件的时候这类协处理器会被 Server 端调用。Observer Coprocessor 就是一些散布在 HBase Server 端代码中的 hook 钩子， 在固定的事件发生时被调用。比如： put 操作之前有钩子函数 prePut，该函数在 put 操作
执行前会被 Region Server 调用；在 put 操作之后则有 postPut 钩子函数

以 Hbase2.0.0 版本为例，它提供了三种观察者接口：

1. RegionObserver：提供客户端的数据操纵事件钩子： Get、 Put、 Delete、 Scan 等。
2. WALObserver：提供 WAL 相关操作钩子。
3. MasterObserver：提供 DDL-类型的操作钩子。如创建、删除、修改数据表等。

到 0.96 版本又新增一个 RegionServerObserver

下图是以 RegionObserver 为例子讲解 Observer 这种协处理器的原理：

![](img\hbase协处理器.png)

**Endpoint **

) Endpoint 协处理器类似传统数据库中的存储过程，客户端可以调用这些 Endpoint 协处 理器执行一段 Server 端代码，并将 Server 端代码的结果返回给客户端进一步处理，最常 见的用法就是进行聚集操作。如果没有协处理器，当用户需要找出一张表中的最大数据，即

max 聚合操作，就必须进行全表扫描，在客户端代码内遍历扫描结果，并执行求最大值的 操作。这样的方法无法利用底层集群的并发能力，而将所有计算都集中到 Client 端统一执 行，势必效率低下。利用 Coprocessor，用户可以将求最大值的代码部署到 HBase Server 端，
HBase 将利用底层 cluster 的多个节点并发执行求最大值的操作。即在每个 Region 范围内 执行求最大值的代码，将每个 Region 的最大值在 Region Server 端计算出，仅仅将该 max 值返回给客户端。在客户端进一步将多个 Region 的最大值进一步处理而找到其中的最大值。
这样整体的执行效率就会提高很多
下图是 EndPoint 的工作原理：

![](img\EndPoint 的工作原理.png)

**总结**

1. Observer 允许集群在正常的客户端操作过程中可以有不同的行为表现
2. Endpoint 允许扩展集群的能力，对客户端应用开放新的运算命令
3. observer 类似于 RDBMS 中的触发器，主要在服务端工作
4. endpoint 类似于 RDBMS 中的存储过程，主要在 client 端工作
5. observer 可以实现权限管理、优先级设置、监控、 ddl 控制、 二级索引等功能
6. endpoint 可以实现 min、 max、 avg、 sum、 distinct、 group by 等功能

## 8.3 协处理器加载方式  

 协处理器的加载方式有两种，我们称之为静态加载方式（ Static Load） 和动态加载方式 （ Dynamic Load）。 静态加载的协处理器称之为 System Coprocessor，动态加载的协处理器称 之为 Table Coprocessor

1. 静态加载 

   通过修改 hbase-site.xml 这个文件来实现， 启动全局 aggregation，能过操纵所有的表上 的数据。只需要添加如下代码：

   ```xml
   <property>
   <name>hbase.coprocessor.user.region.classes</name>
   <value>org.apache.hadoop.hbase.coprocessor.AggregateImplementation</value>
   </property>
   ```

   　为所有 table 加载了一个 cp class，可以用” ,”分割加载多个 class

2. 动态加载

   启用表 aggregation，只对特定的表生效。通过 HBase Shell 来实现。
   disable 指定表。 hbase> disable 'mytable'
   添加 aggregation

   ```
   hbase> alter 'mytable', METHOD => 'table_att','coprocessor'=>
   '|org.apache.Hadoop.hbase.coprocessor.AggregateImplementation||'
   ```

   重启指定表 hbase> enable 'mytable'

3. 协处理器卸载

   ![](img\卸载协处理器.png)

## 8.4 协处理器Observer应用实战

通过协处理器Observer实现hbase当中一张表插入数据，然后通过协处理器，将数据复制一份保存到另外一张表当中去，但是只取当第一张表当中的部分列数据保存到第二张表当中去

1. 第一步：HBase当中创建第一张表proc1

   在HBase当中创建一张表，表名user2，并只有一个列族info

   ```
   cd /export/servers/hbase-2.0.0/
   bin/hbase shell
   hbase(main):053:0> create 'proc1','info'
   ```

2. 第二步：Hbase当中创建第二张表proc2

   创建第二张表'proc2，作为目标表，将第一张表当中插入数据的部分列，使用协处理器，复制到'proc2表当中来

   ```
   hbase(main):054:0> create 'proc2','info'
   ```

3. 第三步：开发HBase的协处理器

   开发HBase的协处理器Copo

   ```java
   public class MyProcessor implements RegionObserver,RegionCoprocessor {
   
       static Connection connection = null;
       static Table table = null;
       static{
           Configuration conf = HBaseConfiguration.create();
           conf.set("hbase.zookeeper.quorum","bigdata111:2181");
           try {
               connection = ConnectionFactory.createConnection(conf);
               table = connection.getTable(TableName.valueOf("proc2"));
           } catch (Exception e) {
               e.printStackTrace();
           }
       }
       private RegionCoprocessorEnvironment env = null;
       private static final String FAMAILLY_NAME = "info";
       private static final String QUALIFIER_NAME = "name";
       //2.0加入该方法，否则无法生效
       @Override
       public Optional<RegionObserver> getRegionObserver() {
           // Extremely important to be sure that the coprocessor is invoked as a RegionObserver
           return Optional.of(this);
       }
       @Override
       public void start(CoprocessorEnvironment e) throws IOException {
           env = (RegionCoprocessorEnvironment) e;
       }
       @Override
       public void stop(CoprocessorEnvironment e) throws IOException {
           // nothing to do here
       }
       /**
        * 覆写prePut方法，在我们数据插入之前进行拦截，
        * @param e
        * @param put  put对象里面封装了我们需要插入到目标表的数据
        * @param edit
        * @param durability
        * @throws IOException
        */
       @Override
       public void prePut(final ObserverContext<RegionCoprocessorEnvironment> e,
                          final Put put, final WALEdit edit, final Durability durability)
               throws IOException {
           try {
               //通过put对象获取插入数据的rowkey
               byte[] rowBytes = put.getRow();
               String rowkey = Bytes.toString(rowBytes);
               //获取我们插入数据的name字段的值
               List<Cell> list = put.get(Bytes.toBytes(FAMAILLY_NAME), Bytes.toBytes(QUALIFIER_NAME));
               if (list == null || list.size() == 0) {
                   return;
               }
               //获取到info列族，name列对应的cell
               Cell cell2 = list.get(0);
   
               //通过cell获取数据值
               String nameValue = Bytes.toString(CellUtil.cloneValue(cell2));
               //创建put对象，将数据插入到proc2表里面去
               Put put2 = new Put(rowkey.getBytes());
               put2.addColumn(Bytes.toBytes(FAMAILLY_NAME), Bytes.toBytes(QUALIFIER_NAME),  nameValue.getBytes());
               table.put(put2);
               table.close();
           } catch (Exception e1) {
               return ;
           }
       }
   }
   ```

4. 第四步：将项目打成jar包，并上传到HDFS上面

   - 将我们的协处理器打成一个jar包，此处不需要用任何的打包插件即可，然后上传到hdfs

   - 将打好的jar包上传到linux的/opt/module/jars路径下

     ```
     cd /opt/module/jars
     mv original-hbase-1.0-SNAPSHOT.jar  processor.jar
     hdfs dfs -mkdir -p /processor
     hdfs dfs -put processor.jar /processor
     ```

5. 第五步：将打好的jar包挂载到proc1表当中去

   ```
   hbase(main):056:0> describe 'proc1'
   hbase(main):055:0> alter 'proc1',METHOD => 'table_att','Coprocessor'=>'hdfs://bigdata111:9000/processor/processor.jar|cn.itcast.hbasemr.demo4.MyProcessor|1001|'
   ```

   再次查看proc1表

   ```
   hbase(main):043:0> describe 'proc1'
   ```

   可以查看到我们的卸载器已经加载了

6. proc1表当中添加数据

   进入hbase-shell客户端，然后直接执行以下命令向proc1表当中添加数据

   ```
   put 'proc1','0001','info:name','zhangsan'
   put 'proc1','0001','info:age','28'
   put 'proc1','0002','info:name','lisi'
   put 'proc1','0002','info:age','25'
   ```

   我们会发现，proc2表当中也插入了数据，并且只有info列族，name列

   **注意：**如果需要卸载我们的协处理器，那么进入hbase的shell命令行，执行以下命令即可

   ```
   disable 'proc1'
   alter 'proc1',METHOD=>'table_att_unset',NAME=>'coprocessor$1'
   enable 'proc1'
   ```



# 9.HBase当中的二级索引的基本介绍

由于HBase的查询比较弱，如果需要实现类似于 `select  name,salary,count(1),max(salary) from user  group  by name,salary order  by  salary `等这样的复杂性的统计需求，基本上不可能，或者说比较困难，所以我们在使用HBase的时候，一般都会借助二级索引的方案来进行实现

HBase的一级索引就是rowkey，我们只能通过rowkey进行检索。如果我们相对hbase里面列族的列列进行一些组合查询，就需要采用HBase的二级索引方案来进行多条件的查询。 

1.  MapReduce方案 
2. ITHBASE（Indexed-Transanctional HBase）方案 
3.  IHBASE（Index HBase）方案 
4. Hbase Coprocessor(协处理器)方案 
5.  Solr+hbase方案
6. CCIndex（complementalclustering index）方案

常见的二级索引我们一般可以借助各种其他的方式来实现，例如Phoenix或者solr或者ES等

# 10.节点的管理

## 10.1 服役（commissioning）

当启动regionserver时，regionserver会向HMaster注册并开始接收本地数据，开始的时候，新加入的节点不会有任何数据，平衡器开启的情况下，将会有新的region移动到开启的RegionServer上。如果启动和停止进程是使用ssh和HBase脚本，那么会将新添加的节点的主机名加入到conf/regionservers文件中。

```
./bin/hbase-daemon.sh stop regionserver
hbase(main):001:0>balance_switch true
```

## 10.2 退役（decommissioning）

顾名思义，就是从当前HBase集群中删除某个RegionServer，这个过程分为如下几个过程：

在0.90.2之前，我们只能通过在要卸载的节点上执行

1. 停止负载平衡器

   ```
   hbase> balance_switch false
   ```

2. 在退役节点上停止RegionServer

   ```
   [root@bigdata11 hbase-1.3.1] hbase-daemon.sh stop regionserver
   ```

3. RegionServer一旦停止，会关闭维护的所有region

4. Zookeeper上的该RegionServer节点消失

5. Master节点检测到该RegionServer下线，开启平衡器

6. 下线的RegionServer的region服务得到重新分配

这种方法很大的一个缺点是该节点上的Region会离线很长时间。因为假如该RegionServer上有大量Region的话，因为Region的关闭是顺序执行的，第一个关闭的Region得等到和最后一个Region关闭并Assigned后一起上线。这是一个相当漫长的时间。每个Region Assigned需要4s，也就是说光Assigned就至少需要2个小时。该关闭方法比较传统，需要花费一定的时间，而且会造成部分region短暂的不可用。

**另一种方案：**

1. 新方法

   自0.90.2之后，HBase添加了一个新的方法，即“graceful_stop”,只需要在HBase Master节点执行

   ```
   bin/graceful_stop.sh <RegionServer-hostname>
   ```

   该命令会自动关闭Load Balancer，然后Assigned Region，之后会将该节点关闭。除此之外，你还可以查看remove的过程，已经assigned了多少个Region，还剩多少个Region，每个Region 的Assigned耗时

2. 开启负载平衡器

   ```
   hbase> balance_switch false
   ```

   