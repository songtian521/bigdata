# 15_MapReduce运行机制详解

MapReduce工作机制全流程图

![](C:\Users\宋天\Desktop\大数据\img\MapReduce\0-MapReduce工作机制-全流程.jpg)

## 1.MapTask工作机制

**概述：**inputFile通过split被逻辑切分为多个split文件，通过Record按行读取内容给Map（用户自己实现）进行处理，数据被Map处理结束之后交给OutPutCollector收集器，对其结果key进行分区（默认使用hash分区），然后写入buffer，每个MapTask都有一个内存缓冲区，存储着Map的输出结果，当缓冲区快满的时候需要将缓冲区的数据以一个临时文件的方式存放到磁盘，当整个MapTask结束后再对磁盘中这个MapTask产生的所有临时文件做合并，生成最终的正式输出文件，然后等待ReduceTask来拉取数据



### 1.1详细步骤

1. 读取数据组件InputFormat（默认TextInputFormat）会通过getSplits方法对输入目录中文件进行逻辑切片规划得到block，有多少个block就对应启动多少个MapTask

2. 将输入文件切分作为block之后，由RecorReader对象（默认是LineRecordReader）进行读取，以`\n`作为分隔符，读取一行数据，返回`<key,value>`。key表示每行首字符偏移值，Value表示这一行文本内容

3. 读取block返回`<key,value>`，进入用户自己继承的Mapper类中，执行用户重写的map函数，RecordReader读取一行这里调用一次

4. Mapper逻辑结束之后，将Mapper的每条结果通过`context.write`进行collect数据收集。在collect中，会先对其进行分区处理，默认是同HashPartitioner

   > MapReduce提供Partitioner接口，作用就是根据key或value及Reducer的数量来决定当前的这对输出数据最终应该交由那个Reduce task处理，默认对Key Hash后再以Reducer数量取模。默认的取模方式只是为了平均Reducer的处理能力，如果用户自己对Partitioner有需求，可以定制并设置到Job上

5. 接下来，会奖数据写入内存，内存中这片区域叫做环形缓冲区，缓冲区的作用是批量收集Mapper结果，减少磁盘IO的影响。**我们的key和value对以及Partition的结果都会被写入缓冲区**。当然，写入之前，key与value的值都会被序列化成字节数组。

   > 环形缓冲区其实是一个数组，数组中存放着key,value的序列化数据和key，value的元数据信息，包括partition,key的起始位置，value的起始位置以及value的长度。环形结构是一个抽象概念

   > 缓冲区是有大小限制，默认是100MB。当Mapper的输出结果很多时，就可能会撑爆内存，所以需要在一定条件下将缓冲区中的数据临时写入磁盘，然后重新利用这块缓冲区。这个从内存往磁盘写数据的过程称为spill，中文可译为**溢写**。这个溢写是由单独的线程来完成，不影响往缓冲区写Mapper结果的线程。溢写线程启动时不应该阻止Mapper的结果输出，所以整个缓冲区有个溢写的比例spill.percent。这个比例默认是0.8，也就是当缓冲区的数据已经达到阈值`buffer size * spill percent = 100MB * 0.8 = 80MB`，溢写线程启动锁定这80MB的内存，执行溢写过程，Mapper的输出结果还可以往剩下的20MB内存中写，互不影响

6. 当溢写线程启动后，需要对着80MB空间内的Key做排序（sort）。排序是MapReduce模型默认的行为，这里的排序也是对序列化的字节做的排序

   > 如果Job设置过Combiner，那么现在就是使用Combiner的时候了。将有相同key的`key/value`对的value加起来，减少溢写到磁盘的数据量。Combiner会优化MapReduce的中间结果，所以它在整个模型中会多次使用

   > 那么，哪些场景才能使用Combiner呢？从这里分析，cCombiner的输出的Reducer的输入，Combiner决不能改变最终的计算结果。Combiner只应该用于那种Reduce的输入，Key/value与输出key/value类型完全一致，且不影响最终结果的场景。比如累加，最大值等。Combiner的使用一定得慎重，如果用好，她对Job执行效率有帮助，反之会影响Reducer的最终结果。

7. 合并溢写文件，每次溢写会在磁盘上生成一个临时文件(写之前判断是否有Combiner)，如果Mapper的输出结果真的很大，又多次这样的溢写发生，磁盘上相应的就会有多个临时文件存在。当整个数据处理结束之后开始对磁盘中的临时文件进行Mege合并，因为最终的文件只有一个，写入磁盘，并且为这个文件提供了一个索引文件，以记录每个Reduce对应数据量的偏移量

### 1.2配置

| 配置                             | 默认值                         | 解释                       |
| -------------------------------- | ------------------------------ | -------------------------- |
| mapreduce.task.io.sort.mb        | 100                            | 设置环型缓冲区的内存值大小 |
| mapreduce.map.sort.spill.percent | 0.8                            | 设置溢写的比例             |
| mapreduce.cluster.local.dir      | ${hadoop.tmp.dir}/mapred/local | 溢写数据目录               |
| mapreduce.task.io.sort.factor    | 10                             | 设置一次合并多少个溢写文件 |

