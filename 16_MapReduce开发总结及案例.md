# 16_MapReduce开发总结及案例

## 1.MapReeduce开发总结

在编写MapReduce程序时，需要考虑的几个方面：

1. 输入数据接口：InputFormat

   1. 默认使用的实现类是TextInputFormat
   2. TextInputFormat的功能逻辑是：一次读一行文本，然后将该行的起始偏移量作为key，行内容作为value返回
   3. KeyValueTextInputFormat每一行均为一条记录，被分隔符分割为key，value。默认分隔符是tab（\t）
   4. NlineInputFormat按照指定的行数N来划分切片
   5. CombinerTextInputFormat可以把多个小文件合并成一个切片处理，提高处理效率
   6. 用户还可以自定义InputFormat

2. 逻辑处理接口Mapper

   用户根据业务需求实现其中的三个方法map(),setuo(),cleanup()

3. Partitioner分区

   有默认实现HashPartitioner，逻辑是根据key的哈希值和numReduces来返回一个分区号

   ```
   key.hashCode()&Integer.MAXVALUE % numReduces
   ```

   如果业务上有特殊需求，可以自定义分区

4. Comparable排序

   1. 当我们用自定义的对象作为key来输出的时候，就必须要实现WritableComparable接口，重写其中的compareTo()方法
   2. 部分排序：对最终输出的每一个文件进行内部排序
   3. 全排序：对所有的数据进行排序，通常只有一个Reduce
   4. 二次排序：排序的条件有两个

5. Combiner合并

   Combiner合并可以提高程序执行效率，减少IO传输，但是使用时必须不能影响原有的业务处理结果

6. Reduce端分组：Groupingcomparator

   1. ReduceTask拿到输入数据（一个partition的所有数据）后，首先需要对数据进行分组，其分组的默认原则是key相同，然后对每一组kv数据调用一次reduce()方法，并且将这一组kv中的第一个kv的key作为参数传给Reduce的key，将这一组数据的value的迭代器传给Reduce()的values参数
   2. 利用上述机制，我们可以实现一个高效分组取最大值的逻辑
   3. 自定义一个bean对象用来封装我们的数据，然后改写其compareTo方法产生倒序排序的效果。然后自定义一个Groupingcomparator，将bean对象的分组逻辑改成按照我们的业务分组ID来分组（比如订单号）。这样我们要取的最大值就是Reduce()方法中传进来的key

7. 逻辑处理接口Reduce

   用户根据业务需求实现其中三个方法：reduce(),setup(),cleanup()

8. 输出数据接口：OutputFormat

   1. 默认实现类是TextOutputFormat，功能逻辑是：将每一个KV对目标文本文件中输出为一行
   2. SequenceFileOutputFormat将它的输出写为一个顺序文件。如果输出需要作为后续MapReduce任务的输入，这便是一种好的输出格式，因为它的格式紧凑，很容易被压缩
   3. 用户还可以自定义OutputFormat

## 2. Hadoop数据压缩

### 2.1 概述

1. 压缩技术能够有效减少底层存储系统（HDFS）读写字节数。压缩提高了网络带宽和磁盘空间的效率。在Hadoop下，尤其是数据规模很大和工作符在密集的情况下，使用数据压缩显得非常重要。在这种情况下，I/O操作和网络数据传输需要花费大量的时间。还有，Shuffle与Merge过程同样也面临着巨大的I/O压力。
2. 见于磁盘I/O和网络带宽是Hadoop的宝贵资源，**数据压缩对于节省资源、最小化磁盘I/O和网络传输非常有帮助**。不过，尽管压缩与解压操作的CPU开销不高，其性能的提升和资源的节省并非没有代价。
3. 如果磁盘I/O和网络带宽影响了MapReduce作业性能，在任意,apReduce阶段启用压缩都可以改善端到端处理时间并减少I/O和网络流量
4. 压缩**MapReduce的一种优化策略：通过压缩编码对Mapper或者Reducer的输出进行压缩，以减少磁盘I/O，**提高MR程序的运行速度（但相应**增加了cpu的运算负担**）
5. **注意：**压缩特性运用得当能提高性能，但运用不当也可能降低性能

