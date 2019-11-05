# 13_HDFSAPI操作

## 1.maven准备

```xml
<dependencies>
<dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.8.4</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.8.4</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.8.4</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.8.4</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.10</version>
        </dependency>

        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.7</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/junit/junit -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
   </dependencies>
```

## 2.本地配置Hadoop环境

在windows系统需要配置hadoop运行环境，否则直接运行代码会出现以下问题:

1. 解压hadoop2.8.4文件至需要安装的全英文目录下

2. 配置hadoop的环境变量 HADOOP_HOME

   将以下配置到path变量中

   ```
   D:\hadoop-2.8.4\hadoop-2.8.4\sbin
   D:\hadoop-2.8.4\hadoop-2.8.4\bin
   ```

   注：按照自己的安装路径进行配置

## 3.获取文件系统

```java
/**
     * 打印本地hadoop地址值
     * IO的方式写代码
     */
    @Test
    public void intiHDFS() throws IOException {
        //F2 可以快速的定位错误
        // alt + enter自动找错误
        //1.创建配信信息对象 ctrl + alt + v  后推前  ctrl + shitl + enter 补全
        Configuration conf = new Configuration();

        //2.获取文件系统
        FileSystem fs = FileSystem.get(conf);

        //3.打印文件系统
        System.out.println(fs.toString());
    }
```

## 4.遍历HDFS所有文件

```java
 //遍历HDFS所有文件
    @Test
    public void listFiles() throws Exception{
        //1.获取filesystem对象
        FileSystem fileSystem = FileSystem.get(new URI("hdfs://bigdata111:9000"), new Configuration());
        //2.获取RemoteIterator 得到所有的文件或者文件夹，第一个参数指定遍历的路径，第二个can阿叔表示是否要递归遍历
        RemoteIterator<LocatedFileStatus> lists = fileSystem.listFiles(new Path("/"), true);
        //3.循环
        while (lists.hasNext()){
            LocatedFileStatus next = lists.next();
            System.out.println(next.getPath().toString());

            //文件的block信息
            BlockLocation[] blockLocations = next.getBlockLocations();
            System.out.println("block数： "+blockLocations.length);
        }
        fileSystem.close();
    }
```

## 5.HDFS上创建文件夹

```java
/**
     * hadoop fs -mkdir /xinshou
     */
    @Test
    public void mkmdirHDFS() throws Exception {
        //1.创新配置信息对象
        Configuration configuration = new Configuration();

        //2.链接文件系统
        //final URI uri  地址
        //final Configuration conf  配置
        //String user   Linux用户
        FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), configuration, "root");

        //3.创建目录
        fs.mkdirs(new Path("hdfs://bigdata111:9000/Good/Goog/Study"));

        //4.关闭
        fs.close();
        System.out.println("创建文件夹成功");
    }
```

## 6. 文件下载

```java
  /**
     * hadoop fs -get /HDFS文件系统
     * @throws Exception
     */
    @Test
    public void getFileFromHDFS() throws Exception {
        //1.创建配置信息对象  Configuration:配置
        Configuration conf = new Configuration();

        //2.找到文件系统
        //final URI uri     ：HDFS地址
        //final Configuration conf：配置信息
        // String user ：Linux用户名
        FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");

        //3.下载文件
        //boolean delSrc:是否将原文件删除
        //Path src ：要下载的路径
        //Path dst ：要下载到哪
        //boolean useRawLocalFileSystem ：是否校验文件
        fs.copyToLocalFile(false,new Path("hdfs://bigdata111:9000/README.txt"),
                new Path("F:\\date\\README.txt"),true);

        //4.关闭fs
        //alt + enter 找错误
        //ctrl + alt + o  可以快速的去除没有用的导包
        fs.close();
        System.out.println("下载成功");
    }
```

## 7.文件上传

```java

 /**
     * 上传代码
     * 注意：如果上传的内容大于128MB,则是2块
     */
    @Test
    public void putFileToHDFS() throws Exception {
        //注：import org.apache.hadoop.conf.Configuration;
        //ctrl + alt + v 推动出对象
        //1.创建配置信息对象
        Configuration conf = new Configuration();

        //2.设置部分参数
        conf.set("dfs.replication","2");

        //3.找到HDFS的地址
        FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");

        //4.上传本地Windows文件的路径
        Path src = new Path("D:\\hadoop-2.7.2.rar");
         //5.要上传到HDFS的路径
        Path dst = new Path("hdfs://bigdata111:9000/Andy");

        //6.以拷贝的方式上传，从src -> dst
        fs.copyFromLocalFile(src,dst);

        //7.关闭
        fs.close();

        System.out.println("上传成功");
}
```