## 2.ReduceTask工作机制

**概述：**Reduce大致分为copy，sort，reduce三个阶段，重点在前两个阶段。copy阶段包含一个eventFetcher来获取已完成的map列表，由Fetcher线程去copy数据，在此过程中会启动两个merge线程，分别为inMemoryMerger和onDiskMerger，分别将内存中的数据merge到磁盘和将磁盘中的数据进行merge。待数据copy完成之后，copy阶段就完成了，开始进行sot阶段，sort阶段主要是执行finalMerge操作，纯粹的sort阶段，完成之后就是reduce阶段，调用用户自定义的reduce函数进行处理

### 2.1详细步骤

1. Copy阶段，简单地拉取数据。Reduce进程启动一些数据copy线程（Fetcher），通过HTTP方式请求maptask获取属于自己的文件。
2. Merge阶段，这里的merge如map端的merge动作，只是数组中存放的是不同map端copy来的数值。copy过来的数据会先放入内存缓冲区中，这里的缓冲区大小要比map端的更为灵活。merge有三种形式：内存到内存，内存到磁盘，磁盘到磁盘。默认情况下第一种形式不启用。当内存中的数据量达到一定的阈值，就启动内存到磁盘的merge。与map端类似，这也是一个溢写的过程，这个过程中如果你设置有Combiner，也是会启用的，然后在磁盘中生成了众多的溢写文件。第二种merge方式一直在运行，直到没有map端的数据时才结束，然后启动第三种磁盘到磁盘的merge方式生成最终的文件
3. 合并排序，把分散的数据合并成一个大的数据后，还会再对合并后的数据排序
4. 对排序后的键值对调用reduce方法，键相等的键值对调用一次reduce方法，每次调用会产生零个或多个键值对，最后把这些输出的键值对写入到HDFS文件中

## 3.Shuffle过程

map阶段处理的数据如何传递给reduce阶段，是MapReduce框架中最关键的一个流程，这个流程就叫shuffle

shuffle：洗牌、发牌（核心机制：数据分区，排序，分组，规约，合并等过程）

Shuffle是MapReduce的核心，它分布在MapRedue的Map阶段和Reduce阶段。一般把从Map产生输出开始到Reduce取得数据作为输入之前的过程称为shuffle

### 3.1 详细过程

1. Collect阶段：将MapTask的结果输出到默认大小为100MB的环形缓冲区，保存的是`key/value`，partiton分区信息等
2. Spill阶段：当内存中的数据量达到一定的阀值的时候，就会将数据写入本地磁盘，在将数据写入磁盘之前需要对数据进行一次排序的操作，如果配置了combiner，还会将有相同分区号和key的数据进行排序
3. Merge阶段：把所有溢出的临时文件进行一次合并操作，以确保一个MapTask最终只产生一个中间数据文件
4. Copy阶段：ReduceTask启动Fetcher线程到已经完成MapTask的节点上复制一份属于自己的数据，这些数据默认会保存在内存的缓冲区中，当内存的缓冲区达到一定的阀值的时候，就会将数据写到磁盘之上
5. Merge阶段：在ReduceTask远程复制数据的同时，会在后台开启两个线程对内存到本地的数据文件进行合并操作
6. Sort阶段：在对数据进行合并的同时，会进行排序操作，由于MapTask阶段已经对数据进行了局部的排序，ReduceTask只需要保证Copy的数据的最终整体有效性即可。Shuffle中的缓冲区大小会影响到MapReduce程序的执行效率，原则上说，缓冲区越大，磁盘IO的次数越小，执行速度就越快。缓冲区的大小可以通过参数调整，参数：`mapreduce.task.io.sort.mb`默认100M

## 4.案例：Reduce端实现Join

### 4.1 需求

> 假如数据量巨大，两表的数据是以文件的形式存储来HDFS中，需要用MapReduce程序来实现以下SQL查询运算

```sql
select a.id,a.date,b.name,b.category_id,b.price from t_order a left join t_product b on a.pid = b.id
```

商品表：

```
id 		pname 	category_id 	price
P0001 	小米5 	1000 			2000
P0002 	锤子T1 	1000 			3000
```

订单数据表：

```
id 		 date 		pid 	amount
1001 	20150710 	P0001 		2
1002 	20150710 	P000		2
```

### 4.2实现步骤

![](img\MapReduce\1-Reduce端join操作.bmp)

![](img\MapReduce\2-Reduce端join操作问题.bmp)

通过将关联的条件作为map输出的key，将两表满足join条件的数据并携带数据所来源的文件
信息，发往同一个reduce task，在reduce中进行数据的串联

