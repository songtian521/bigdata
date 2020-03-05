# 27_HBase调优及实战

# 1.通用优化

1. NameNode的元数据备份使用SSD

2. 定时备份NameNode上的元数据，每小时或者每天备份，如果数据极其重要，可以5~10分钟备份一次。备份可以通过定时任务复制元数据目录即可。

3. 为NameNode指定多个元数据目录，使用dfs.name.dir或者dfs.namenode.name.dir指定。一个指定本地磁盘，一个指定网络磁盘。这样可以提供元数据的冗余和健壮性，以免发生故障。

4. 设置dfs.namenode.name.dir.restore为true，允许尝试恢复之前失败的dfs.namenode.name.dir目录，在创建checkpoint时做此尝试，如果设置了多个磁盘，建议允许。

5. NameNode节点必须配置为RAID1（镜像盘）结构。

6. 补充：什么是Raid0、Raid0+1、Raid1、Raid5

   - Standalone
     最普遍的单磁盘储存方式。

   - Cluster

     集群储存是通过将数据分布到集群中各节点的存储方式,提供单一的使用接口与界面,使用户可以方便地对所有数据进行统一使用与管理。

   - Hot swap

     用户可以再不关闭系统,不切断电源的情况下取出和更换硬盘,提高系统的恢复能力、拓展性和灵活性。

   - Raid0

     Raid0是所有raid中存储性能最强的阵列形式。其工作原理就是在多个磁盘上分散存取连续的数据,这样,当需要存取数据是多个磁盘可以并排执行,每个磁盘执行属于它自己的那部分数据请求,显著提高磁盘整体存取性能。但是不具备容错能力,适用于低成本、低可靠性的台式系统。

   - Raid1

     又称镜像盘,把一个磁盘的数据镜像到另一个磁盘上,采用镜像容错来提高可靠性,具有raid中最高的数据冗余能力。存数据时会将数据同时写入镜像盘内,读取数据则只从工作盘读出。发生故障时,系统将从镜像盘读取数据,然后再恢复工作盘正确数据。这种阵列方式可靠性极高,但是其容量会减去一半。广泛用于数据要求极严的应用场合,如商业金融、档案管理等领域。只允许一颗硬盘出故障。

   - Raid0+1

     将Raid0和Raid1技术结合在一起,兼顾两者的优势。在数据得到保障的同时,还能提供较强的存储性能。不过至少要求4个或以上的硬盘，但也只允许一个磁盘出错。是一种三高技术。

   - Raid5

     Raid5可以看成是Raid0+1的低成本方案。采用循环偶校验独立存取的阵列方式。将数据和相对应的奇偶校验信息分布存储到组成RAID5的各个磁盘上。当其中一个磁盘数据发生损坏后,利用剩下的磁盘和相应的奇偶校验信息 重新恢复/生成丢失的数据而不影响数据的可用性。至少需要3个或以上的硬盘。适用于大数据量的操作。成本稍高、储存性强、可靠性强的阵列方式。

     RAID还有其他方式，请自行查阅。

7. 保持NameNode日志目录有足够的空间，这些日志有助于帮助你发现问题。

8. 因为Hadoop是IO密集型框架，所以尽量提升存储的速度和吞吐量（类似位宽）。

# 2.Linux优化

1. 开启文件系统的预读缓存可以提高读取速度

   ```
    sudo blockdev --setra 32768 /dev/sda
   ```

   （尖叫提示：ra是readahead的缩写）

2. 关闭进程睡眠池

   ```
   sudo sysctl -w vm.swappiness=0
   ```

3. 调整ulimit上限，默认值为比较小的数字

   ```
   ulimit -n 查看允许最大进程数
   ulimit -u 查看允许打开最大文件数
   ```

   修改

   ```
   sudo vi /etc/security/limits.conf 修改打开文件数限制
   末尾添加：
   *                soft    nofile          1024000
   *                hard    nofile          1024000
   Hive             -       nofile          1024000
   hive             -       nproc           1024000 
   
   $ sudo vi /etc/security/limits.d/20-nproc.conf 修改用户打开进程数限制
   修改为：
   #*          soft    nproc     4096
   #root       soft    nproc     unlimited
   *          soft    nproc     40960
   root       soft    nproc     unlimited
   ```

4. 开启集群的时间同步NTP，请参看之前文档

5. 更新系统补丁（尖叫提示：更新补丁前，请先测试新版本补丁对集群节点的兼容性）

# 3. HDFS优化（hdfs-site.xml）

