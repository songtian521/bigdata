# 14_MapReduce初解

## 1.MapReduce介绍

MapReduce思想在生活中处处可见。或多或少都曾接触过这种思想。MapReduce的思想核心是“分而治之”，适用于大量复杂的任务处理场景（大规模数据处理场景）。

- Map负责“分”，即把复杂的任务分解为若干个简单的任务来并行处理。可以拆分的前提是这些小任务可以并行计算，彼此间几乎没有依赖关系

- Reduce负责“和”，即对map阶段的结果进行全局汇总。

- MapReduce运行在yarn集群

  - RecourceManager
  - NodeManager

  这两个阶段合起来正是MapReduce思想的体现

![](C:\Users\宋天\Desktop\大数据\img\MapReduce\MapReduce编程流程.jpg)

使用比较通俗的语言进行解释MapReduce就是：

- 我们要数图书馆的所有书，你数1号书架，我数2号书架。这就是Map。我们人越多，数书就更快。
- 现在我们到一起，把所有人的统计数加在一起。这就是Reduce。

## 2.MapReduce设计构思

MapReduce是一个分布式运算程序的编程框架。核心功能是将用户编写的业务逻辑代码和自带默认组件整合成一个完整的分布式运算程序，并发运行在Hadoop集群上

MapReduce设计并提供了统一的计算框架，为程序员隐藏了绝大多数系统层面的处理细节。为程序员提供了一个抽象和高层的编程接口和框架。程序员仅需要关心其应用层的具体计算问题，仅需编写少量的处理应用本身计算问题的程序代码。如何具体完成这个并行计算任务所相关的诸多系统层细节被隐藏起来，交给计算框架去处理

Map和Reduce为程序员提供了一个清晰的接口抽象描述。MapReduce中定义了如下的Map和Reuce两个抽象的编程接口，由用户去编程实现Map和Reduce，MapReduce处理的数据类型是<key,value>键值对。

- Map：(k1,v1)-->[(k2,v2)]
- Reduce：(k2,[v2])--->[(k3,v3)]

一个完整的MapReduce程序在分布式运行时有三类实例进程：

1. MRAppMaster负责整个程序的过程调度及装填协调
2. MapTask负责整个map阶段的整个数据处理流程
3. ReduceTask负责Reduce阶段的整个数据处理流程

![](C:\Users\宋天\Desktop\大数据\img\MapReduce\MapReduce思想.bmp)

## 3.MapReduce编程规范

MapReduce的开发一共有八个步骤，其中Map阶段分为2个步骤，Shuffle阶段有4个步骤，Reduce分为2个步骤

1. Map阶段2个步骤
   - 设置InputForMat类，将数据切分为key-value（k1，v1）对，输入到第二步
   - 自定义Map逻辑，将第一步的结果转换为另外的key-value（k2，v2）对，输出结果
2. Shuffle阶段的四个步骤
   - 对输出的key-value进行**分区**
   - 对不同分区的数据按照相同的key**排序**
   - （可选）对分组过的数据初步**规约**，降低数据的网络拷贝
   - 对数组进行**分组**，相同key的value放入一个集合中
3. Reduce阶段2个步骤
   - 对多个Map任务的结果进行排序以及合并，编写Reduce函数实现自己的逻辑，对输入的key-value进行处理，转为新的key-value（k3，v3）输出
   - 设置OutputFormat处理并保存Reduce输出的key-value数据

## 4.WordCound案例

需求：在一堆给定的文本文件中统计输出每一个单词出现的总次数

分析：

![WordCount案例](img\MapReduce\WordCount案例.jpg)

1. 数据格式准备，保存为txt文件

   ```txt
   hello,world,hadoop
   hive,sqoop,flume,hello
   kitty,tom,jerry,world
   hadoop
   ```

2. Mapper

   ```java
   public class WordCountMapper extends Mapper<LongWritable, Text,Text,LongWritable> { //运用的是自己的类型
       /**
        * 四个泛型解释
        * KEYIN k1的类型
        * VALUEIN v1类型
        *
        * KEYOUT k2的类型
        * VALUEOUT v2的类型
        */
   
       //map方法 将k1 和 v1 转为 k2 和 v2
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           /**
            * 参数：
            * key 行偏移量
            * value 每一行的文本数据
            * context 表示上下文对象
            */
   
           /**
            * 如何将k1和v1转为k2和v2？
            * k1（行偏移量）       v1（数据）
            * 0    			hello,word,hadoop
            * 15   			hdfs,hive,hello
            * .......
            *
            * k2       v2
            * hello    1
            * word     1
            * hadoop   1
            * hive     1
            * .....
            *
            */
   
           Text text = new Text();
           LongWritable longWritable = new LongWritable();
           //1.将一行的文本数据进行拆分
           String[] split = value.toString().split(",");
   
           //2.遍历数组，组装k2和 v2
           for (String word : split) {
               //3.将k2 和 v2 写如上下文
               text.set(word);
               longWritable.set(1);
               context.write(text,longWritable);
           }
   
       }
   }
   
   ```

3. Reducer

   ```java
   public class WordCountReducer extends Reducer<Text, LongWritable,Text,LongWritable> {
       /**
        * 四个泛型解释
        * KEYIN K2类型
        * VALUEIN v2类型
        *
        * KEYOUT K3类型
        * VALUEOUT V3类型
        */
   
       @Override
       protected void reduce(Text key, Iterable<LongWritable> values, Context context) throws IOException, InterruptedException {
           //reduce 方法作用：将新k2 和 v2 转化为 k3 和 v3 ，将 k3 和 v3 写入上下文对象
   
           /**
            *参数
            * key 新k2
            * values 集合 新v2
            * context 表示上下文对象
            *
            * 如何将新的 k2 和 v2 转为  k3 和 v3？
            * 新k2      v2
            * hello    <1,1,1,1>
            * word     <1,1>
            * hadoop   <1>
            *.............
            * k3       v3
            * hello    4
            * word     3
            * hadoop   1
            */
   
           LongWritable longWritable = new LongWritable();
   
           //1. 遍历集合，将集合中的数字相加，得到v3
           long count = 0;
           for (LongWritable value : values){
               count += value.get();
           }
           //2.将K3和V3写入上下文中
           longWritable.set(count);
           context.write(key,longWritable);
   
   
   
       }
   }
   
   ```