1. 定义Mapper

   ```java
   /**
    * k1 LongWritable
    * v1 Text
    *
    * k2 Text 商品ID
    * v2 Text 行文本信息
    */
   public class joinMapper extends Mapper<LongWritable,Text, Text,Text> {
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           /**
            * product.txt
            * k1       v1
            * 0        P0001 小米5 1000 2000
            *
            * orders.txt
            * k1       v1
            * 0        1001 20150710 P0001 2
            *
            * -----------------------
            *
            * k2       v2
            * P0001    P0001 小米5 1000 2000
            *
            * P0001    1001 20150710 P0001 2
            */
   
   
           //1.判断数据来自哪个文件
           FileSplit fileSplit = (FileSplit)context.getInputSplit();//获取文件的切片
           String fileName = fileSplit.getPath().getName();//获取切片的名称
           if (fileName.equals("product.txt")){
               //数据来源于商品表
               //2.将k1 v1 转为k2 v2 ，然后写入上下文
               String[] split = value.toString().split(",");
               String productId = split[0];
               context.write(new Text(productId),value);
   
           }else{
               //数据源自订单数据表
               //2.将k1 v1 转为k2 v2 ，然后写入上下文
               String[] split = value.toString().split(",");
               String productId = split[2];
               context.write(new Text(productId),value);
           }
   
   
       }
   }
   
   ```

2. 定义Reduce

   ```java
   public class joinReducer extends Reducer<Text,Text,Text, Text> {
       @Override
       protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
   
           //1.遍历集合，获取v3(first + second)
           String first = "";
           String second = "";
           for ( Text value: values) {
               if (value.toString().startsWith("p")){
                   first = value.toString();
               }else {
                   second += value.toString();
               }
           }
   
           //2.k3和v3写入上下文
           context.write(key,new Text(first+ "\t" +second));
       }
   }
   
   ```

3. 定义主类

   ```java
   public class joinMain extends Configured implements Tool {
   
       @Override
       public int run(String[] strings) throws Exception {
           //1.获取job对象
           Job job = Job.getInstance(super.getConf(), "reduce_join");
           //2.设置job任务
   
           //设置输入类和输入路径
           job.setInputFormatClass(TextInputFormat.class);
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\reduce_join\\MyOutputformat.input"));
           //设置map类和数据类型
           job.setMapperClass(joinMapper.class);
           job.setMapOutputKeyClass(Text.class);
           job.setMapOutputValueClass(Text.class);
           //shuffle阶段省略，采用默认方法
           //设置Reducer类和数据类型
           job.setReducerClass(joinReducer.class);
           job.setOutputKeyClass(Text.class);
           job.setOutputValueClass(Text.class);
           //设置输出类和输出的路径
           job.setOutputFormatClass(TextOutputFormat.class);
           TextOutputFormat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\reduce_join\\out"));
   
           //3.等待job任务结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception {
           Configuration conf = new Configuration();
   
           //启动job任务
           int run = ToolRunner.run(conf, new joinMain(), args);
       }
   }
   
   ```

4. 输出结果

   ```
   p0001	p0001,小米5,1000,2000	2001,20150710,p0001,21001,20150710,p0001,2
   p0002	p0002,锤子T1,1000,3000	2002,20150710,p0002,31002,20150710,p0002,3
   ```

   

## 5.案例：Map端实现Join

概述：适用于关联表中有小表的情形

使用分布式缓存，可以将小表分发到所有的map节点，这样map节点就可以在本地对自己所读到的大表的数据进行join并输出最终结果，可以大大提高join的操作的并发度，加快处理速度

### 5.1 实现步骤

![](img\MapReduce\3-Map端join操作.bmp)

先在mapper类中预定义好小表，进行join

引入实际场景中的解决方案：一次加载数据库

1. 定义Mapper

   ```java
   public class joinMapper extends Mapper<LongWritable, Text,Text,Text> {
       private HashMap<String,String> hashMap = new HashMap<>();
       //1.将分布式缓存的小表数据读取到本地map集合（只需要做一次）
       //setup方法只执行一次
       @Override
       protected void setup(Context context) throws IOException, InterruptedException {
           //1.获取分布式缓存列表
           URI[] cacheFiles = context.getCacheFiles();//包含了所有的缓存文件
           //2.获取指定的缓存文件的文件系统（fileSystem）
           FileSystem fileSystem = FileSystem.get(cacheFiles[0], context.getConfiguration());
           //3.获取文件的输入流
           FSDataInputStream inputStream = fileSystem.open(new Path(cacheFiles[0]));
           //4.读取文件内容，并将数据存入map集合
               //将字节输入流转为字符缓冲流FSdataInputStream --> BufferedReader
               BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
               //以行为单位读取小表内容，并将读取的数据存入map集合
   
               String line = null;
               while ((line = bufferedReader.readLine())!= null){
                   String[] split = line.split(",");
                   hashMap.put(split[0],line);
   
               }
   
           //5.关闭流
           bufferedReader.close();
           fileSystem.close();
   
       }
   
       //2.对大表的业务处理逻辑，而且要实现大表和小表的join操作
   
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           //1.从行文本数据中获取商品ID
           String[] split = value.toString().split(",");
           String productID = split[2];
           //2.在map集合中，将商品的ID作为键，获取值（商品的行文本数据），将value和值拼接，得到v2
           String productLine = hashMap.get(productID);
           String valueLine = productLine + "\t"+ value.toString();
           //3.将k2和v2写入上下文中
           context.write(new Text(productID),new Text(valueLine));
   
       }
   }
   
   ```