## 8.小文件合并至HDFS

```java
//小文件合并至HDFS
    @Test
    public void mergeFile() throws Exception{
        //1.获取FileSystem系统
        FileSystem fileSystem = FileSystem.get(new URI("hdfs://bigdata111:9000"),new Configuration());
        //2.获取hdfs大文件输出流
        FSDataOutputStream OutputStream = fileSystem.create(new Path("/hello.txt"));
        //3.获取一个本地文件系统
        LocalFileSystem local = FileSystem.getLocal(new Configuration());
        //4.获取本地文件夹下所有文件的详情
        FileStatus[] fileStatuses = local.listStatus(new Path("G:\\file"));
        //5.遍历每个文件，获取每个文件的输入流
        for (FileStatus fileStatus : fileStatuses) {
            FSDataInputStream open = local.open(fileStatus.getPath());
            //6.将小文件复制到大文件
            IOUtils.copy(open,OutputStream);
            //7.关闭流
            IOUtils.closeQuietly(open);
        }
        IOUtils.closeQuietly(OutputStream);
        local.close();
        fileSystem.close();
    }
```

## 9.HDFS文件夹删除

```java
 /**
     * hadoop fs -rm -r /文件
     */
    @Test
    public void deleteHDFS() throws Exception {
        //1.创建配置对象
        Configuration conf = new Configuration();

        //2.链接文件系统
        //final URI uri, final Configuration conf, String user
        //final URI uri  地址
        //final Configuration conf  配置
        //String user   Linux用户
        FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");

        //3.删除文件
        //Path var1   : HDFS地址
        //boolean var2 : 是否递归删除
        fs.delete(new Path("hdfs://bigdata111:9000/a"),false);

        //4.关闭
        fs.close();
        System.out.println("删除成功啦");
    }
```

## 10.HDFS文件名更改

```java
	@Test
	public void renameAtHDFS() throws Exception{
		// 1 创建配置信息对象
		Configuration configuration = new Configuration();
		
		FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"),configuration, "itstar");
		
		//2 重命名文件或文件夹
		fs.rename(new Path("hdfs://bigdata111:9000/user/itstar/hello.txt"), new Path("hdfs://bigdata111:9000/user/itstar/hellonihao.txt"));
	fs.close();
	}
```

## 11.HDFS文件详情查看

```java
  /**
     * 查看【文件】名称、权限等
     */
    @Test
    public void readListFiles() throws Exception {
        //1.创建配置对象
        Configuration conf = new Configuration();

        //2.链接文件系统
        FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");

        //3.迭代器
        RemoteIterator<LocatedFileStatus> listFiles = fs.listFiles(new Path("/"), true);

        //4.遍历迭代器
        while (listFiles.hasNext()){
            //一个一个出
            LocatedFileStatus fileStatus = listFiles.next();

            //名字
            System.out.println("文件名：" + fileStatus.getPath().getName());
            //块大小
            System.out.println("大小：" + fileStatus.getBlockSize());
            //权限
            System.out.println("权限：" + fileStatus.getPermission());
            System.out.println(fileStatus.getLen());


            BlockLocation[] locations = fileStatus.getBlockLocations();

            for (BlockLocation bl:locations){
                System.out.println("block-offset:" + bl.getOffset());
                String[] hosts = bl.getHosts();
                for (String host:hosts){
                    System.out.println(host);
                }
            }

            System.out.println("------------------华丽的分割线----------------");
        }
```

## 12.HDFS文件和文件夹判断

```java
   /**
     * 判断是否是个文件还是目录，然后打印
     * @throws Exception
     */
    @Test
    public void judge() throws Exception {
        //1.创建配置文件信息
        Configuration conf = new Configuration();

        //2.获取文件系统
        FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");

        //3.遍历所有的文件
        FileStatus[] liststatus = fs.listStatus(new Path("/Andy"));
        for(FileStatus status :liststatus)
        {
            //判断是否是文件
            if (status.isFile()){
                //ctrl + d:复制一行
                //ctrl + x 是剪切一行，可以用来当作是删除一行
                System.out.println("文件:" + status.getPath().getName());
            } else {
                System.out.println("目录:" + status.getPath().getName());
            }
        }
    }
```

## 13.通过IO流操作HDFS

### 13.1HDFS文件上传