4. 定义主类，描述Job，并提交

   ```java
   public class JobMain extends Configured implements Tool {
   
       //该方法用于指定一个job任务
       @Override
       public int run(String[] strings) throws Exception {
           //1.创建job任务对象
           Job job = Job.getInstance(super.getConf(), "wordcount");
           //如果打包运行出错，需要加如下代码
           job.setJarByClass(JobMain.class);
   
           //2.配置job任务对象(八个步骤)
           job.setInputFormatClass(TextInputFormat.class);//1.读文件的方式使用InputFormat的子类TextInputFormat
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\WordCount\\wordcount.txt"));//2.指定读取源文件对象地址。
   
           job.setMapperClass(WordCountMapper.class);//3.指定map阶段的处理方式
           job.setMapOutputKeyClass(Text.class);//4.设置map阶段k2的类型
           job.setMapOutputValueClass(LongWritable.class);//5.设置map阶段v2的类型
   
   //        shuffle阶段暂不做处理，此处采用默认方式
   
           //6.指定Reduce阶段的处理方式和数据类型
           job.setReducerClass(WordCountReducer.class);
           //7.设置k3类型
           job.setOutputKeyClass(Text.class);
           //8.设置v3的类型
           job.setOutputValueClass(LongWritable.class);
   
           //9.设置输出类型
           job.setOutputFormatClass(TextOutputFormat.class);
           //10.设置输出路径
           Path path = new Path("file:///G:\\学习\\maven\\src\\main\\java\\WordCount\\OutWordCount");
           TextOutputFormat.setOutputPath(job,path);
   
           //11.等待任务结束
           boolean b1 = job.waitForCompletion(true);
           return b1 ? 0 : 1;//根据返回结果判断任务是否正常
       }
   
       public static void main(String[] args) throws Exception {
           //配置文件对象
           Configuration conf = new Configuration();
           //启动job任务
           int run = ToolRunner.run(conf, new JobMain(), args);
           System.exit(run);//退出任务
       }
   }
   
   ```

注：实际开发中会将项目打包成jar包放入集群上面运行，只需要将输入输出路径改为集群地址即可

## 5.MapReduce运行模式

### 5.1本地运行模式

1. MapReduce程序是被提交到LocalJobRunner在本地以单进程的形式运行
2. 处理的数据即输出结果可以在本地文件系统，也可以在hdfs上
3. 怎样实现本地运行？写一个程序，不要带集群的配置文件，本质是程序的conf中是否有`mapreduce.framework.name=local` 以及`yarn.resourcemanager.hostname=local `参数
4. 本地模式非常便于业务逻辑的debug，只要偶在idea中打断点即可

```java
configuration.set("mapreduce.framework.name","local");
configuration.set(" yarn.resourcemanager.hostname","local");
TextInputFormat.addInputPath(job,new Path("file:///F:\\wordcount\\input"));
TextOutputFormat.setOutputPath(job,new Path("file:///F:\\wordcount\\output"));
```

### 5.2集群运行模式

1. 将MapReduce程序交给Yarn集群，分发到很多的节点上并发执行
2. 处理的数据和输出结果应位于hdfs文件系统
3. 提交集群的实现步骤：将程序打包jar，然后在集群的任意一个节点上使用hadoop命令穹顶

## 6.MapReduce分区

- 在MapReduce中，通过我们制定分区，会将同一个分区的数据发送到同一个Reduce当中进行处理
- 例如：为了数据的统计，可以把一批类似的数据发送到同一个Reduce当中，在同一个Reduce当中统计相同类型的数据，就可以实现类似的数据分区和统计等
- 其实就是相同类型的数据，有共性的数据，送到一起处理
- Reduce当中默认的分区只要一个

![](C:\Users\宋天\Desktop\大数据\img\MapReduce\Shuffle阶段-分区.bmp)

需求：将以下数据进行分开

详细数据参见partition.csv 这个文本文件，其中第五个字段表示开奖结果数值，现在需求将15
以上的结果以及15以下的结果进行分开成两个文件进行保存

1. 准备文件：

   partition.csv

2. Mapper

   ```java
   public class PartitionMapper extends Mapper<LongWritable, Text,Text, NullWritable> {
       /**
        * k1 行偏移量，LongWritable
        * v1 行文本数据 Text
        *
        * k2 行文本数据 Text
        * v2 NullWritable
        */
   
       //map方法将k1和v1转为k2和v2
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           //1.自定义计数器，方式1
           Counter counter = context.getCounter("MR_COUNTER","partition_counter");
           //每次执行该方法，则计数器变量的值加1
           counter.increment(1L);
           /**
            * 结果
            * MR_COUNTER
            * 		partition_counter=15213
            */
   
           context.write(value,NullWritable.get());
       }
   
   }
   
   ```

3. 自定义Partitioner

   主要的逻辑就在这里，通过Partition将数据发送到不同的Reducer

   ```java
   public class MyPartition extends Partitioner<Text, NullWritable> {
   
       /**
        * 1. 定义分区规则
        * 2. 返回对应的分区编号
        * @param text
        * @param nullWritable
        * @param i
        * @return 0
        */
       @Override
       public int getPartition(Text text, NullWritable nullWritable, int i) {
           //1. 拆分文本数据k2，获取中奖字段的值
           String[] split = text.toString().split("\t");
           String numstr = split[5];
           //2. 判断中奖字段的值和15的关系，然后返回对应的分区编号
           if(Integer.parseInt(numstr) > 15){
               return 1;
           }else {
               return 0;
           }
       }
   }
   ```

4. Reduce

   ```java
   public class partitionReducer extends Reducer<Text, NullWritable,Text,NullWritable> {
       public static enum Counter{
           MY_INPUT_RECOREDS,MY_INPUT_BYTES
       }
       @Override
       protected void reduce(Text key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
           //2.自定义计数器，方式2：使用枚举定义计数器
           context.getCounter(Counter.MY_INPUT_RECOREDS).increment(1L);
           /**
            * 结果：
            * partition.partitionReducer$Counter
            * 		MY_INPUT_RECOREDS=15213
            */
   
           context.write(key,NullWritable.get());
       }
   }
   
   ```