1. 保证RPC调用会有较多的线程数

   **属性：**dfs.namenode.handler.count

   **解释：**该属性是NameNode服务默认线程数，的默认值是10，根据机器的可用内存可以调整为50~100

    

   **属性：**dfs.datanode.handler.count

   **解释：**该属性默认值为10，是DataNode的处理线程数，如果HDFS客户端程序读写请求比较多，可以调高到15~20，设置的值越大，内存消耗越多，不要调整的过高，一般业务中，5~10即可。

2. 副本数的调整

   **属性：**dfs.replication

   **解释：**如果数据量巨大，且不是非常之重要，可以调整为2~3，如果数据非常之重要，可以调整为3~5。

3. 文件块大小的调整

   **属性：**dfs.blocksize
   **解释：**块大小定义，该属性应该根据存储的大量的单个文件大小来设置，如果大量的单个文件都小于100M，建议设置成64M块大小，对于大于100M或者达到GB的这种情况，建议设置成256M，一般设置范围波动在64M~256M之间。

# 4.MapReduce优化（mapred-site.xml）

1. Job任务服务线程数调整

   mapreduce.jobtracker.handler.count

   该属性是Job任务线程数，默认值是10，根据机器的可用内存可以调整为50~100

2. Http服务器工作线程数
   **属性：**mapreduce.tasktracker.http.threads
   **解释：**定义HTTP服务器工作线程数，默认值为40，对于大集群可以调整到80~100

3. 文件排序合并优化
   **属性：**mapreduce.task.io.sort.factor
   **解释：**文件排序时同时合并的数据流的数量，这也定义了同时打开文件的个数，默认值为10，如果调高该参数，可以明显减少磁盘IO，即减少文件读取的次数。

4. 设置任务并发

   **属性：**mapreduce.map.speculative
   **解释：**该属性可以设置任务是否可以并发执行，如果任务多而小，该属性设置为true可以明显加快任务执行效率，但是对于延迟非常高的任务，建议改为false，这就类似于迅雷下载。

5. MR输出数据的压缩
   **属性：**mapreduce.map.output.compress、mapreduce.output.fileoutputformat.compress
   **解释：**对于大集群而言，建议设置Map-Reduce的输出为压缩的数据，而对于小集群，则不需要。

6. 优化Mapper和Reducer的个数

   **属性：**
   mapreduce.tasktracker.map.tasks.maximum
   mapreduce.tasktracker.reduce.tasks.maximum
   **解释：**以上两个属性分别为一个单独的Job任务可以同时运行的Map和Reduce的数量。

   > 设置上面两个参数时，需要考虑CPU核数、磁盘和内存容量。假设一个8核的CPU，业务内容非常消耗CPU，那么可以设置map数量为4，如果该业务不是特别消耗CPU类型的，那么可以设置map数量为40，reduce数量为20。这些参数的值修改完成之后，一定要观察是否有较长等待的任务，如果有的话，可以减少数量以加快任务执行，如果设置一个很大的值，会引起大量的上下文切换，以及内存与磁盘之间的数据交换，这里没有标准的配置数值，需要根据业务和硬件配置以及经验来做出选择。
   >
   > 在同一时刻，不要同时运行太多的MapReduce，这样会消耗过多的内存，任务会执行的非常缓慢，我们需要根据CPU核数，内存容量设置一个MR任务并发的最大值，使固定数据量的任务完全加载到内存中，避免频繁的内存和磁盘数据交换，从而降低磁盘IO，提高性能。
   >
   > 大概配比：
   >
   > | CPU CORE | MEM（GB） | Map  | Reduce |
   > | -------- | --------- | ---- | ------ |
   > | 1        | 1         | 1    | 1      |
   > | 1        | 5         | 1    | 1      |
   > | 4        | 5         | 1~4  | 2      |
   > | 16       | 32        | 16   | 8      |
   > | 16       | 64        | 16   | 8      |
   > | 24       | 64        | 24   | 12     |
   > | 24       | 128       | 24   | 12     |
   >
   > 大概估算公式：
   >
   > map = 2 + ⅔cpu_core
   >
   > reduce = 2 + ⅓cpu_core

# 5.HBase优化

1. 在HDFS的文件中追加内容

   **属性：**dfs.support.append
   **文件：**hdfs-site.xml、hbase-site.xml
   **解释：**开启HDFS追加同步，可以优秀的配合HBase的数据同步和持久化。默认值为true。

2. 优化DataNode允许的最大文件打开数
   **属性：**dfs.datanode.max.transfer.threads
   **文件：**hdfs-site.xml
   **解释：**HBase一般都会同一时间操作大量的文件，根据集群的数量和规模以及数据动作，设置为4096或者更高。默认值：4096

3. 优化延迟高的数据操作的等待时间

   **属性：**dfs.image.transfer.timeout
   **文件：**hdfs-site.xml
   **解释：**如果对于某一次数据操作来讲，延迟非常高，socket需要等待更长的时间，建议把该值设置为更大的值（默认60000毫秒），以确保socket不会被timeout掉。