2. 定义主类：

   ```java
   public class joinMain extends Configured implements Tool {
   
       @Override
       public int run(String[] strings) throws Exception {
           //1.获取job对象
           Job job = Job.getInstance(super.getConf(), "map_join");
   
           //2.设置job对象（将小表放在分布式缓存当中）
           //将小表放在分布式缓存当中
           job.addCacheFile(new URI("hdfs://bigdata111:9000/product.txt"));
   
           //设置输入类和输入路径
           job.setInputFormatClass(TextInputFormat.class);
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\Map_join\\MyOutputformat.input"));
           //设置mapper
           job.setMapperClass(joinMapper.class);
           job.setMapOutputKeyClass(Text.class);
           job.setMapOutputValueClass(Text.class);
           //shuffle阶段默认
           //本次案例没有Reduce
   
           //设置输出类和输出路径
           job.setOutputValueClass(TextOutputFormat.class);
           TextOutputFormat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\Map_join\\out"));
   
           //3.等待任务结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception{
           Configuration conf = new Configuration();
   
           //启动job任务
           int run = ToolRunner.run(conf, new joinMain(), args);
       }
   }
   
   ```

3. 输出结果：

   ```
   p0001	p0001,小米5,1000,2000		2001,20150710,p0001,2
   p0001	p0001,小米5,1000,2000		1001,20150710,p0001,2
   p0002	p0002,锤子T1,1000,3000	2002,20150710,p0002,3
   p0002	p0002,锤子T1,1000,3000	1002,20150710,p0002,3
   
   ```

   

## 6.案例：求共同好友

### 6.1 需求分析

以下是QQ好友列表数据，冒号前是一个用户，冒号后是该用户的所有好友（数据中的好友关系是单向的）

```
A:B,C,D,F,E,O
B:A,C,E,K
C:A,B,D,E,I
D:A,E,F,L
E:B,C,D,M,L
F:A,B,C,D,E,O,M
G:A,C,D,E,F
H:A,C,D,E,O
I:A,O
J:B,O
K:A,C,D
L:D,E,F
M:E,F,G
O:A,H,I,J
```

求出哪些人两两之间有共同好友，及他两的共同好友都有谁？

### 6.2实现步骤

![](img\MapReduce\4-求共同好友.bmp)

第一步：

1. 定义Mapper

   ```java
   public class stepMapper1 extends Mapper<LongWritable,Text, Text,Text> {
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           //1.拆分行文本数据，冒号拆分，冒号左边为v2
           String[] split = value.toString().split(":");
           String userStr = split[0];
   
           //2.将冒号右边的字符串以逗号拆分，每个成员就是k2
           String[] split1 = split[1].split(",");
           for (String s : split1) {
               //3.将k2和v2写入上下文
               context.write(new Text(s),new Text(userStr));
           }
   
   
       }
   }
   
   ```

2. 定义Reduce

   ```java
   public class stepReducer1 extends Reducer<Text,Text,Text,Text> {
       @Override
       protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
           //1.遍历集合，将每一个元素拼接得到k3
           StringBuffer buffer = new StringBuffer();
           for(Text value : values){
               buffer.append(value.toString()).append("-");
           }
           //2.k2就是v3
           //3.将k3和v3写入上下文
           context.write(new Text(buffer.toString()),key);
       }
   }
   
   ```

3. 定义主类：

   ```java
   public class joinMain extends Configured implements Tool {
   
       @Override
       public int run(String[] strings) throws Exception {
           //1.获取job对象
           Job job = Job.getInstance(super.getConf(), "common_friends");
           //2.设置job任务
   
           //设置输入类和输入路径
           job.setInputFormatClass(TextInputFormat.class);
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\common_friends_step1\\MyOutputformat.input"));
           //设置map类和数据类型
           job.setMapperClass(stepMapper1.class);
           job.setMapOutputKeyClass(Text.class);
           job.setMapOutputValueClass(Text.class);
           //shuffle阶段省略，采用默认方法
           //设置Reducer类和数据类型
           job.setReducerClass(stepReducer1.class);
           job.setOutputKeyClass(Text.class);
           job.setOutputValueClass(Text.class);
           //设置输出类和输出的路径
           job.setOutputFormatClass(TextOutputFormat.class);
           TextOutputFormat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\common_friends_step1\\out"));
   
           //3.等待job任务结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception {
           Configuration conf = new Configuration();
   
           //启动job任务
           int run = ToolRunner.run(conf, new joinMain(), args);
       }
   }
   
   ```