5. JobMain

   主类中设置分区类和ReduceTask个数

   ```java
   public class JobMain extends Configured implements Tool {
       @Override
       public int run(String[] strings) throws Exception {
           //1.创建job任务对象
           Job job = Job.getInstance(super.getConf(),"partition");
           //2.对job任务进行配置
   
           //1设置输入类和输入的路径
           job.setInputFormatClass(TextInputFormat.class);
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\partition\\partition.csv"));
           //2设置mapper类和数据类型
           job.setMapperClass(PartitionMapper.class);
           job.setMapOutputKeyClass(Text.class);
           job.setMapOutputValueClass(NullWritable.class);
           //3.shuffle 分区，排序，规约，分组
           job.setPartitionerClass(MyPartition.class);//自定义分区规则，其他采用默认
           //7.指定Reducer和数据类型
           job.setReducerClass(partitionReducer.class);
           job.setOutputValueClass(Text.class);
           job.setOutputValueClass(NullWritable.class);
   
           //设置reduceTask的个数
           job.setNumReduceTasks(2);
   
           //8.指定输出类和输出路径
           job.setOutputFormatClass(TextOutputFormat.class);
           TextOutputFormat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\partition\\partitionOUT"));
           //3.等待任务结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception {
           Configuration conf = new Configuration();
   //        1.启动job任务
           int run = ToolRunner.run(conf, new JobMain(), args);
           //2.退出进程
           System.exit(run);
       }
   }
   
   ```



## 7.MapReuduce中的计数器

计数器是收集作业统计信息的有效手段之一，用于质量控制或应用，计数器还可辅助诊断系统故障。如果需要将日志信息传输到map或reduce任务，更好的方法通常是看能否用一个计数器值来记录某一特定时间的发生。对于大型分布式作业而言，使用计数器更为方便。除了因为获取计数器值比输出日志更方便，还有根据计数器值统计特定事件的发生次数要比分析一堆日志文件容易得多

hadoop内置计数器列表

| MapReduce任务计数器    | org.apache.hadoop.mapreduce.TaskCounter                      |
| ---------------------- | ------------------------------------------------------------ |
| 文件系统计数器         | org.apache.hadoop.mapreduce.FileSystemCounter                |
| FileInputFormat计数器  | org.apache.hadoop.mapreduce.lib.input.FileInputFormatCounter |
| FileOutputFormat计数器 | org.apache.hadoop.mapreduce.lib.output.FileOutputFormatCounter |
| 作业计数器             | org.apache.hadoop.mapreduce.JobCounter                       |

每次mapreduce执行完成之后，我们都会看到一些日志记录出来，其中最重要的一些日志记
录如下截图

![](C:\Users\宋天\Desktop\大数据\img\MapReduce\日志详情.PNG)

所有的这些都是MapReduce的计数器的功能，既然MapReduce当中有计数器的功能，我们如
何实现自己的计数器？？？

需求：以以上分区代码为案例，统计map接收到的数据记录条数

### 7.1实现计数器方式1

定义计数器，通过context上下文队形可以获取计数器，进行记录，通过context上下文对象，在map端使用计数器进行统计

实现代码如下：

```java
public class PartitionMapper  extends
Mapper<LongWritable,Text,Text,NullWritable>{
    //map方法将K1和V1转为K2和V2
    @Override
    protected void map(LongWritable key, Text value, Context context) 
throws Exception{
        Counter counter = context.getCounter("MR_COUNT", 
"MyRecordCounter");
        counter.increment(1L);
        context.write(value,NullWritable.get());
   }
}
```

运行程序之后就可以看到我们自定义的计数器在map阶段读取了7条数据

```
MR_COUNTER
		partition_counter=15213
```

### 7.2实现计数器方式2

通过enum枚举类型来定义计数器，统计reduce端数据的输入的key有多少

实现代码

```java
public class partitionReducer extends Reducer<Text, NullWritable,Text,NullWritable> {
    public static enum Counter{
        MY_INPUT_RECOREDS,MY_INPUT_BYTES
    }
    @Override
    protected void reduce(Text key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
        //2.自定义计数器，方式2：使用枚举定义计数器
        context.getCounter(Counter.MY_INPUT_RECOREDS).increment(1L);
        /**
         * 结果：
         * partition.partitionReducer$Counter
         * 		MY_INPUT_RECOREDS=15213
         */

        context.write(key,NullWritable.get());
    }
}

```



## 8.MapReduce排序和序列化

- 序列化是指把结构化对象转化为字节流
- 反序列化是序列化的逆过程，把字节流转为结构化对象。当要在进程间传递对象或持久化对象的时候，就需要序列化对象成字节流，反之当要接收到或从磁盘读取的字节流转换为对象，就要进行反序列化
- Java的序列化是一个重量级序列化框架，一个对象被序列化后，会附带很多额外的信息（各种校验信息，header，继承体系等），不便于在网络中搞笑传输。所以，hadoop自己开发了一套序列化机制Writable，精简高效，不用像java对象类一样传输多层的父子关系，需要哪个属性就传输哪个属性值，大大的减少懒得网络传输的开销
- writable是hadoop的序列化格式，hadoop定义了这样一个writable接口。一个类要支持可序列化就只需要实现这个接口即可
- 另外writable有一个子接口是writableComparable，writableComparable是即可实现序列化，也可以对key进行比较，我们这里可以通过自定义key实现writableComparable来实现我们的排序功能

![1-Shuffle阶段的排序和序列化](img\MapReduce\1-Shuffle阶段的排序和序列化.jpg)

案例：shuffle 排序和序列化

1. 准备数据

   ```txt
   a 1
   a 9
   b 3
   a 7
   b 8
   b 10
   a 5
   ```