```java
 /**
     * IO流方式上传
     *
     * @throws URISyntaxException
     * @throws FileNotFoundException
     * @throws InterruptedException
     */
    @Test
    public void putFileToHDFSIO() throws URISyntaxException, IOException, InterruptedException {
        //1.创建配置文件信息
        Configuration conf = new Configuration();

        //2.获取文件系统
        FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");

        //3.创建输入流
        FileInputStream fis = new FileInputStream(new File("F:\\date\\Sogou.txt"));

        //4.输出路径
        //注意：不能/Andy  记得后边写个名 比如：/Andy/Sogou.txt
        Path writePath = new Path("hdfs://bigdata111:9000/Andy/Sogou.txt");
        FSDataOutputStream fos = fs.create(writePath);

        //5.流对接
        //InputStream in    输入
        //OutputStream out  输出
        //int buffSize      缓冲区
        //boolean close     是否关闭流
        try {
            IOUtils.copyBytes(fis,fos,4 * 1024,false);
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            IOUtils.closeStream(fos);
            IOUtils.closeStream(fis);
            System.out.println("上传成功啦");
        }
    }
```

### 13.2 HDFS文件下载

```java
/**
     * IO读取HDFS到控制台
     *
     * @throws URISyntaxException
     * @throws IOException
     * @throws InterruptedException
     */
    @Test
    public void getFileToHDFSIO() throws URISyntaxException, IOException, InterruptedException {
        //1.创建配置文件信息
        Configuration conf = new Configuration();

        //2.获取文件系统
        FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");

        //3.读取路径
        Path readPath = new Path("hdfs://bigdata111:9000/Andy/Sogou.txt");

        //4.输入
        FSDataInputStream fis = fs.open(readPath);

        //5.输出到控制台
        //InputStream in    输入
        //OutputStream out  输出
        //int buffSize      缓冲区
        //boolean close     是否关闭流
        IOUtils.copyBytes(fis,System.out,4 * 1024 ,true);
    }
```

### 13.3 定位文件读取

1. 下载第一块

   ```java
   /**
        * IO读取第一块的内容
        *
        * @throws Exception
        */
       @Test
       public void  readFlieSeek1() throws Exception {
           //1.创建配置文件信息
           Configuration conf = new Configuration();
   
           //2.获取文件系统
           FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");
   
           //3.输入
           Path path = new Path("hdfs://bigdata111:9000/Andy/hadoop-2.7.2.rar");
           FSDataInputStream fis = fs.open(path);
   
           //4.输出
           FileOutputStream fos = new FileOutputStream("F:\\date\\readFileSeek\\A1");
   
           //5.流对接
           byte[] buf = new byte[1024];
           for (int i = 0; i < 128 * 1024; i++) {
               fis.read(buf);
               fos.write(buf);
           }
   
           //6.关闭流
           IOUtils.closeStream(fos);
           IOUtils.closeStream(fis);
       }
   ```

2. 下载第二块

   ```java
   /**
        * IO读取第二块的内容
        *
        * @throws Exception
        */
       @Test
       public void readFlieSeek2() throws Exception {
           //1.创建配置文件信息
           Configuration conf = new Configuration();
   
           //2.获取文件系统
           FileSystem fs = FileSystem.get(new URI("hdfs://bigdata111:9000"), conf, "root");
   
           //3.输入
           Path path = new Path("hdfs://bigdata111:9000/Andy/hadoop-2.7.2.rar");
           FSDataInputStream fis = fs.open(path);
   
           //4.输出
           FileOutputStream fos = new FileOutputStream("F:\\date\\readFileSeek\\A2");
   
           //5.定位偏移量/offset/游标/读取进度 (目的：找到第一块的尾巴，第二块的开头)
           fis.seek(128 * 1024 * 1024);
   
           //6.流对接
           IOUtils.copyBytes(fis, fos, 1024);
   
           //7.关闭流
           IOUtils.closeStream(fos);
           IOUtils.closeStream(fis);
       }
   ```

3. 合并文件

   在window命令窗口中执行

   type A2 >> A1  然后更改后缀为rar即可

   

## 14.一致性模型

```java
@Test
	public void writeFile() throws Exception{
		// 1 创建配置信息对象
		Configuration configuration = new Configuration();
		fs = FileSystem.get(configuration);
		
		// 2 创建文件输出流
		Path path = new Path("F:\\date\\H.txt");
		FSDataOutputStream fos = fs.create(path);
		
		// 3 写数据
		fos.write("hello Andy".getBytes());
        // 4 一致性刷新
		fos.hflush();
		
		fos.close();
	}
```

总结：

写入数据时，如果希望数据被其他client立即可见，调用如下方法

FSDataOutputStream. hflush ();		//清理客户端缓冲区数据，被其他client立即可见