4. 优化数据的写入效率
   **属性：**
   mapreduce.map.output.compress
   mapreduce.map.output.compress.codec
   **文件：**mapred-site.xml
   **解释**：开启这两个数据可以大大提高文件的写入效率，减少写入时间。第一个属性值修改为true，第二个属性值修改为：org.apache.hadoop.io.compress.GzipCodec

5. 优化DataNode存储

   **属性：**dfs.datanode.failed.volumes.tolerated

   **文件：**hdfs-site.xml

   **解释：**默认为0，意思是当DataNode中有一个磁盘出现故障，则会认为该DataNode shutdown了。如果修改为1，则一个磁盘出现故障时，数据会被复制到其他正常的DataNode上，当前的DataNode继续工作。

6. 设置RPC监听数量
   **属性：**hbase.regionserver.handler.count
   **文件：**hbase-site.xml
   **解释：**默认值为30，用于指定RPC监听的数量，可以根据客户端的请求数进行调整，读写请求较多时，增加此值。 

7. 优化HStore文件大小
   **属性：**hbase.hregion.max.filesize
   **文件：**hbase-site.xml
   **解释：**默认值10737418240（10GB），如果需要运行HBase的MR任务，可以减小此值，因为一个region对应一个map任务，如果单个region过大，会导致map任务执行时间过长。该值的意思就是，如果HFile的大小达到这个数值，则这个region会被切分为两个Hfile。

8. 优化hbase客户端缓存
   **属性：**hbase.client.write.buffer
   **文件：**hbase-site.xml
   **解释：**用于指定HBase客户端缓存，增大该值可以减少RPC调用次数，但是会消耗更多内存，反之则反之。一般我们需要设定一定的缓存大小，以达到减少RPC次数的目的。

9. 指定scan.next扫描HBase所获取的行数
   **属性：**hbase.client.scanner.caching
   **文件：**hbase-site.xml
   **解释：**用于指定scan.next方法获取的默认行数，值越大，消耗内存越大。

# 6. 内存优化

   HBase操作过程中需要大量的内存开销，毕竟Table是可以缓存在内存中的，一般会分配整个可用内存的70%给HBase的Java堆。但是不建议分配非常大的堆内存，因为GC过程持续太久会导致RegionServer处于长期不可用状态，一般16~48G内存就可以了，如果因为框架占用内存过高导致系统内存不足，框架一样会被系统服务拖死。

# 7.JVM优化

涉及文件：hbase-env.sh

1. 并行GC
   **参数：**-XX:+UseParallelGC
   **解释：**开启并行GC
2. 同时处理垃圾回收的线程数
   **参数：**-XX:ParallelGCThreads=cpu_core – 1
   **解释：**该属性设置了同时处理垃圾回收的线程数。
3. 禁用手动GC
   **参数：**-XX:DisableExplicitGC
   **解释：**防止开发人员手动调用GC

# 8.Zookeeper优化

**优化Zookeeper会话超时时间**

**参数：**zookeeper.session.timeout

**文件：**hbase-site.xml

**解释：**In hbase-site.xml, set zookeeper.session.timeout to 30 seconds or less to bound failure detection (20-30 seconds is a good start).该值会直接关系到master发现服务器宕机的最大周期，默认值为30秒，如果该值过小，会在HBase在写入大量数据发生而GC时，导致RegionServer短暂的不可用，从而没有向ZK发送心跳包，最终导致认为从节点shutdown。一般20台左右的集群需要配置5台zookeeper。

# 9.高可用

在HBase中Hmaster负责监控RegionServer的生命周期，均衡RegionServer的负载，如果Hmaster挂掉了，那么整个HBase集群将陷入不健康的状态，并且此时的工作状态并不会维持太久。所以HBase支持对Hmaster的高可用配置。

1. 关闭HBase集群（如果没有开启则跳过此步）

   ```
   stop-hbase.sh
   ```

2. 在conf目录下创建backup-masters文件

   ```
    touch conf/backup-masters
   ```

3. 在backup-masters文件中配置高可用HMaster节点

   ```
   echo bigdata112 >  conf/backup-masters
   ```

4. 将整个conf目录scp到其他节点

   ```
   scp -r conf/ bigdata112:/opt/module/hbase-1.3.1
   scp -r conf/ bigdata113:/opt/module/hbase-1.3.1
   ```

5. 重新启动HBase后打开页面测试查看

   http://bigdata111:16010

注：之前配置的时候已经配置了高可用

# 10.HBase实战之微博

## 10.1 hbase的namespace介绍