4. 第一步 输出结果

   ```
   I-K-B-G-F-H-O-C-D-	A
   A-F-C-J-E-	B
   E-A-H-B-F-G-K-	C
   K-E-C-L-A-F-H-G-	D
   F-M-L-H-G-D-C-B-A-	E
   G-D-A-M-L-	F
   M-	G
   O-	H
   C-O-	I
   O-	J
   B-	K
   D-E-	L
   E-F-	M
   J-I-H-A-F-	O
   
   ```

第二步：

1. 定义Mapper

   ```java
   public class stepMapper2 extends Mapper<LongWritable, Text,Text,Text> {
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           /**
            * k1       v1
            * 0         A-F-C-J-E-	B
            *
            * k2       v2
            * A-C      B
            * A-E      B
            * A-F      B
            * C-E      B
            * ......
            */
           //1.拆分行文本数据，得到v2
           String[] split = value.toString().split("\t");
           String friendStr = split[1];
           //2.继续拆分以-为分隔符拆分行文本数据，得到一个数组
           String[] userArray = split[0].split("-");
           //3.对数组做一个排序
           Arrays.sort(userArray);
           //4.对数组中的元素进行两两组合，得到k2
           /**
            * A-E-C-J   --->       A C E
            *
            * A C E
            *   A C E
            */
           for(int i = 0; i < userArray.length - 1 ; i++){
               for (int j = i+1; j < userArray.length; j++) {
                   //5.将k2和v2写入上下文中
                   context.write(new Text(userArray[i] + "-" + userArray[j]),new Text(friendStr));
               }
           }
   
       }
   }
   
   ```

2. 定义Reduce

   ```java
   public class stepReducer2 extends Reducer<Text,Text, Text,Text> {
       @Override
       protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
           //1.原来的k2就是k3
           //2.将集合进行遍历，将集合中的元素拼接，得到v3
           StringBuffer stringBuffer = new StringBuffer();
           for (Text value : values) {
               stringBuffer.append(value.toString()).append("-");
   
            }
           //3.将k3和v3写入上下文
           context.write(key,new Text((stringBuffer.toString())));
       }
   }
   
   ```

3. 定义JoinMain

   ```java
   public class joinMain extends Configured implements Tool {
   
       @Override
       public int run(String[] strings) throws Exception {
           //1.获取job对象
           Job job = Job.getInstance(super.getConf(), "common_friends2");
           //2.设置job任务
   
           //设置输入类和输入路径
           job.setInputFormatClass(TextInputFormat.class);
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\common_friends_step1\\out"));//上一个的输出
           //设置map类和数据类型
           job.setMapperClass(stepMapper2.class);
           job.setMapOutputKeyClass(Text.class);
           job.setMapOutputValueClass(Text.class);
           //shuffle阶段省略，采用默认方法
           //设置Reducer类和数据类型
           job.setReducerClass(stepReducer2.class);
           job.setOutputKeyClass(Text.class);
           job.setOutputValueClass(Text.class);
           //设置输出类和输出的路径
           job.setOutputFormatClass(TextOutputFormat.class);
           TextOutputFormat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\common_friends_step2\\out"));
   
           //3.等待job任务结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception {
           Configuration conf = new Configuration();
   
           //启动job任务
           int run = ToolRunner.run(conf, new joinMain(), args);
       }
   }
   
   ```

4. 第二步输出结果的一部分：

   ```
   A-B	C-E-
   A-C	B-E-D-
   A-D	F-E-
   A-E	B-D-C-
   A-F	O-E-D-B-C-
   A-G	C-F-D-E-
   A-H	O-D-C-E-
   A-I	O-
   A-J	B-O-
   ```

   

## 7.案例：自定义InputFormt合并小文件

### 7.1 需求

无论hdfs还是MapReduce，对于小文件都有折损率，实践中，又难免面临处理大量小文件的场景，此时，就需要有相应的解决方案

### 7.2 分析实现

小文件的优化无非以下几种方式：

- 在数据采集之前，就将小文件或小批数据合并成大文件在上传HDFS
- 在业务处理之前，在HDFS上使用MapReduce程序对小文件进行合并
- 在MapReduce处理是，可采用combineInputFormat提高效率

代码实现：

采用上述第二种方式

程序的核心机制：

- 自定义一个InputFormat，改写RecordReader，实现一次读取一个完整文件封装为KV
- 在输出时使用SequenceFileOutPutFormat输出合并小文件

代码如下：

1. 自定义InputFormat

   ```java
   public class MyInputformat extends FileInputFormat<NullWritable, BytesWritable> {
       @Override
       public RecordReader<NullWritable, BytesWritable> createRecordReader(InputSplit inputSplit, TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
           //1.创建自定义RecordReader对象
           MyRecordReader myRecordReader = new MyRecordReader();
           //2.将inputSplit和context对象传给MyRecordReader
           myRecordReader.initialize(inputSplit,taskAttemptContext);
           return myRecordReader;
       }
   
       //设置文件是否可以被切割
       @Override
       protected boolean isSplitable(JobContext context, Path filename) {
           return false;
       }
   }
   
   ```