2. sortBean

   ```java
   public class SortBean implements WritableComparable<SortBean> {
       private String word;
       private int num;
   
       public String getWord() {
           return word;
       }
   
       public void setWord(String word) {
           this.word = word;
       }
   
       public int getNum() {
           return num;
       }
   
       public void setNum(int num) {
           this.num = num;
       }
   
   
       @Override
       public String toString() {
           return  word + '\t' + num;
       }
   
       //实现比较器，指定排序规则
   
       /**
        * 规则：
        *  第一列（word）按照字典顺序进行排序
        *  第一列相同的时候，第二列（num）按照升序进行排序
        * @param sortBean
        * @return
        */
       @Override
       public int compareTo(SortBean sortBean) {
           //先对第一列进行排序
           int result = this.word.compareTo(sortBean.word);//这里的compareTo方法并不是自己写的
           //如果第一类相同，则按照第二列进行排序
           if (result == 0){
               return this.num - sortBean.num;
           }
           return result;
       }
   
       //实现序列化
       @Override
       public void write(DataOutput dataOutput) throws IOException {
           dataOutput.writeUTF(word);
           dataOutput.writeInt(num);
       }
   
       //实现反序列化
       @Override
       public void readFields(DataInput dataInput) throws IOException {
           this.word = dataInput.readUTF();
           this.num = dataInput.readInt();
       }
   }
   
   ```

3. mapper

   ```java
   public class SortMapper extends Mapper<LongWritable, Text,SortBean, NullWritable> {
       /**
        * map方法将k1和v1转为K2和v2
        *
        * k1   v1
        * 0    a 3
        * 5    b 7
        * -------
        * k2               v2
        * sortBean(a 3)    NullWritable
        * sortBean(b 7)    NullWritable
        */
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
   
           //1.将行文本数据v1拆分，并将数据封装到SortBean对象，就可以得到k2
           String[] split = value.toString().split(" ");//注：linux中表示空格为\t
   
           SortBean sortBean = new SortBean();
           sortBean.setWord(split[0]);
           sortBean.setNum(Integer.parseInt(split[1]));
   
           //2.将k2和v2写入上下文中
           context.write(sortBean,NullWritable.get());
       }
   }
   ```

4. reduce

   ```java
   public class SortReducer extends Reducer<SortBean, NullWritable,SortBean,NullWritable> {
       //reduce 将新的k2和 v2 转为 k3 和 v3
       @Override
       protected void reduce(SortBean key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
           context.write(key,NullWritable.get());
       }
   }
   
   ```

5. JobMain

   ```java
   public class JobMain extends Configured implements Tool {
       @Override
       public int run(String[] strings) throws Exception {
           //1.创建job对象
           Job job = Job.getInstance(super.getConf(), "mapreduce_SortBean");
   
           //2.配置job任务(8步）
   
           //1.设置输入类和输入的路径
           job.setInputFormatClass(TextInputFormat.class);
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\mapreduce_demo\\sortBean.txt"));//如果指定目录，则会读取文件夹内所有文件
           //2.设置mapper的类和数据类型
           job.setMapperClass(SortMapper.class);
           job.setMapOutputKeyClass(SortBean.class);
           job.setMapOutputValueClass(NullWritable.class);
           //shuffle阶段暂不做调整
           //7.设置reducer类和类型
           job.setReducerClass(SortReducer.class);
           job.setOutputKeyClass(SortBean.class);
           job.setOutputValueClass(NullWritable.class);
           //8.设置输出路径
           job.setOutputFormatClass(TextOutputFormat.class);
           TextOutputFormat.setOutputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\mapreduce_demo\\SortBean_out"));
   
   
           //3 等待任务结束
           boolean b = job.waitForCompletion(true);
           return b?0:1;
       }
   
       public static void main(String[] args) throws Exception {
           Configuration conf = new Configuration();
   
           //启动job任务
           int run = ToolRunner.run(conf,new JobMain(),args);
   
           System.exit(run);
       }
   }
   
   ```

   

## 9.规约Combiner

概念：每一个map都可能会产生大量的本地输出，Combiner的作用就是对map端的输出夏怒走一次合并，以减少在map和reduce节点之间的数据传输量，以提高网络IO性能，是MapReduce的一种优化手段之一

- combiner是MR程序中Mapper和Reducer之外的一种组件
- combiner组件的父类就是Reducer
- combiner和reducer的区别在于运行的位置
  - combiner是在每一个maptask所在的节点运行
  - Reducer是接收全局所有的Mapper输出结果
- combiner的意义就是对每一个maptask的输出进行局部汇总，以减少网络传输量

![2-规约概念](C:\Users\宋天\Desktop\大数据\img\MapReduce\2-规约概念.bmp)

实现步骤：

1. 自定义一个combiner继承Reducer，重写reducer
2. 在job中设置`job.setCombinerClass(CustomCombiner.class)`

combiner能够应用的前提是不能影响最终的业务逻辑，而且，combiner的输出kv应该和reducer的输入kv类型要对应

### 9.1案例：统计每个单词出现的个数

1. 准备文件

   ```
   hello,world,hadoop
   hive,sqoop,flume,hello
   kitty,tom,jerry,world
   hadoop
   ```

2. 自定义MyCombiner

   ```java
   public class MyCombiner extends Reducer<Text, LongWritable, Text,LongWritable> {
   
       /**
        * key hello
        * values <1,1,1,1>
        * @param key
        * @param values
        * @param context
        * @throws IOException
        * @throws InterruptedException
        */
       @Override
       protected void reduce(Text key, Iterable<LongWritable> values, Context context) throws IOException, InterruptedException {
           //1. 遍历集合，将集合中的数字相加，得到v3
           long count = 0;
           for (LongWritable value : values){
               count += value.get();
           }
           //2.将K3和V3写入上下文中
           context.write(key,new LongWritable(count));
       }
   }
   ```