### 10.1.1 namespace基本介绍

在Hbase中，namespace命名空间指一组表的逻辑分组，类似RDBMS中的database，方便对表在业务上划分。

Apache HBase从0.98.0, 0.95.2两个版本号開始支持namespace级别的授权操作，HBase全局管理员能够创建、改动和回收namespace的授权。

### 10.1.2 namespace的作用

1. 配额管理：限制一个namespace可以使用的资源，包括region和table
2. 命名空间安全管理：提供了另一个层面的多租户安全管理
3. Region服务器组：一个命名或一张表，可以被固定到一组RegionServers上，从而保证了数据的隔离性

### 10.1.3 namespace的基本操作

1. 创建namespace

   ```
   create_namespace 'nametest'
   ```

2. 查看namespace

   ```
   describe_namespace 'nametest'
   ```

3. 列出所有的namespace

   ```
   list_namespace
   ```

4. 在namespace下创建表

   ```
   create 'nametest:testtable','fm1'
   ```

5. 查看namespace下的表

   ```
   list_namespace_tables 'nametest'
   ```

6. 删除namespace

   ```
   drop_namespace 'nametest'
   ```

   注：删除namespace前需要保证其内无表。不然会删除失败

   namespace下有表时删除：

   ```
   disable "weibo:content"
   drop "weibo:content"
   drop_namespace 'weibo'
   ```

## 10.2 Hbase的数据版本的确界以及TTL

### 10.2.1 数据的确界

在hbase当中，我们可以为数据设置上界和下界，其实就是定义数据的历史版本保留多少个，通过自定义历史版本的保存数量，我们可以实现历史多个版本的数据的查询

1. 版本的下界

   默认的版本下界是0，即禁用。row版本使用的最小数目是与生存时间（TTL Time To Live）相结合的，并且我们根据实际需求可以有0或者更多的版本，使用0，即只有一个版本的值写入cell

2. 版本的上界

   之前默认的版本上界是3，也就是一个row保留3个副本（基于时间戳的插入）。该值不要设计的过大，一般的业务不会超过100。如果cell中存储的数据版本号超过了3个，再次插入数据时，最新的值会将最老的值覆盖。（先版本默认为1）

### 10.2.2 数据的TTL

在实际工作当中经常会遇到有些数据过了一段时间我们可能就不在需要了，name这个时候我们可以使用定时任务去定时的删除这些数据，或者我们也可以使用Hbase的TTL功能，让我们的数据定期的会进行清除

使用代码来设置数据的确界以及设置数据的TTL如下：

1. 创建maven，导入jar包

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

2. 代码开发

   ```java
   package HBaseVersion;
   
   import org.apache.hadoop.conf.Configuration;
   import org.apache.hadoop.hbase.*;
   import org.apache.hadoop.hbase.client.*;
   import org.apache.hadoop.hbase.util.Bytes;
   
   import java.io.IOException;
   
   /**
    * @Class:HBase_weibo.HBaseVersion.HBaseVersionAndTTL
    * @Descript:
    * @Author:宋天
    * @Date:2020/1/10
    */
   public class HBaseVersionAndTTL {
   
       public static void main(String[] args) throws IOException, InterruptedException {
   //        操作hbase，向hbase表当中添加一条数据。并且设置数据的上界以及下界，设置数据的TTL过期时间
           Configuration conf = HBaseConfiguration.create();
           conf.set("hbase.zookeeper.quorum","bigdata111:2181,bigdata222:2181,bigdata333:2181");
   //        获取连接
           Connection connection = ConnectionFactory.createConnection(conf);
   //        创建hbase表
           Admin admin = connection.getAdmin();
   //        判断如果hbase表不存在，则创建
           if (!admin.tableExists(TableName.valueOf("version_hbase"))){
               //指定表名
               HTableDescriptor hTableDescriptor = new HTableDescriptor(TableName.valueOf("version_hbase"));
               //指定列簇名
               HColumnDescriptor f1 = new HColumnDescriptor("f1");
               //针对列簇设置版本的上界以及版本下界
               f1.setMinVersions(3);
               f1.setMaxVersions(5);
               f1.setTimeToLive(30);//设置f1列簇所有列的存活时间为30秒
               hTableDescriptor.addFamily(f1);
               admin.createTable(hTableDescriptor);
   
           }
           Table version_hbase = connection.getTable(TableName.valueOf("version_hbase"));
   
   
   //注释的内容是为了添加一些数据
   //        Put put = new Put("1".getBytes());
   ////        如果需要保存多个版本，需要带上时间戳
   //        put.addColumn("f1".getBytes(),"name".getBytes(),System.currentTimeMillis(),"lisi1".getBytes());
   //        Thread.sleep(1000);
   //
   //        Put put2 = new Put("1".getBytes());
   //        put2.addColumn("f1".getBytes(),"name".getBytes(),System.currentTimeMillis(),"lisi2".getBytes());
   //        Thread.sleep(1000);
   //
   //        Put put3 = new Put("1".getBytes());
   //        put3.addColumn("f1".getBytes(),"name".getBytes(),System.currentTimeMillis(),"lisi3".getBytes());
   //        Thread.sleep(1000);
   //
   //        Put put4 = new Put("1".getBytes());
   //        put4.addColumn("f1".getBytes(),"name".getBytes(),System.currentTimeMillis(),"lisi4".getBytes());
   //        Thread.sleep(1000);
   //
   //        Put put5 = new Put("1".getBytes());
   //        put5.addColumn("f1".getBytes(),"name".getBytes(),System.currentTimeMillis(),"lisi5".getBytes());
   //        Thread.sleep(1000);
   //
   //        Put put6 = new Put("1".getBytes());
   ////        针对某一条数据设置过期时间
   //        put6.setTTL(3000);
   //        put6.addColumn("f1".getBytes(),"name".getBytes(),System.currentTimeMillis(),"lisi6".getBytes());
   //
   //        version_hbase.put(put);
   //        version_hbase.put(put2);
   //        version_hbase.put(put3);
   //        version_hbase.put(put4);
   //        version_hbase.put(put5);
   //        version_hbase.put(put6);
   
           Get get = new Get("1".getBytes());
   //        不带参数表示将数据的所有版本获取到，带上参数获取指定的版本数据
           get.setMaxVersions();
           Result result = version_hbase.get(get);
   //        获取所有的cell
           Cell[] cells = result.rawCells();
           for (Cell cell : cells){
               System.out.println(Bytes.toString(CellUtil.cloneValue(cell)));
           }
   
           version_hbase.close();
           connection.close();
       }
   }
   
   ```