**基本原则：**

1. 运算密集型的job，少用压缩
2. IO密集型的job，多用压缩

### 2.2 MR支持的压缩编码

| 压缩格式 | hadoop自带？ | 算法    | 文件扩展名 | 是否可切分 | 换成压缩格式后，原来的程序是否需要修改 |
| -------- | ------------ | ------- | ---------- | ---------- | -------------------------------------- |
| DEDAULT  | 是，直接使用 | DEFAULT | .deflate   | 否         | 和文本处理一样，不需要修改             |
| Gzip     | 是，直接使用 | DEFAULT | .gz        | 否         | 和文本处理一样，不需要修改             |
| bzip2    | 是，直接使用 | bzip2   | .bz2       | **是**     | 和文本处理一样，不需要修改             |
| LZO      | 否，需要安装 | LZO     | .lzo       | **是**     | **需要建索引，还需要指定输入格式**     |
| Snappy   | 否，需要安装 | Snappy  | .snappy    | 否         | 和文本处理一样，不需要修改             |

为了支持多种压缩/解压缩算法，Hadoop引入了编码/解码器，如下表所示

| 压缩格式 | 对应的编码/解码器                          |
| -------- | ------------------------------------------ |
| DEFLATE  | org.apache.hadoop.io.compress.DefaultCodec |
| gzip     | org.apache.hadoop.io.compress.GzipCodec    |
| bzip2    | org.apache.hadoop.io.compress.BZip2Codec   |
| LZO      | com.hadoop.compression.lzo.LzopCodec       |
| Snappy   | org.apache.hadoop.io.compress.SnappyCodec  |

压缩性能的比较

| 压缩算法 | 原始文件大小 | 压缩文件大小 | 压缩速度 | 解压速度 |
| -------- | ------------ | ------------ | -------- | -------- |
| gzip     | 8.3GB        | 1.8GB        | 17.5MB/S | 58MB/S   |
| bzip2    | 8.3GB        | 1.1GB        | 2.4MB/S  | 9.5MB/S  |
| LZO      | 8.3GB        | 2.9GB        | 49.3MB/S | 74.6MB/S |

### 2.3 压缩方式原则

1. Gzip压缩
   
   1. 优点
   
      压缩率比较高，而且压缩/解压缩的速度也比较快；hadoop本身支持，在应用中处理gzip格式的文件就和直接处理文本一样；大部分linux系统都自带gzip命令，使用方便
   
   2. 缺点：不支持split
   
      应用场景：当每个文件压缩之后在130M以内的（一个块大小内），都可以考虑用gzip压缩格式。例如说，一天或者一个小时的日志压缩成一个gzip文件，运行mapReduce程序的时候通过多个gzip文件达到并发。hive程序，streaming程序，和java写的mapReduce程序完全和文本处理一样，压缩之后原来的程序不需要做任何修改
   
2. Bzip2压缩

   1. 优点：

      支持split，具有很高的压缩率，比gzip压缩率都高；hadoop本身支持，但不支持native（java和c互操作的API接口）；在linux系统下自带bzip2命令，使用方便

   2. 缺点：压缩/解压缩速度慢；不支持native

   3. 应用场景：

      适合对速度要求不高，但需要较高的压缩率的时候，可以作为MapReduce作业的输出格式；或者输出之后的数据比较大，处理之后的数据需要压缩存档减少磁盘空间并且以后数据用的比较少的情况；或者对单个很大的文本文件向压缩减少存储空间，同时又需要支持split，而且兼容之前的应用程序（即应用程序不要要修改）的情况