3. mapper

   ```java
   public class WordCountMapper extends Mapper<LongWritable, Text,Text,LongWritable> { //运用的是自己的类型
       /**
        * 四个泛型解释
        * KEYIN k1的类型
        * VALUEIN v1类型
        *
        * KEYOUT k2的类型
        * VALUEOUT v2的类型
        */
   
       //map方法 将k1 和 v1 转为 k2 和 v2
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           /**
            * 参数：
            * key 行偏移量
            * value 每一行的文本数据
            * context 表示上下文对象
            */
   
           /**
            * 如何将k1和v1转为k2和v2？
            * k1       v1
            * 0    hello,word,hadoop
            * 15   hdfs,hive,hello
            * .......
            *
            * k2       v2
            * hello    1
            * word     1
            * hadoop   1
            * hive     1
            * .....
            *
            */
   
           Text text = new Text();
           LongWritable longWritable = new LongWritable();
           //1.将一行的文本数据进行拆分
           String[] split = value.toString().split(",");
   
           //2.遍历数组，组装k2和 v2
           for (String word : split) {
               //3.将k2 和 v2 写如上下文
               text.set(word);
               longWritable.set(1);
               context.write(text,longWritable);
           }
   
       }
   }
   
   ```

4. reducer

   ```java
   public class WordCountReducer extends Reducer<Text, LongWritable,Text,LongWritable> {
       /**
        * 四个泛型解释
        * KEYIN K2类型
        * VALUEIN v2类型
        *
        * KEYOUT K3类型
        * VALUEOUT V3类型
        */
   
       @Override
       protected void reduce(Text key, Iterable<LongWritable> values, Context context) throws IOException, InterruptedException {
           //reduce 方法作用：将新k2 和 v2 转化为 k3 和 v3 ，将 k3 和 v3 写入上下文对象
   
           /**
            *参数
            * key 新k2
            * values 集合 新v2
            * context 表示上下文对象
            *
            * 如何将新的 k2 和 v2 转为  k3 和 v3？
            * 新k2      v2
            * hello    <1,1,1,1>
            * word     <1,1>
            * hadoop   <1>
            *.............
            * k3       v3
            * hello    4
            * word     3
            * hadoop   1
            */
   
           LongWritable longWritable = new LongWritable();
   
           //1. 遍历集合，将集合中的数字相加，得到v3
           long count = 0;
           for (LongWritable value : values){
               count += value.get();
           }
           //2.将K3和V3写入上下文中
           longWritable.set(count);
           context.write(key,longWritable);
   
   
   
       }
   }
   
   ```

5. jobmain

   ```java
   public class JobMain extends Configured implements Tool {
   
       //该方法用于指定一个job任务
       @Override
       public int run(String[] strings) throws Exception {
           //1.创建job任务对象
           Job job = Job.getInstance(super.getConf(), "wordcount");
           //如果打包运行出错，需要加如下代码
           job.setJarByClass(JobMain.class);
   
           //2.配置job任务对象
           job.setInputFormatClass(TextInputFormat.class);//1.读文件的方式使用InputFormat的子类TextInputFormat
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\combiner\\wordcount.txt"));//2.指定读取源文件对象地址。
   
           job.setMapperClass(WordCountMapper.class);//3.指定map阶段的处理方式
           job.setMapOutputKeyClass(Text.class);//4.设置map阶段k2的类型
           job.setMapOutputValueClass(LongWritable.class);//5.设置map阶段v2的类型
   
   //        shuffle阶段(分区，排序，规约，分组)暂不做处理，此处采用默认方式
           //shuffle 规约 设置
           job.setCombinerClass(MyCombiner.class);
   
           //6.指定Reduce阶段的处理方式和数据类型
           job.setReducerClass(WordCountReducer.class);
           //7.设置k3类型
           job.setOutputKeyClass(Text.class);
           //8.设置v3的类型
           job.setOutputValueClass(LongWritable.class);
   
           //9.设置输出类型
           job.setOutputFormatClass(TextOutputFormat.class);
           //10.设置输出路径
           Path path = new Path("file:///G:\\学习\\maven\\src\\main\\java\\combiner\\OutWordCount");
           TextOutputFormat.setOutputPath(job,path);
   
           //11.等待任务结束
           boolean b1 = job.waitForCompletion(true);
           return b1 ? 0 : 1;//根据返回结果判断任务是否正常
       }
   
       public static void main(String[] args) throws Exception {
           //配置文件对象
           Configuration conf = new Configuration();
           //启动job任务
           int run = ToolRunner.run(conf, new JobMain(), args);
           System.exit(run);//退出任务
       }
   }
   
   ```

   

### 9.2 案例：流量统计求和

需求：统计求和，统计每个手机号的上行数据包总和，下行数据包总和，上行总数据之和，下行总数据之和

分析：以手机号码作为key值，下行流量，上行流量，上行总流量下行总流量四个字段分别作为value值，然后以这key，和value作为map阶段的输出，reduce阶段的输入

![3-流量统计求和](C:\Users\宋天\Desktop\大数据\img\MapReduce\3-流量统计求和.bmp)

1. 自定义map的输出value对象flowBean

   ```java
   public class FlowBean implements Writable {
       private Integer upFlow;// 上行数据包数
       private Integer downFlow;// 下行数据包数
       private Integer upCountFlow;//上行流量总和
       private Integer downCountFlow;//下行流量总和
   
       public Integer getUpFlow() {
           return upFlow;
       }
   
       public void setUpFlow(Integer upFlow) {
           this.upFlow = upFlow;
       }
   
       public Integer getDownFlow() {
           return downFlow;
       }
   
       public void setDownFlow(Integer downFlow) {
           this.downFlow = downFlow;
       }
   
       public Integer getUpCountFlow() {
           return upCountFlow;
       }
   
       public void setUpCountFlow(Integer upCountFlow) {
           this.upCountFlow = upCountFlow;
       }
   
       public Integer getDownCountFlow() {
           return downCountFlow;
       }
   
       public void setDownCountFlow(Integer downCountFlow) {
           this.downCountFlow = downCountFlow;
       }
   
       @Override
       public String toString() {
           return  upFlow +
                   "\t" + downFlow +
                   "\t" + upCountFlow +
                   "\t" + downCountFlow;
       }
   
       //序列化方法
       @Override
       public void write(DataOutput dataOutput) throws IOException {
           dataOutput.writeInt(upFlow);
           dataOutput.writeInt(downFlow);
           dataOutput.writeInt(upCountFlow);
           dataOutput.writeInt(downCountFlow);
       }
       //反序列化
       @Override
       public void readFields(DataInput dataInput) throws IOException {
           this.upFlow = dataInput.readInt();
           this.downFlow = dataInput.readInt();
           this.upCountFlow = dataInput.readInt();
           this.downCountFlow = dataInput.readInt();
       }
   }
   
   ```