## 10.3 HBase微博实战案例

### 10.3.1 前期准备

1. 需求分析

   - 微博内容的浏览，数据库表的设计
   - 用户社交体现，关注用户，取关用户
   - 拉取关注的人的微博内容

2. 创建maven工程，并导入jar包

   jar同上

3. 将服务器上的配置文件拷贝到maven工程的resources文件夹下

   - core-site.xml
   - hbase-site.xml
   - hdfs-site.xml

   以上文件都在hbase的安装目录下的conf目录中

4. 代码设计总览

   - 创建命名空间以及表名的定义
   -  创建微博内容表
   - 创建用户关系表
   - 创建用户微博内容接收邮件表
   - 发布微博内容
   - 添加关注用户
   - 移除（取关）用户
   - 获取关注的人的微博内容

### 10.3.2 代码实现

1. 创建命名空间以及表名的定义

   ```java
       /**
        * 初始化命名空间
        */
       public void initNameSpace() throws IOException {
   //        连接hbase集群
           Connection connection = getConnection();
   //        获取客户端管理员对象
           Admin admin = connection.getAdmin();
   //        通过管理员对象创建命名空间
           NamespaceDescriptor namespaceDescriptor = NamespaceDescriptor.create("weibo").addConfiguration("creator","jim").build();
           admin.createNamespace(namespaceDescriptor);
           admin.close();
           connection.close();
       }
   
       public Connection getConnection() throws IOException {
   //        连接hbase集群
           Configuration configuration = HBaseConfiguration.create();
           configuration.set("hbase.zookeeper.quorum","bigdata111:2181,bigdata222:2181,bigdata333:2181");
           Connection connection = ConnectionFactory.createConnection();
           return connection;
       }
   
   ```

2. 创建微博内容表

   - 表结构

     | ***\*方法名\**** | creatTableeContent |
     | ---------------- | ------------------ |
     | Table Name       | weibo:content      |
     | RowKey           | 用户ID_时间戳      |
     | ColumnFamily     | info               |
     | ColumnLabel      | 标题,内容,图片     |
     | Version          | 1个版本            |

   - 代码实现

     ```java
      /**
          * 创建微博内容存储表
          *
          * 方法名 creatTableeContent
          Table Name    weibo:content
          RowKey    用户ID_时间戳
          ColumnFamily  info
          ColumnLabel   标题,内容,图片
          Version   1个版本
     
          *
          */
         public void creatTableeContent() throws IOException {
             //获取连接
             Connection connection = getConnection();
             //获取管理员对象
             Admin admin = connection.getAdmin();
     
             if (!admin.tableExists(TableName.valueOf(TABLE_CONTENT))){
                 //通过管理员对象来创建表
                 HTableDescriptor hTableDescriptor = new HTableDescriptor(TableName.valueOf(TABLE_CONTENT));
                 //添加列族info
                 HColumnDescriptor info = new HColumnDescriptor("info");
                 //设置版本确界
                 info.setMaxVersions(1);
                 info.setMinVersions(1);
                 //设置数据压缩
                 //   info.setCompressionType(Compression.Algorithm.SNAPPY);
                 info.setBlocksize(2048*1024);
                 info.setBlockCacheEnabled(true);
                 hTableDescriptor.addFamily(info);
                 //创建表
                 admin.createTable(hTableDescriptor);
             }
     
     
     
             admin.close();
             connection.close();
         }
     ```