2. 自定义RecordReader

   ```java
   public class MyRecordReader extends RecordReader<NullWritable, BytesWritable> {
       private Configuration configuration = null;
       private FileSplit fileSplit = null;
       private boolean processed = false;//状态值
       private BytesWritable bytesWritable =  bytesWritable= new BytesWritable();
       private FileSystem fileSystem = null;
       private FSDataInputStream inputStream = null;
       //进行初始化工作
       @Override
       public void initialize(InputSplit inputSplit, TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
           //获取文件的切片
           fileSplit = (FileSplit) inputSplit;
           //获取configuration对象
           configuration = taskAttemptContext.getConfiguration();
       }
   
       //用于获取k1和v1
       @Override
       public boolean nextKeyValue() throws IOException, InterruptedException {
           /*
            * k1  NullWritable
            * v1  BytesWritable
            *
            * */
           if(!processed){
               //1.获取源文件的字节输入流
                   //1.1获取源文件文件系统fileSystem
                   fileSystem= FileSystem.get(configuration);
                   //1.2通过fileSystem获取文件字节输入流
                   inputStream = fileSystem.open(fileSplit.getPath());
   
               //2.读取源文件数据到普通的字节数组（byte[])
               byte[] bytes = new byte[(int)fileSplit.getLength()];
               IOUtils.readFully(inputStream,bytes,0,(int)fileSplit.getLength());
               //3.将字节数组中的数据封装到bytesWritable，得到v1
               bytesWritable.set(bytes,0,(int)fileSplit.getLength());
   
               processed = true;
               return true;
           }
   
           return false;
       }
   
       //返回k1
       @Override
       public NullWritable getCurrentKey() throws IOException, InterruptedException {
           return NullWritable.get();
       }
       //返回v1
       @Override
       public BytesWritable getCurrentValue() throws IOException, InterruptedException {
           return bytesWritable;
       }
   
       //回去文件读取的进度
       @Override
       public float getProgress() throws IOException, InterruptedException {
           return 0;
       }
   
       //进行资源的释放
       @Override
       public void close() throws IOException {
           inputStream.close();
           fileSystem.close();
       }
   }
   
   ```

3. Mapper类

   ```java
   public class SequenceFileMapper extends Mapper<NullWritable, BytesWritable, Text,BytesWritable> {
       @Override
       protected void map(NullWritable key, BytesWritable value, Context context) throws IOException, InterruptedException {
           //1.获取文件的名字，作为k2
           FileSplit fileSplit = (FileSplit)context.getInputSplit();
           String filename = fileSplit.getPath().getName();
           //2.将k2和v2写入上下文
           context.write(new Text(filename),value);
       }
   }
   
   ```

4. 主类

   ```java
   public class JobMain extends Configured implements Tool {
       @Override
       public int run(String[] strings) throws Exception {
           //1.获取job对象
           Job job = Job.getInstance(super.getConf(), "sequence_file_job");
   
           //2.设置job对象
               //1.设置输入类和输入路径
               job.setInputFormatClass(MyInputformat.class);
               MyInputformat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\MyInputformat\\MyOutputformat.input"));
               //2.设置mapper类和数据类型
               job.setMapperClass(SequenceFileMapper.class);
               job.setMapOutputKeyClass(Text.class);
               job.setMapOutputValueClass(BytesWritable.class);
               //3.不需要设置reducer类，但是必须设置数据类型
               job.setOutputKeyClass(Text.class);
               job.setOutputValueClass(BytesWritable.class);
               //4.设置输出类和输出的路径
               job.setOutputFormatClass(SequenceFileOutputFormat.class);
               SequenceFileOutputFormat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\MyInputformat\\out"));
           //3.等到job任务执行结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception {
           Configuration conf = new Configuration();
           int run = ToolRunner.run(conf, new JobMain(), args);
           System.exit(run);
       }
   }
   
   ```

   

## 8.案例：自定义OutputFormat

### 8.1 需求

现在有一些订单的评论数据，需求：将订单的好评与差评进行区分，将最终的数据分开到不同的文件夹下面去，数据内容参见文件，其中数据第九个字段表示好评，中评，差评。0：好评，1：中评，2：差评

### 8.2 分析实现

实现要点：

- 在MapReduce中访问外部资源
- 自定义OutputFormat，改写其中的RecordWriter，改写具体输出数据的方法write();

代码实现：