2. Mapper

   ```java
   public class FlowCountMapper extends Mapper<LongWritable, Text,Text,FlowBean> {
       /**
        * 将k1和 v1 转为 k2 和 v2
        * k1       v1
        * 0        1363157985059 	13600217502	00-1F-64-E2-E8-B1:CMCC	120.196.100.55	www.baidu.com	综合门户	19	128	1177	16852	200
        *
        * ------------
        *
        * k2               v2
        * 13600217502      FlowBean(19	128	1177	16852)
        */
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           //1.拆分行文本数据，得到手机号 --》k2
           String[] split = value.toString().split("\t");
           String phoneNum = split[1];
   
           //2.创建FlowBean对象，并从行文本数据拆分出流量的四个字段，并将四个字段赋值给FlowBean对象
           FlowBean flowBean = new FlowBean();
           flowBean.setUpFlow(Integer.parseInt(split[6]));
           flowBean.setDownFlow(Integer.parseInt(split[7]));
           flowBean.setUpCountFlow(Integer.parseInt(split[8]));
           flowBean.setDownCountFlow(Integer.parseInt(split[9]));
   
           //3.将k2 和  v2 写入上下文中
           context.write(new Text(phoneNum),flowBean);
   
       }
   }
   
   ```

3. reducer

   ```java
   public class FlowReducer extends Reducer<Text,FlowBean,Text,FlowBean> {
       @Override
       protected void reduce(Text key, Iterable<FlowBean> values, Context context) throws IOException, InterruptedException {
           //1.遍历集合，并将集合中对应的四个字段累加
            Integer upFlow = 0;// 上行数据包数
            Integer downFlow = 0;// 下行数据包数
            Integer upCountFlow = 0;//上行流量总和
            Integer downCountFlow = 0;//下行流量总和
           for (FlowBean value :values){
               upFlow += value.getUpFlow();
               downFlow += value.getDownFlow();
               upCountFlow += value.getUpCountFlow();
               downCountFlow += value.getDownCountFlow();
           }
           //2.创建FlowBean对象，并给对象赋值
           FlowBean flowBean = new FlowBean();
           flowBean.setUpFlow(upFlow);
           flowBean.setDownFlow(downFlow);
           flowBean.setUpCountFlow(upCountFlow);
           flowBean.setDownCountFlow(downCountFlow);
           //3.将k3和v3写入上下文中
           context.write(key,flowBean);
       }
   }
   
   ```

4. jobMain

   ```java
   public class JobMain extends Configured implements Tool {
   
       //该方法用于指定一个job任务
       @Override
       public int run(String[] strings) throws Exception {
           //1.创建job任务对象
           Job job = Job.getInstance(super.getConf(), "mapreduce_flowBean");
           //如果打包运行出错，需要加如下代码
           job.setJarByClass(JobMain.class);
   
           //2.配置job任务对象
           job.setInputFormatClass(TextInputFormat.class);//1.读文件的方式使用InputFormat的子类TextInputFormat
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\FlowCount\\data_flow.dat"));//2.指定读取源文件对象地址。
   
           job.setMapperClass(FlowCountMapper.class);//3.指定map阶段的处理方式
           job.setMapOutputKeyClass(Text.class);//4.设置map阶段k2的类型
           job.setMapOutputValueClass(FlowBean.class);//5.设置map阶段v2的类型
   
   //        shuffle阶段(分区，排序，规约，分组)暂不做处理，此处采用默认方式
   
           //6.指定Reduce阶段的处理方式和数据类型
           job.setReducerClass(FlowReducer.class);
           //7.设置k3类型
           job.setOutputKeyClass(Text.class);
           //8.设置v3的类型
           job.setOutputValueClass(FlowBean.class);
   
           //9.设置输出类型
           job.setOutputFormatClass(TextOutputFormat.class);
           //10.设置输出路径
           Path path = new Path("file:///G:\\学习\\maven\\src\\main\\java\\FlowCount\\FlowCountOut");
           TextOutputFormat.setOutputPath(job,path);
   
           //11.等待任务结束
           boolean b1 = job.waitForCompletion(true);
           return b1 ? 0 : 1;//根据返回结果判断任务是否正常
       }
   
       public static void main(String[] args) throws Exception {
           //配置文件对象
           Configuration conf = new Configuration();
           //启动job任务
           int run = ToolRunner.run(conf, new JobMain(), args);
           System.exit(run);//退出任务
       }
   }
   
   ```

### 9.3 案例：上行流量倒序排序（递减排序）

分析：以案例9.2的输出数据作为排序的输入数据，自定义FloeBean为map输出的key，以手机号作为map输出的value，因为MapReduce程序会对map阶段输出的key进行排序