3. Lzo压缩

   1. 优点：

      压缩/解压的速度比较快，合理的压缩率；支持split，是hadoop中最流行的压缩格式；可以在linux西永下安装lzop命令，使用方便

   2. 缺点：

      压缩率比gzip要低一些；hadoop本身不支持，需要安装；在应用中对lzo格式的文件需要做一些特殊处理（为了支持split需要建索引，还需要指定inputformat为lzo格式）。

   3. 应用场景：

      一个很大的文本文件，压缩之后还大于200M以上的可以考虑，而且单个文件越大，lzo优点越明显

4. Snappy压缩

   1. 优点：高速压缩速度和合理的压缩率
   2. 缺点：不支持split；压缩率比gzip要低；hadoop本身不支持，需要安装；
   3. 应用场景：当MapReduce作业的Map输出的数据比较大的时候，作为Map到Reduce的中间数据的压缩格式；或者作为一个MapReduce作业的输出和另外一个MapReduce作业的输入。

### 2.4 压缩位置的选择

压缩可以在MapReduce作用的任意阶段启用

### 2.5 压缩配置参数

要在Hadoop中启用压缩，可以配置如下参数

| 参数                                                         | 默认值                                                       | 阶段        | 建议                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ----------- | -------------------------------------------- |
| io.compression.codecs  （在core-site.xml中配置）             | org.apache.hadoop.io.compress.DefaultCodec, org.apache.hadoop.io.compress.GzipCodec, org.apache.hadoop.io.compress.BZip2Codec | 输入压缩    | Hadoop使用文件扩展名判断是否支持某种编解码器 |
| mapreduce.map.output.compress（在mapred-site.xml中配置）     | false                                                        | mapper输出  | 这个参数设为true启用压缩                     |
| mapreduce.map.output.compress.codec（在mapred-site.xml中配置） | org.apache.hadoop.io.compress.DefaultCodec                   | mapper输出  | 使用LZO或snappy编解码器在此阶段压缩数据      |
| mapreduce.output.fileoutputformat.compress（在mapred-site.xml中配置） | false                                                        | reducer输出 | 这个参数设为true启用压缩                     |
| mapreduce.output.fileoutputformat.compress.codec（在mapred-site.xml中配置） | org.apache.hadoop.io.compress. DefaultCodec                  | reducer输出 | 使用标准工具或者编解码器，如gzip和bzip2      |
| mapreduce.output.fileoutputformat.compress.type（在mapred-site.xml中配置） | RECORD                                                       | reducer输出 | SequenceFile输出使用的压缩类型：NONE和BLOCK  |

## 3.压缩/解压缩案例

### 3.1 数据流的压缩和解压缩

CompressionCodec有两个方法可以用于轻松地压缩或解压缩数据。想要对正在被写入的一个输出流的数据进行压缩，我们可以使用createOutputStream(OutputStreamout)方法创建一个CompressionOutputStream，将其以压缩格式写入底层的流。相反，要想对从输入流读取而来的数据进行压缩，则调用createInputStream(InputStreamin)函数，从而获得一个CompressionInputStream，从而从底层的流读取未压缩的数据

### 3.2 java方式压缩和解压缩

```java
public class TestCompress {
    public static void main(String[] args) throws Exception{
//        compress("G:\\学习\\maven\\src\\main\\java\\Compress\\kkkk.doc","org.apache.hadoop.io.compress.BZip2Codec");

        decompress("G:\\学习\\maven\\src\\main\\java\\Compress\\kkkk.doc.bz2",".doc");
    }

    /**
     * 压缩方法
     * @param filename 文件路径+文件名
     * @param methon 解码器
     */
    private static void compress(String filename,String methon) throws Exception {
        //创建输入流
        FileInputStream fis = new FileInputStream(new File(filename));
        //通过反射类找到解码器的类
        Class<?> codeClass = Class.forName(methon);
        //通过反射工具类找到解码器对象，需要用到配置conf对象
        CompressionCodec codec = (CompressionCodec) ReflectionUtils.newInstance(codeClass, new Configuration());
        //创建输出流
        FileOutputStream fos = new FileOutputStream(new File(filename + codec.getDefaultExtension()));
        //获得解码器输出对象
        CompressionOutputStream cos = codec.createOutputStream(fos);
        //流拷贝
        IOUtils.copyBytes(fis,cos,5*1024*1024,false);
        //关闭流
        cos.close();
        fos.close();
        fis.close();
    }

    /**
     * 解开压缩
     * @param filename 文件路径 + 文件名
     * @param decoded 后缀
     * @throws Exception
     */
    private static void decompress(String filename,String decoded)throws Exception{
        //获取factory实例
        CompressionCodecFactory factory = new CompressionCodecFactory(new Configuration());
        CompressionCodec codec = factory.getCodec(new Path(filename));
        if (codec == null){
            System.out.println(filename);
            return;
        }
        //解压缩的输入
        CompressionInputStream cis = codec.createInputStream(new FileInputStream(new File(filename)));
        //输出流
        FileOutputStream fos = new FileOutputStream(new File(filename + "." + decoded));
        //流拷贝
        IOUtils.copyBytes(cis,fos,5*1024*1024,false);
        //关闭流
        cis.close();
        fos.close();
    }

}

```