1. 自定义MyOutputFormat

   ```java
   public class MyOutputformat extends FileOutputFormat<Text, NullWritable> {
   
       @Override
       public RecordWriter<Text, NullWritable> getRecordWriter(TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
           //1.获取数据源目标文件的输出流（两个）
           FileSystem fileSystem = FileSystem.get(taskAttemptContext.getConfiguration());
           FSDataOutputStream goodcommentsOutputStream = fileSystem.create(new Path("file:///G:\\学习\\maven\\src\\main\\java\\MyOutputformat\\out1\\good.txt"));
           FSDataOutputStream badcommentsOutputStream = fileSystem.create(new Path("file:///G:\\学习\\maven\\src\\main\\java\\MyOutputformat\\out2\\bad.txt"));
           //2.将输出流传给MeRecordWrite
           MyRecordWrite myRecordWrite = new MyRecordWrite(goodcommentsOutputStream,badcommentsOutputStream);
           return myRecordWrite;
       }
   }
   
   ```

2. MyRecordReader类:

   ```java
   public class MyRecordWrite extends RecordWriter<Text, NullWritable> {
       private FSDataOutputStream goodcommentsOutputStream = null;
       private FSDataOutputStream badcommentsOutputStream = null;
   
       public MyRecordWrite(FSDataOutputStream goodcommentsOutputStream, FSDataOutputStream badcommentsOutputStream) {
           this.goodcommentsOutputStream = goodcommentsOutputStream;
           this.badcommentsOutputStream = badcommentsOutputStream;
       }
   
       public MyRecordWrite() {
       }
   
       /**
        *
        * @param text 行文本内容
        * @param nullWritable
        * @throws IOException
        * @throws InterruptedException
        */
       @Override
       public void write(Text text, NullWritable nullWritable) throws IOException, InterruptedException {
           //1.从行文本中获取源数据第九个字段
           String[] split = text.toString().split("\t");
           String numStr = split[9];
   
           //2.根据字段的值判断评论的类型，将对应的数据写入不同的文件夹
           if(Integer.parseInt(numStr) <= 1){
               //好评或中评
               goodcommentsOutputStream.write(text.toString().getBytes());
               goodcommentsOutputStream.write("\r\n".getBytes());
           }else{
               //差评
               badcommentsOutputStream.write(text.toString().getBytes());
               badcommentsOutputStream.write("\r\n".getBytes());
           }
   
       }
   
       @Override
       public void close(TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
           IOUtils.closeStream(goodcommentsOutputStream);
           IOUtils.closeStream(badcommentsOutputStream);
       }
   }
   
   ```

3. 自定义Mapper类

   ```java
   public class MyOutputFormatMapper extends Mapper<LongWritable, Text,Text, NullWritable> {
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           context.write(value,NullWritable.get());
       }
   }
   
   ```

4. 主类

   ```java
   public class JobMain extends Configured implements Tool {
   
       @Override
       public int run(String[] strings) throws Exception {
           //1.获取job对象
           Job job = Job.getInstance(super.getConf(), "myoutputFormat_job");
   
           //2.设置job任务
           //1.设置输入类和输入的路径
           job.setInputFormatClass(TextInputFormat.class);
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\MyOutputformat\\input"));
           //2.设置mapper类和数据类型
           job.setMapperClass(MyOutputFormatMapper.class);
           job.setMapOutputKeyClass(Text.class);
           job.setMapOutputValueClass(NullWritable.class);
           //3.设置输出类和输出的路径
           job.setOutputFormatClass(MyOutputformat.class);
           MyOutputformat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\MyOutputformat\\out"));//校验文件存放目录
           //3.等待任务结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception{
           Configuration configuration = new Configuration();
           int run = ToolRunner.run(configuration, new JobMain(),args);
           System.exit(run);
       }
   }
   
   ```

   

## 9.自定义分组求取topN

分组是MapReduce当中Reduce端的一个功能组件，主要的作用是决定哪些数据作为一组，调用一次Reduce的逻辑，默认是每个不同的key，作为多个不同的组，每个组调用一次Reduce逻辑，我们可以自定义分组实现不同的key作为同一组，调用一次Reduce逻辑

### 9.1需求

有如下订单数据：

```
订单id 			商品id 	成交金额
Order_0000001 	Pdt_01 		222.8
Order_0000001 	Pdt_05 		25.8
Order_0000002 	Pdt_03 		522.8
Order_0000002 	Pdt_04 		122.4
Order_0000002 	Pdt_05 		722.4
Order_0000003 	Pdt_01 		222.8
```

现在需要求出每一个订单中成交金额最大的一笔

### 9.2 分析实现

- 利用订单ID和成交金额作为key，可以将map解读那读取到的所有订单数据按照ID区分，按照金额排序，发送到Reduce
- 在Reduce端利用分组将订单ID相同的KV聚合成组，然后取第一个即是最大值

代码实现：