1. 定义FlowBean实现WritableComparable实现比较排序

   java的compareTo方法说明：

   - compareTo方法用于将当前对象与方法的参数进行比较
   - 如果指定的数与参数相等返回0
   - 如果指定的数小于参数返回-1
   - 如果指定的数大于参数返回1

   eg：o1.compareTo(o2)返回正数的话，当前对象（调用compareTo方法的对象o1）要排在比较对象（compareeTo传参对象o2）后面，返回负数的话，放在前面

   ```java
   public class FlowBean implements WritableComparable<FlowBean> {
       private Integer upFlow;// 上行数据包数
       private Integer downFlow;// 下行数据包数
       private Integer upCountFlow;//上行流量总和
       private Integer downCountFlow;//下行流量总和
   
       public Integer getUpFlow() {
           return upFlow;
       }
   
       public void setUpFlow(Integer upFlow) {
           this.upFlow = upFlow;
       }
   
       public Integer getDownFlow() {
           return downFlow;
       }
   
       public void setDownFlow(Integer downFlow) {
           this.downFlow = downFlow;
       }
   
       public Integer getUpCountFlow() {
           return upCountFlow;
       }
   
       public void setUpCountFlow(Integer upCountFlow) {
           this.upCountFlow = upCountFlow;
       }
   
       public Integer getDownCountFlow() {
           return downCountFlow;
       }
   
       public void setDownCountFlow(Integer downCountFlow) {
           this.downCountFlow = downCountFlow;
       }
   
       @Override
       public String toString() {
           return  upFlow +
                   "\t" + downFlow +
                   "\t" + upCountFlow +
                   "\t" + downCountFlow;
       }
   
       //序列化方法
       @Override
       public void write(DataOutput dataOutput) throws IOException {
           dataOutput.writeInt(upFlow);
           dataOutput.writeInt(downFlow);
           dataOutput.writeInt(upCountFlow);
           dataOutput.writeInt(downCountFlow);
       }
       //反序列化
       @Override
       public void readFields(DataInput dataInput) throws IOException {
           this.upFlow = dataInput.readInt();
           this.downFlow = dataInput.readInt();
           this.upCountFlow = dataInput.readInt();
           this.downCountFlow = dataInput.readInt();
       }
   
       //指定排序的规则
       @Override
       public int compareTo(FlowBean o) {
           //this.upFlow .compareTo(o.getUpFlow());
           return  o.upFlow - this.upFlow;//降序
       }
   }
   
   ```

2. mapper

   ```java
   public class FlowSortMapper extends Mapper<LongWritable, Text,FlowBean,Text> {
       //map方法：k1,v1转为k2,v2
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           //1.拆分行文本数据（v1），得到四个流量字段，并封装FlowBean对象，得到k2
           String[] split = value.toString().split("\t");
           FlowBean flowBean = new FlowBean();
           flowBean.setUpFlow(Integer.parseInt(split[1]));
           flowBean.setDownFlow(Integer.parseInt(split[2]));
           flowBean.setUpCountFlow(Integer.parseInt(split[3]));
           flowBean.setDownCountFlow(Integer.parseInt(split[4]));
           //2.通过行文本数据，得到手机号 v2
           String phoneNum = split[0];//手机号
           //3.将k2和v2写入上下文中
           context.write(flowBean,new Text(phoneNum));
       }
   }
   
   ```

3. reducer

   ```java
   public class FlowSortReducer extends Reducer<FlowBean,Text, Text,FlowBean> {
       @Override
       protected void reduce(FlowBean key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
           //1.遍历集合，取出k3，并将k3和v3写入上下文
           for ( Text value : values){
               context.write(value,key);
           }
   
       }
   }
   
   ```

4. jobMain

   ```java
   public class JobMain extends Configured implements Tool {
   
       //该方法用于指定一个job任务
       @Override
       public int run(String[] strings) throws Exception {
           //1.创建job任务对象
           Job job = Job.getInstance(super.getConf(), "mapreduce_flowBean");
   
           //2.配置job任务对象
           job.setInputFormatClass(TextInputFormat.class);//1.读文件的方式使用InputFormat的子类TextInputFormat
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\FlowCount\\FlowCountOut"));//2.指定读取源文件对象地址,将上次的输出作为本次的输入
   
           job.setMapperClass(FlowSortMapper.class);//3.指定map阶段的处理方式
           job.setMapOutputKeyClass(FlowBean.class);//4.设置map阶段k2的类型
           job.setMapOutputValueClass(Text.class);//5.设置map阶段v2的类型
   
   //        shuffle阶段(分区，排序，规约，分组)暂不做处理，此处采用默认方式
   
           //6.指定Reduce阶段的处理方式和数据类型
           job.setReducerClass(FlowSortReducer.class);
           //7.设置k3类型
           job.setOutputKeyClass(Text.class);
           //8.设置v3的类型
           job.setOutputValueClass(FlowBean.class);
   
           //9.设置输出类型
           job.setOutputFormatClass(TextOutputFormat.class);
           //10.设置输出路径
           Path path = new Path("file:///G:\\学习\\maven\\src\\main\\java\\FlowCount_sort\\FlowCount_sortOut");
           TextOutputFormat.setOutputPath(job,path);
   
           //11.等待任务结束
           boolean b1 = job.waitForCompletion(true);
           return b1 ? 0 : 1;//根据返回结果判断任务是否正常
       }
   
       public static void main(String[] args) throws Exception {
           //配置文件对象
           Configuration conf = new Configuration();
           //启动job任务
           int run = ToolRunner.run(conf, new JobMain(), args);
           System.exit(run);//退出任务
       }
   }
   
   ```

   

### 9.4案例：手机号分区

在案例9.2的基础上继续完善将不同的手机号分到不同的数据文件的当中去，需要自定义
分区来实现，这里我们自定义来模拟分区，将以下数字开头的手机号进行分开

1. 自定义FlowBean

   ```java
   public class FlowBean implements Writable {
       private Integer upFlow;// 上行数据包数
       private Integer downFlow;// 下行数据包数
       private Integer upCountFlow;//上行流量总和
       private Integer downCountFlow;//下行流量总和
   
       public Integer getUpFlow() {
           return upFlow;
       }
   
       public void setUpFlow(Integer upFlow) {
           this.upFlow = upFlow;
       }
   
       public Integer getDownFlow() {
           return downFlow;
       }
   
       public void setDownFlow(Integer downFlow) {
           this.downFlow = downFlow;
       }
   
       public Integer getUpCountFlow() {
           return upCountFlow;
       }
   
       public void setUpCountFlow(Integer upCountFlow) {
           this.upCountFlow = upCountFlow;
       }
   
       public Integer getDownCountFlow() {
           return downCountFlow;
       }
   
       public void setDownCountFlow(Integer downCountFlow) {
           this.downCountFlow = downCountFlow;
       }
   
       @Override
       public String toString() {
           return  upFlow +
                   "\t" + downFlow +
                   "\t" + upCountFlow +
                   "\t" + downCountFlow;
       }
   
       //序列化方法
       @Override
       public void write(DataOutput dataOutput) throws IOException {
           dataOutput.writeInt(upFlow);
           dataOutput.writeInt(downFlow);
           dataOutput.writeInt(upCountFlow);
           dataOutput.writeInt(downCountFlow);
       }
       //反序列化
       @Override
       public void readFields(DataInput dataInput) throws IOException {
           this.upFlow = dataInput.readInt();
           this.downFlow = dataInput.readInt();
           this.upCountFlow = dataInput.readInt();
           this.downCountFlow = dataInput.readInt();
       }
   }
   
   ```