3. 创建用户关系表

   - 表结构

     | ***\*方法名\**** | createTableRelations   |
     | ---------------- | ---------------------- |
     | Table Name       | weibo:relations        |
     | RowKey           | 用户ID                 |
     | ColumnFamily     | attends、fans          |
     | ColumnLabel      | 关注用户ID，粉丝用户ID |
     | ColumnValue      | 用户ID                 |
     | Version          | 1个版本                |

   - 代码实现

     ```java
     /**
          * 创建用户关系表
          * 方法名 createTableRelations
          Table Name    weibo:relations
          RowKey    用户ID
          ColumnFamily  attends、fans
          ColumnLabel   关注用户ID，粉丝用户ID
          ColumnValue   用户ID
          Version   1个版本
     
          */
         public void createTableRelations() throws IOException {
             //获取连接
             Connection connection = getConnection();
             //获取管理员对象
             Admin admin = connection.getAdmin();
     //        不存在的时候创建
             if (!admin.tableExists(TableName.valueOf(TABLE_RELATIONS))){
                 //通过管理对象创建表
                 HTableDescriptor hTableDescriptor = new HTableDescriptor(TableName.valueOf(TABLE_RELATIONS));
                 //存储关注人的id
                 HColumnDescriptor attends = new HColumnDescriptor("attends");
                 attends.setBlocksize(2048*1024);
                 attends.setBlockCacheEnabled(true);
                 attends.setMinVersions(1);
                 attends.setMaxVersions(1);
     
                 HColumnDescriptor fans = new HColumnDescriptor("fans");
                 fans.setBlocksize(2048*1024);
                 fans.setBlockCacheEnabled(true);
                 fans.setMinVersions(1);
                 fans.setMaxVersions(1);
     
                 hTableDescriptor.addFamily(attends);
                 hTableDescriptor.addFamily(fans);
     //        创建表
                 admin.createTable(hTableDescriptor);
             }
     
             admin.close();
             connection.close();
         }
     ```

4. 创建微博收件箱表

   - 表结构

     | ***\*方法名\**** | createTableReceiveContentEmails |
     | ---------------- | ------------------------------- |
     | Table Name       | weibo:receive_content_email     |
     | RowKey           | 用户ID                          |
     | ColumnFamily     | info                            |
     | ColumnLabel      | 用户ID                          |
     | ColumnValue      | 取微博内容的RowKey              |
     | Version          | 1000                            |

   - 代码实现

     ```java
         /**
          * 创建微博收件箱表
          * 表结构：
          方法名   createTableReceiveContentEmails
          Table Name    weibo:receive_content_email
          RowKey    用户ID
          ColumnFamily  info
          ColumnLabel   用户ID
          ColumnValue   取微博内容的RowKey
          Version   1000
     
          */
         public void createTableReceiveContentEmails() throws IOException {
             //获取连接
             Connection connection = getConnection();
             //得到管理员对象
             Admin admin = connection.getAdmin();
             if (!admin.tableExists(TableName.valueOf(TABLE_RECEIVE_CONTENT_EMAIL))){
                 //获取HTableDescriptor
                 HTableDescriptor hTableDescriptor = new HTableDescriptor(TableName.valueOf(TABLE_RECEIVE_CONTENT_EMAIL));
                 //定义列族名称
                 HColumnDescriptor info = new HColumnDescriptor("info");
     //            设置版本保存1000个，就可以将某个人的微博保存1000条
                 info.setMaxVersions(1000);
                 info.setMinVersions(1000);
                 info.setBlockCacheEnabled(true);
                 info.setBlocksize(2048*1024);
                 hTableDescriptor.addFamily(info);
                 //创建表
                 admin.createTable(hTableDescriptor);
             }
     
             admin.close();
             connection.close();
         }
     ```