1. 定义OrderBean

   > 定义一个OrderBean，里面定义两个字段，第一个字段是我们的orderId，第二个字段是我们的金额（注意金额一定要使用Double或者DoubleWritable类型，否则没法按照金额顺序排序

   ```java
   public class OrderBean implements WritableComparable<OrderBean> {
       private String orderID;
       private Double price;
   
       public String getOrderID() {
           return orderID;
       }
   
       public void setOrderID(String orderID) {
           this.orderID = orderID;
       }
   
       public Double getPrice() {
           return price;
       }
   
       public void setPrice(Double price) {
           this.price = price;
       }
   
       @Override
       public String toString() {
           return  orderID + "\t" + price;
       }
   
       //指定排序规则
       @Override
       public int compareTo(OrderBean o) {
           //如果订单ID一致，则排序订单金额（降序）
           int i = this.orderID.compareTo(o.orderID);
           if (i == 0){
               i = this.price.compareTo(o.price) * -1;
           }
           return i;
       }
   
       //实现对象的序列化
       @Override
       public void write(DataOutput dataOutput) throws IOException {
           dataOutput.writeUTF(orderID);
           dataOutput.writeDouble(price);
       }
       //实现对象的反序列化
       @Override
       public void readFields(DataInput dataInput) throws IOException {
           this.orderID = dataInput.readUTF();
           this.price = dataInput.readDouble();
       }
   }
   
   ```

2. 定义Mapper类

   ```java
   public class GroupMapper extends Mapper<LongWritable, Text,OrderBean,Text> {
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
   
           //1.拆分行文本数据，得到订单ID，订单的金额
           String[] split = value.toString().split("\t");
           //2.封装OrderBean，得到K2
           OrderBean orderBean = new OrderBean();
           orderBean.setOrderID(split[0]);
           orderBean.setPrice(Double.valueOf(split[2]));
           //3.将k2和v2写入上下文
           context.write(orderBean,value);
       }
   }
   ```

3. 自定义分区

   > 自定义分区，按照订单id进行分区，把所有订单id相同的数据，都发送到同一个reduce中去

   ```java
   public class OrderPartition extends Partitioner<OrderBean, Text> {
       //分区规则，根据订单的ID实现分区
   
       /**
        *
        * @param orderBean k2
        * @param text v2
        * @param i ReduceTask的个数
        * @return 返回分区的编号
        */
       @Override
       public int getPartition(OrderBean orderBean, Text text, int i) {
           return (orderBean.getOrderID().hashCode() & 2147483647) % i;
       }
   }
   
   ```

4. 自定义分组

   > 按照我们自己的逻辑进行分组，通过比较相同的订单id，将相同的订单id放到一个组里面去，进过分组之后当中的数据，已经全部是排好序的数据，我们只需要取前topN即可

   ```java
   // 1.继承WritableComparator
   public class OrderGroupComparator extends WritableComparator {
   //    2.用父类的有参构造
       public OrderGroupComparator(){
           super(OrderBean.class,true);
       }
   
   //    3.指定分组的规则（重写方法）
   
       @Override
       public int compare(WritableComparable a, WritableComparable b) {
           //1.对形参做强制类型转换
           OrderBean first = (OrderBean)a;
           OrderBean second = (OrderBean)b;
           //2.指定分组规则
           return first.getOrderID().compareTo(second.getOrderID());
       }
   }
   
   ```

5. 定义Reducer类

   ```java
   public class GroupReducer extends Reducer<OrderBean,Text, Text, NullWritable> {
       @Override
       protected void reduce(OrderBean key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
          int i = 0;
          //获取集合中的前N条数据
           for(Text value : values){
               context.write(value,NullWritable.get());
               i++;
               if (i>=1){
                   break;
               }
           }
       }
   }
   
   ```

6. 程序Main函数入口

   ```java
   public class JobMain extends Configured implements Tool {
   
       @Override
       public int run(String[] strings) throws Exception {
           //1.获取job对象
           Job job = Job.getInstance(super.getConf(), "mygroup_job");
   
           //2.设置job任务
           //1.设置输入类和输入的路径
           job.setInputFormatClass(TextInputFormat.class);
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\DemoTopN\\input"));
           //2.设置mapper类和数据类型
           job.setMapperClass(GroupMapper.class);
           job.setMapOutputKeyClass(OrderBean.class);
           job.setMapOutputValueClass(Text.class);
           //3.设置分区
           job.setPartitionerClass(OrderPartition.class);
           //4.设置分组
           job.setGroupingComparatorClass(OrderGroupComparator.class);
   
           //5.设置Reducer类和数据类型
           job.setReducerClass(GroupReducer.class);
           job.setOutputKeyClass(Text.class);
           job.setOutputValueClass(NullWritable.class);
   
           //6.设置输出类和输出的路径
           job.setOutputFormatClass(TextOutputFormat.class);
           MyOutputformat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\DemoTopN\\out"));//校验文件存放目录
           //3.等待任务结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception{
           Configuration configuration = new Configuration();
           int run = ToolRunner.run(configuration, new JobMain(), args);
           System.exit(run);
       }
   }
   
   ```

7. 输出结果

   ```
   Order_0000001	Pdt_01	222.8
   Order_0000002	Pdt_05	722.4
   Order_0000003	Pdt_01	222.8
   ```

   