2. 自定义分区规则partition

   ```java
   public class FlowCountPartition  extends Partitioner<Text,FlowBean> {
       /**
        * 该方法用来指定分区的规则
        * 135开头的数据到一个分区文件
        * 136开头的数据到一个分区文件
        * 147开头的数据到一个分区文件
        * @param text k2
        * @param flowBean v2
        * @param i reduceTask的个数
        * @return
        */
       @Override
       public int getPartition(Text text, FlowBean flowBean, int i) {
           // 1.获取手机号
           String phoneNum = text.toString();
           //2.判断手机号以什么开头，返回对应的分区别号（0-3）
           if (phoneNum.startsWith("135")){
               return 0;
           }else if(phoneNum.startsWith("136")){
               return 1;
           }else if (phoneNum.startsWith("137")){
               return 2;
           }else {
               return 3;
           }
       }
   }
   
   ```

3. mapper

   ```java
   public class FlowCountMapper extends Mapper<LongWritable, Text,Text, FlowBean> {
       /**
        * 将k1和 v1 转为 k2 和 v2
        * k1       v1
        * 0        1363157985059 	13600217502	00-1F-64-E2-E8-B1:CMCC	120.196.100.55	www.baidu.com	综合门户	19	128	1177	16852	200
        *
        * ------------
        *
        * k2               v2
        * 13600217502      FlowBean(19	128	1177	16852)
        */
       @Override
       protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
           //1.拆分行文本数据，得到手机号 --》k2
           String[] split = value.toString().split("\t");
           String phoneNum = split[1];
   
           //2.创建FlowBean对象，并从行文本数据拆分出流量的四个字段，并将四个字段赋值给FlowBean对象
           FlowBean flowBean = new FlowBean();
           flowBean.setUpFlow(Integer.parseInt(split[6]));
           flowBean.setDownFlow(Integer.parseInt(split[7]));
           flowBean.setUpCountFlow(Integer.parseInt(split[8]));
           flowBean.setDownCountFlow(Integer.parseInt(split[9]));
   
           //3.将k2 和  v2 写入上下文中
           context.write(new Text(phoneNum),flowBean);
   
       }
   }
   
   ```

4. reducer

   ```java
   public class FlowReducer extends Reducer<Text, FlowBean,Text, FlowBean> {
       @Override
       protected void reduce(Text key, Iterable<FlowBean> values, Context context) throws IOException, InterruptedException {
           //1.遍历集合，并将集合中对应的四个字段累加
            Integer upFlow = 0;// 上行数据包数
            Integer downFlow = 0;// 下行数据包数
            Integer upCountFlow = 0;//上行流量总和
            Integer downCountFlow = 0;//下行流量总和
           for (FlowBean value :values){
               upFlow += value.getUpFlow();
               downFlow += value.getDownFlow();
               upCountFlow += value.getUpCountFlow();
               downCountFlow += value.getDownCountFlow();
           }
           //2.创建FlowBean对象，并给对象赋值
           FlowBean flowBean = new FlowBean();
           flowBean.setUpFlow(upFlow);
           flowBean.setDownFlow(downFlow);
           flowBean.setUpCountFlow(upCountFlow);
           flowBean.setDownCountFlow(downCountFlow);
           //3.将k3和v3写入上下文中
           context.write(key,flowBean);
       }
   }
   
   ```

5. jobMain

   ```java
   public class JobMain extends Configured implements Tool {
   
       //该方法用于指定一个job任务
       @Override
       public int run(String[] strings) throws Exception {
           //1.创建job任务对象
           Job job = Job.getInstance(super.getConf(), "mapreduce_flowBean_partition");
           //如果打包运行出错，需要加如下代码
           job.setJarByClass(JobMain.class);
   
           //2.配置job任务对象
           job.setInputFormatClass(TextInputFormat.class);//1.读文件的方式使用InputFormat的子类TextInputFormat
           TextInputFormat.addInputPath(job,new Path("file:///G:\\学习\\maven\\src\\main\\java\\FlowCount_partition\\data_flow.dat"));//2.指定读取源文件对象地址。
   
           job.setMapperClass(FlowCountMapper.class);//3.指定map阶段的处理方式
           job.setMapOutputKeyClass(Text.class);//4.设置map阶段k2的类型
           job.setMapOutputValueClass(FlowBean.class);//5.设置map阶段v2的类型
   
   //        shuffle阶段(分区，排序，规约，分组)暂不做处理，此处采用默认方式
           //指定分区
           job.setPartitionerClass(FlowCountPartition.class);
   
           //6.指定Reduce阶段的处理方式和数据类型
           job.setReducerClass(FlowReducer.class);
           //7.设置k3类型
           job.setOutputKeyClass(Text.class);
           //8.设置v3的类型
           job.setOutputValueClass(FlowBean.class);
   
           //设置reduce的个数
           job.setNumReduceTasks(4);
   
           //9.设置输出类型
           job.setOutputFormatClass(TextOutputFormat.class);
           //10.设置输出路径
           Path path = new Path("file:///G:\\学习\\maven\\src\\main\\java\\FlowCount_partition\\FlowPartitionOut");
           TextOutputFormat.setOutputPath(job,path);
   
           //11.等待任务结束
           boolean b1 = job.waitForCompletion(true);
           return b1 ? 0 : 1;//根据返回结果判断任务是否正常
       }
   
       public static void main(String[] args) throws Exception {
           //配置文件对象
           Configuration conf = new Configuration();
           //启动job任务
           int run = ToolRunner.run(conf, new JobMain(), args);
           System.exit(run);//退出任务
       }
   }
   
   ```