5. 发布微博内容

   - 微博内容表中添加1条数据
   - 微博收件箱表对所有粉丝用户添加数据

   ```java
    /**
        * 发布微博内容
        * @param uid
        * @param content
        */
       public void publishWeiBo(String uid ,String content) throws IOException {
   //        将发布的微博内容保存到content表里面去
           Connection connection = getConnection();
           Table table = connection.getTable(TableName.valueOf(TABLE_CONTENT));
   //        发布微博的rowkey
           String rowkey = uid + "_"+ System.currentTimeMillis();
           Put put = new Put(rowkey.getBytes());
           put.addColumn("info".getBytes(),"content".getBytes(),System.currentTimeMillis(),content.getBytes());
           table.put(put);
           //查看用户id和他的fans有哪些，查询relation表
           Table table_relations = connection.getTable(TableName.valueOf(TABLE_RELATIONS));
           Get get = new Get(uid.getBytes());
           get.addFamily("fans".getBytes());
           Result result = table_relations.get(get);
           Cell[] cells = result.rawCells();
           if(cells.length <= 0){
               return ;
           }
   //        定义list列表，用于保存uid用户所有的粉丝
           List<byte[]> allFans = new ArrayList<byte[]>();
           for (Cell cell : cells) {
   //            获取用户有哪些列，列名对应粉丝id
               byte[] bytes = CellUtil.cloneQualifier(cell);
               allFans.add(bytes);
           }
   //        操作email表，将用户的所有得到粉丝id作为rowkey，然后以用户发送微博的row可以作为列值，用户id作为列名来保存数据
           Table table_receive_content_email = connection.getTable(TableName.valueOf(TABLE_RECEIVE_CONTENT_EMAIL));
           List<Put> putFansList = new ArrayList<>();
           for (byte[] allFan : allFans) {
               Put put1 = new Put(allFan);
               put1.addColumn("info".getBytes(),Bytes.toBytes(uid),System.currentTimeMillis(),rowkey.getBytes());
               putFansList.add(put1);
           }
           table_receive_content_email.put(putFansList);
   
           table_receive_content_email.close();
           connection.close();
           table_relations.close();
       }
   
   ```

6. 添加关注用户

   - 在微博用户关系表中，对当前主动操作的用户添加新关注的好友
   - 在微博用户关系表中，对被关注的用户添加新的粉丝
   - 微博收件箱表中添加所关注的用户发布的微博

   ```java
    /**
        * 添加关注用户，一次可能添加多个关注用户
        * A 关注一批用户 B,C ,D
        * 第一步：A是B,C,D的关注者   在weibo:relations 当中attend列族当中以A作为rowkey，B,C,D作为列名，B,C,D作为列值，保存起来
        * 第二步：B,C,D都会多一个粉丝A  在weibo:relations 当中fans列族当中分别以B,C,D作为rowkey，A作为列名，A作为列值，保存起来
        * 第三步：A需要获取B,C,D 的微博内容存放到 receive_content_email 表当中去，以A作为rowkey，B,C,D作为列名，获取B,C,D发布的微博rowkey，放到对应的列值里面去
        *
        *
        * @param uid
        * @param attends
        */
       public void addAttends(String uid ,String ...attends) throws IOException {
           Connection connection = getConnection();
           Table relation_table = connection.getTable(TableName.valueOf(TABLE_RELATIONS));
           //循环遍历所有关注的人
           Put put = new Put(uid.getBytes());
           for (String attend : attends) {
               put.addColumn("attends".getBytes(),attend.getBytes(),attend.getBytes());
           }
           relation_table.put(put);
           //粉丝fans添加粉丝，A 关注B，那么自然B就需要添加一个粉丝A
           for (String attend : attends) {
               Put put1 = new Put(attend.getBytes());
               put1.addColumn("fans".getBytes(),uid.getBytes(),uid.getBytes());
               relation_table.put(put1);
           }
   
           //获取uid的所有关注人的收件箱，放到收件箱列表weibo:receive_content_email里面去
           //A 关注B，那么A需要获取B所有的微博内容
           Table table_content = connection.getTable(TableName.valueOf(TABLE_CONTENT));
           Scan scan = new Scan();
           List<byte[]> rowkeyBytes = new ArrayList<>();
           for (String attend : attends) {
               RowFilter rowFilter = new RowFilter(CompareOperator.EQUAL,new SubstringComparator(attend+"_"));
               scan.setFilter(rowFilter);
               ResultScanner scanner = table_content.getScanner(scan);
               for (Result result : scanner) {
                   //获取到数据的rowkey
                   byte[] rowkey = result.getRow();
                   rowkeyBytes.add(rowkey);
               }
           }
   
   
           if(rowkeyBytes.size() > 0){
               Table table_receive_content = connection.getTable(TableName.valueOf(TABLE_RECEIVE_CONTENT_EMAIL));
               List<Put> recPuts = new ArrayList<Put>();
   
               for (byte[] rowkeyByte : rowkeyBytes) {
                   Put put1 = new Put(uid.getBytes());
                   String rowKeyStr = Bytes.toString(rowkeyByte);
   //                通过截取字符串，获取到用户的uid
                   String attendUid = rowKeyStr.substring(0, rowKeyStr.indexOf("_"));
   //                用户发送微博的时间戳
                   long timestamp = Long.parseLong(rowKeyStr.substring(rowKeyStr.indexOf("_") + 1));
   //              将A用户关注的B，C，D用户的微博的rowkey给保存起来
                   put1.addColumn("info".getBytes(),attendUid.getBytes(), timestamp,rowkeyByte);
                   recPuts.add(put1);
               }
   
               table_receive_content.put(recPuts);
   
               table_content.close();
               table_receive_content.close();
               relation_table.close();
               connection.close();
   
           }
   
       }
   
   ```