### 3.3 Map端采用压缩

即使你的MapReduce的输入输出文件都是未压缩的文件，你仍然可以对map任务的中间结果输出做压缩，因为它要写在硬盘并且通过网络传输到Reduce节点，对其压缩可以提高很多性能，这些工作只要设置两个属性即可 

给大家提供的hadoop源码支持的压缩格式有：BZip2Codec 、DefaultCodec

```java
public class WordCountDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        args = new String[]{"G:\\学习\\maven\\src\\main\\java\\Compress\\wordcount.txt","G:\\学习\\maven\\src\\main\\java\\Compress\\out"};
        Configuration configuration = new Configuration();

        // 开启map端输出压缩
        configuration.setBoolean("mapreduce.map.output.compress", true);
        // 设置map端输出压缩方式
        configuration.setClass("mapreduce.map.output.compress.codec", BZip2Codec.class, CompressionCodec.class);

        Job job = Job.getInstance(configuration);

        job.setJarByClass(WordCountDriver.class);

        job.setMapperClass(CompressMapper.class);
        job.setReducerClass(CompressReduce.class);

        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        boolean result = job.waitForCompletion(true);

        System.exit(result ? 1 : 0);
    }
}
```

mapper保持不变

```java
public class CompressMapper extends Mapper<LongWritable,Text, Text, IntWritable> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        //1.获取一行
        String line = value.toString();
        //2.切割
        String[] words = line.split(" ");
        //3.循环写出
        for(String word : words){
            context.write(new Text(word),new IntWritable(1));
        }

    }
}

```

reduce保持不变

```java
public class CompressReduce extends Reducer<Text, IntWritable, Text,IntWritable> {
    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        int count = 0;
        //1.汇总
        for(IntWritable value:values){
            count+= value.get();
        }
        //2.输出
        context.write(key,new IntWritable(count));
    }
}
```

### 3.4 Reduce输出端采用压缩

基于wordcount案例处理

```java
public class WordCountDriver1 {
    public static void main(String[] args) throws Exception {
        args = new String[]{"G:\\学习\\maven\\src\\main\\java\\Compress\\wordcount.txt","G:\\学习\\maven\\src\\main\\java\\Compress\\out"};
        Configuration configuration = new Configuration();

        Job job = Job.getInstance(configuration);

        job.setJarByClass(WordCountDriver1.class);

        job.setMapperClass(CompressMapper.class);
        job.setReducerClass(CompressReduce.class);

        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        // 设置reduce端输出压缩开启
        FileOutputFormat.setCompressOutput(job, true);

        // 设置压缩的方式
        FileOutputFormat.setOutputCompressorClass(job, BZip2Codec.class);
//	    FileOutputFormat.setOutputCompressorClass(job, GzipCodec.class);
//	    FileOutputFormat.setOutputCompressorClass(job, DefaultCodec.class);

        boolean result = job.waitForCompletion(true);

        System.exit(result?1:0);
    }
}

```

mapper和Reducer保持不变