7. 移除（取关）用户

   - 在微博用户关系表中，对当前主动操作的用户移除取关的好友(attends)
   - 在微博用户关系表中，对被取关的用户移除粉丝
   - 微博收件箱中删除取关的用户发布的微博

   ```java
       /**
        * 取消关注 A取消关注 B,C,D这三个用户
        * 其实逻辑与关注B,C,D相反即可
        * 第一步：在weibo:relation关系表当中，在attends列族当中删除B,C,D这三个列
        * 第二步：在weibo:relation关系表当中，在fans列族当中，以B,C,D为rowkey，查找fans列族当中A这个粉丝，给删除掉
        * 第三步：A取消关注B,C,D,在收件箱中，删除取关的人的微博的rowkey
        */
       public void attendCancel(String uid,String ...cancelAttends) throws IOException {
           Connection connection = getConnection();
           Table table_relations = connection.getTable(TableName.valueOf(TABLE_RELATIONS));
   
           //移除A关注的B,C,D这三个用户
           for (String cancelAttend : cancelAttends) {
               Delete delete = new Delete(uid.getBytes());
               delete.addColumn("attends".getBytes(),cancelAttend.getBytes());
               table_relations.delete(delete);
           }
   
           //B,C,D这三个用户移除粉丝A
           for (String cancelAttend : cancelAttends) {
               Delete delete = new Delete(cancelAttend.getBytes());
               delete.addColumn("fans".getBytes(),uid.getBytes());
               table_relations.delete(delete);
           }
   
           //收件箱表当中  A移除掉B,C,D的信息
           Table table_receive_connection = connection.getTable(TableName.valueOf(TABLE_RECEIVE_CONTENT_EMAIL));
   
           for (String cancelAttend : cancelAttends) {
               Delete delete = new Delete(uid.getBytes());
               delete.addColumn("info".getBytes(),cancelAttend.getBytes());
               table_receive_connection.delete(delete);
   
           }
           table_receive_connection.close();
           table_relations.close();
           connection.close();
       }
   
   ```

8. 获取关注的人的微博内容

   - 从微博收件箱中获取所关注的用户的微博RowKey 
   - 根据获取的RowKey，得到微博内容

   ```java
      /**
        * 某个用户获取收件箱表内容
        * 例如A用户刷新微博，拉取他所有关注人的微博内容
        * A 从 weibo:receive_content_email  表当中获取所有关注人的rowkey
        * 通过rowkey从weibo:content表当中获取微博内容
        */
       public void getContent(String uid) throws IOException {
           //从weibo:receive_content_email 表当中获取用户id为uid的人的所有的微博列表
           Connection connection = getConnection();
           //从 weibo:receive_content_email
           Table table_receive_content_email = connection.getTable(TableName.valueOf(TABLE_RECEIVE_CONTENT_EMAIL));
           //定义list集合里面存储我们的所有的Get对象，用于下一步的查询
           List<Get>  rowkeysList = new ArrayList<Get>();
           Get get = new Get(uid.getBytes());
           //设置最大版本为5个
           get.setMaxVersions(5);
           Result result = table_receive_content_email.get(get);
           Cell[] cells = result.rawCells();
           for (Cell cell : cells) {
               //获取单元格的值
               byte[] rowkeys = CellUtil.cloneValue(cell);
               Get get1 = new Get(rowkeys);
               rowkeysList.add(get1);
           }
           //从weibo:content表当中通过用户id进行查询微博内容
           //table_content内容表
           Table table_content = connection.getTable(TableName.valueOf(TABLE_CONTENT));
           //所有查询出来的内容进行打印出来
           Result[] results = table_content.get(rowkeysList);
           for (Result result1 : results) {
   //            获取微博的内容
               byte[] value = result1.getValue("info".getBytes(), "content".getBytes());
               System.out.println(Bytes.toString(value));
           }
       }
   ```

   