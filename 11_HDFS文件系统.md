# 11_HDFS文件系统

## 1.HDFS概念

HDFS，是一个文件系统，全称为：Hadoop Distributed File System。用于存储文件通过，目录树来定位文件。其次，这是一个分布式的文件系统，由很多服务器联合起来实现其功能，集群中的服务器各自有各自的角色。

### 1.1组成

1. HDFS集群包括：NameNode和DataNode以及Secondary NameNode。
2. NameNode负责管理整个文件新系统的元数据，以及每一个路径（文件）所对应的数据块信息。
3. DataNode负责管理用户的文件数据块，每一个数据库块都可以在多个datanode上存储多个副本。
4. Secondary NameNode用啦监控HDFS状态的辅助后台程序，没隔一段时间获取HDFS元数据的快照。

### 1.2HDFS文件块大小

- HDFS中的文件在物理上是分块存储（block），块的代销可以通过配置参数（dfs.blocksize）来规定，默认大小在hadoop2.X版本是128M，老版本则是64M。
- HDFS的块比磁盘的块大，其目的是为了最小化寻址开销。如果块设置的足够大，从磁盘传输数据的时间会明显大于定位这个块开始位置所需要的时间，因而传输一个由多个块组成的文件的时间取决于磁盘传输速率。
- 如果寻址时间约为10ms，而传输速率为100MB/s，为了是寻址时间占传输时间的1%，我们需要将块大小设置约为100MB。默认的块大小为128MB
- 块的大小：10ms*100*100M/s = 100M

![](C:\Users\宋天\Desktop\大数据\img\HDFS块大小.png)

### 1.3HDFS应用场景

1. 适合的应用场景
   - 存储非常大的文件，需要高吞吐量，对延时没有要求
   - 采用流式的数据访问方式：即一次写入，多次读取，数据集经常从数据源生成或者拷贝一次，然后在其上做很多分析工作
   - 运行于商业硬件上：Hadoop不需要特别贵的机器，可运行于普通廉价机器，可以节约成本
   - 需要高容错性
   - 为数据存储提供所需的扩展能力
2. 不适合的应用场景
   - 低延时的数据访问对延时要求在毫秒级别的应用，不适合财通HDFS。HDFS是为高吞吐数据传输设计的，因此可能牺牲延迟
   - 大量小文件。文件的元数据保存在NameNode的内存中，整个文件系统的文件数量会受限于NameNode的内存大小。经验而言，一个文件/目录/文件快一般占有150字节的元数据内存空间。如果有100万个文件，每个文件占用1个文件快，则需要大约300M的内存。因此10亿级别的文件数量在现有商用机器上难以支持。
   - 多方读写，需要任意的文件修改HDFS采用追加的方式写入数据。不支持文件任意offset的修改，不支持多个写入器

## 2.HDFS的架构

概念：HDFS是一个主/从（Mater/Slave）体系结构。

HDFS由四部分组成。HDFSClient、NameNode、DataNode、Secondary NameNode。

1. client：就是客户端
   - 文件切分。文件上传HDFS的时候，client将文件切分为一个一个的Block，然后进行存储
   - 与NameNode交互，获取文件的位置信息
   - 与DataNode交互，读取或写入数据
   - client提供一些命令来管理和访问HDFS，比如启动或者关闭HDFS
2. NameNode：就是master（主人），是一个主管、管理者
   - 管理HDFS的名称空间
   - 观念里数据块（Block）映射信息
   - 配置副本策略
   - 处理客户端读写请求
3. DataNode：就是slave（奴隶）。NameNode下达命令，DataNode执行实际的操作
   - 存储实际的数据块
   - 执行数据块的读/写操作
4. Secondary（次要的）NameNode ：并不是NameNode的备用。当NameNode挂掉的时候，它并不能马上替换NameNode并提供服务
   - 辅助NameNode，分担其工作量
   - 定期合并fsimage和fsedits，并推送给NameNode
   - 在紧急情况下，可辅助恢复NameNode

## 3.NameNode工作机制

### 3.1NameNode&Secondary NameNode工作机制

![](C:\Users\宋天\Desktop\大数据\img\NameNode&Secondary NameNode工作机制.png)

解读：

1. 第一阶段：namenode启动

   1. 第一次启动namenode格式化后，创建fsimage和edits文件。如果不是第一次启动，直接加载编辑日志和镜像文件到内存。
   2. 客户端对元数据进行增删改的请求
   3. namenode记录操作日志，更新滚动日志。
   4. namenode在内存中对数据进行增删改查

2. 第二阶段：Secondary NameNode工作

   1. Secondary NameNode询问namenode是否需要checkpoint。直接带回namenode是否检查结果。
   2. Secondary NameNode请求执行checkpoint。
   3. namenode滚动正在写的edits日志
   4. 将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode
   5. econdary NameNode加载编辑日志和镜像文件到内存，并合并。
   6. 生成新的镜像文件fsimage.chkpoint
   7. 拷贝fsimage.chkpoint到namenode
   8. namenode将fsimage.chkpoint重新命名成fsimage

3. web端访问SecondaryNameNode

   1. 启动集群
   2. 浏览器中输入：http://bigdata111:50090/status.html
   3. 查看SecondaryNameNode信息

4. chkpoint检查时间参数设置

   1. 通常情况下，SecondaryNameNode每隔一小时执行一次。

      hdfs-default.xml

      ```xml
      <property>
        <name>dfs.namenode.checkpoint.period</name>
        <value>3600</value>
      </property>
      ```

   2. 分钟检查一次操作次数，当操作次数达到1百万时，SecondaryNameNode执行一次。

      ```xml
      <property>
        <name>dfs.namenode.checkpoint.txns</name>
        <value>1000000</value>
      <description>操作动作次数</description>
      </property>
      
      <property>
        <name>dfs.namenode.checkpoint.check.period</name>
        <value>60</value>
      <description> 1分钟检查一次操作次数</description>
      </property>
      ```

### 3.2 HDFS 的元数据辅助管理

概念：namenode被格式化之后，将在/opt/module/hadoop-2.8.4/data/dfs/name/current目录中产生如下文件

```
edits_0000000000000000000
fsimage_0000000000000000000.md5
seen_txid
VERSION
```

解读：

1. Fsimage文件：
   - NameNode 中关于元数据的镜像, 一般称为检查点, fsimage 存放了一份比较完整的元数据信息
   - 因为 fsimage 是 NameNode 的完整的镜像, 如果每次都加载到内存生成树状拓扑结构，这是非常耗内存和CPU, 所以一般开始时对 NameNode 的操作都放在 edits 中
   - fsimage 内容包含了 NameNode 管理下的所有 DataNode 文件及文件 block 及 block 所在的 DataNode 的元数据信息
   - 随着 edits 内容增大, 就需要在一定时间点和 fsimage 合并
2. Edits文件：
   - edits 存放了客户端最近一段时间的操作
   - 客户端对 HDFS 进行写文件时会首先被记录在 edits 文件中
   - edits 修改时元数据也会更新
3. seen_txid文件保存的是一个数字，就是最后一个edits_的数字
4. 每次Namenode启动的时候都会将fsimage文件读入内存，并从00001开始到seen_txid中记录的数字依次执行每个edits里面的更新操作，保证内存中的元数据信息是最新的、同步的，可以看成Namenode启动的时候就将fsimage和edits文件进行了合并。

**3.1.1 oiv查看fsimage文件**

1. 查看oiv和oev命令

   ```
   [root@bigdata111 current]$ hdfs
   oiv                  apply the offline fsimage viewer to an fsimage
   oev                  apply the offline edits viewer to an edits file
   ```

2. 基本语法

   hdfs oiv -p 文件类型 -i镜像文件 -o 转换后文件输出路径

3. 案例实操

   ```
   [root@bigdata111 current]$ pwd
   /opt/module/hadoop-2.7.2/data/tmp/dfs/name/current
   
   [root@bigdata111 current]$ hdfs oiv -p XML -i fsimage_0000000000000000025 -o /opt/fsimage.xml
   
   [root@bigdata111 current]$ cat /opt/fsimage.xml
   ```

   将显示的xml文件内容拷贝到IDEA中创建的xml文件中，并格式化(ctrl+L)，也可以直接cat查看文件
   
   具体信息如下：
   
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <EDITS>
     <EDITS_VERSION>-63</EDITS_VERSION>
     <RECORD>
       <OPCODE>OP_START_LOG_SEGMENT</OPCODE>
       <DATA>
         <TXID>5</TXID>
       </DATA>
     </RECORD>
     <RECORD>
       <OPCODE>OP_ADD</OPCODE>
       <DATA>
         <TXID>6</TXID>
         <LENGTH>0</LENGTH>
         <INODEID>16386</INODEID>
         <PATH>/README.txt._COPYING_</PATH>
         <REPLICATION>3</REPLICATION>
         <MTIME>1553348465857</MTIME>
         <ATIME>1553348465857</ATIME>
         <BLOCKSIZE>134217728</BLOCKSIZE>
         <CLIENT_NAME>DFSClient_NONMAPREDUCE_-2031979444_1</CLIENT_NAME>
         <CLIENT_MACHINE>192.168.1.112</CLIENT_MACHINE>
         <OVERWRITE>true</OVERWRITE>
         <PERMISSION_STATUS>
           <USERNAME>root</USERNAME>
           <GROUPNAME>supergroup</GROUPNAME>
           <MODE>420</MODE>
         </PERMISSION_STATUS>
         <RPC_CLIENTID>fcba804f-44e8-42a8-a375-6bf4a8c242a5</RPC_CLIENTID>
         <RPC_CALLID>3</RPC_CALLID>
       </DATA>
     </RECORD>
   </EDITS>
   ```
   
   
   
   ```
   每个RECORD记录了一次操作，如：
   OP_ADD代表添加文件操作、OP_MKDIR代表创建目录操作。里面还记录了
   文件路径（PATH）
   修改时间（MTIME）
   添加时间（ATIME）
   客户端名称（CLIENT_NAME）
   客户端地址（CLIENT_MACHINE）
   权限（PERMISSION_STATUS）等非常有用的信息
   ```
   
   

**3.1.2oev查看edits文件**

1. 基本语法

   `hdfs oev -p 文件类型 -i编辑日志 -o 转换后文件输出路径`

   ```
   -p	–processor <arg>   指定转换类型: binary (二进制格式), xml (默认，XML格式),stats
   -i	–inputFile <arg>     输入edits文件，如果是xml后缀，表示XML格式，其他表示二进制
   -o 	–outputFile <arg> 输出文件，如果存在，则会覆盖
   ```

2. 案例实操

   ```
   [root@bigdata111 current]$ hdfs oev -p XML -i edits_0000000000000000012-0000000000000000013 -o /opt/edits.xml -p stats
   [root@bigdata111 current]$ cat /opt/edits.xml
   ```

   将显示的xml文件内容拷贝到IDEA中创建的xml文件中，并格式化(ctrl+L)，也可以直接cat查看文件

**3.1.3SecondaryNameNode 如何辅助管理 fsimage 与 edits 文件**

当 Hadoop 的集群当中, NameNode的所有元数据信息都保存在了 FsImage 与 Eidts 文件当中,这两个文件就记录了所有的数据的元数据信息, 元数据信息的保存目录配置在了 **hdfs-site.xml** 

```xml
<property>
    <name>dfs.namenode.name.dir</name>    
    <value>     		file:///export/servers/hadoop2.7.5/hadoopDatas/namenodeDatas,:///export/servers/hadoop-2.7.5/hadoopDatas/namenodeDatas2
    </value>
</property>
<property>
     <name>dfs.namenode.edits.dir</name>
     <value>file:///export/servers/hadoop-2.7.5/hadoopDatas/nn/edits</value
</property>
```

注：可不进行配置。

1. SecondaryNameNode 定期合并 fsimage 和 edits, 把 edits 控制在一个范围内

2. 配置 SecondaryNameNode

   - 修改 core-site.xml  这一步不做配置保持默认也可以

     ```xml
     <!-- 多久记录一次 HDFS 镜像, 默认 1小时 -->
     <property>
       <name>fs.checkpoint.period</name>
       <value>3600</value>
     </property>
     <!-- 一次记录多大, 默认 64M -->
     <property>
       <name>fs.checkpoint.size</name>
       <value>67108864</value>
     </property
     ```

3. SecondaryNameNode工作原理

   ![](C:\Users\宋天\Desktop\大数据\img\SecondaryNameNode工作原理.PNG)

   - SecondaryNameNode 通知 NameNode 切换 editlog
   - SecondaryNameNode 从 NameNode 中获得 fsimage 和 editlog(通过http方式)
   - SecondaryNameNode 将 fsimage 载入内存, 然后开始合并 editlog, 合并之后成为新的fsimage
   - SecondaryNameNode 将新的 fsimage 发回给 NameNode
   - NameNode 用新的 fsimage 替换旧的 fsimage

4. 特点：

   1. 完成合并的是 SecondaryNameNode, 会请求 NameNode 停止使用 edits, 暂时将新写操作放入一个新的文件中 edits.new
   2. SecondaryNameNode 从 NameNode 中通过 Http GET 获得 edits, 因为要和 fsimage 合并, 所以也是通过 Http Get 的方式把 fsimage 加载到内存, 然后逐一执行具体对文件系统的操作, 与 fsimage 合并, 生成新的 fsimage, 然后通过 Http POST 的方式把 fsimage 发送给NameNode. NameNode 从 SecondaryNameNode 获得了 fsimage 后会把原有的 fsimage 替换为新的 fsimage, 把 edits.new 变成 edits. 同时会更新 fstime
   3. Hadoop 进入安全模式时需要管理员使用 dfsadmin 的 save namespace 来创建新的检查点
   4. SecondaryNameNode 在合并 edits 和 fsimage 时需要消耗的内存和 NameNode 差不多, 所以一般把 NameNode 和 SecondaryNameNode 放在不同的机器上

### 3.3滚动编辑日志

正常情况HDFS文件系统有更新操作时，就会滚动编辑日志。也可以用命令强制滚动编辑日志。

1. 滚动编辑日志（前提必须启动集群）

   ```
   [root@bigdata111 current]$ hdfs dfsadmin -rollEdits
   ```

2. 镜像文件什么时候产生

   Namenode启动时加载镜像文件和编辑日志

### 3.4namenode版本号

1. 查看namenode版本号

   在/opt/module/hadoop-2.7.2/data/tmp/dfs/name/current这个目录下查看VERSION

   ```
   clusterID=CID-1f2bf8d1-5ad2-4202-af1c-6713ab381175
   cTime=0
   storageType=NAME_NODE
   blockpoolID=BP-97847618-192.168.10.102-1493726072779
   layoutVersion=-63
   ```
   
2. namenode版本号具体解释

   1.  namespaceID在HDFS上，会有多个Namenode，所以不同Namenode的namespaceID是不同的，分别管理一组blockpoolID。
   2. clusterID集群id，全局唯一
   3. cTime属性标记了namenode存储系统的创建时间，对于刚刚格式化的存储系统，这个属性为0；但是在文件系统升级之后，该值会更新到新的时间戳。
   4. storageType属性说明该存储目录包含的是namenode的数据结构。
   5. blockpoolID：一个block pool id标识一个block pool，并且是跨集群的全局唯一。当一个新的Namespace被创建的时候(format过程的一部分)会创建并持久化一个唯一ID。在创建过程构建全局唯一的BlockPoolID比人为的配置更可靠一些。NN将BlockPoolID持久化到磁盘中，在后续的启动过程中，会再次load并使用。
   6. layoutVersion是一个负整数。通常只有HDFS增加新特性时才会更新这个版本号。

### 3.5 SecondaryNameNode目录结构

Secondary NameNode用来监控HDFS状态的辅助后台程序，每隔一段时间获取HDFS元数据的快照。

在/opt/module/hadoop-2.8.4/data/dfs/name/current这个目录中查看SecondaryNameNode目录结构。（该地址为默认地址，即：没有重新配置时所使用的地址）

```
edits_0000000000000000001-0000000000000000002
fsimage_0000000000000000002
fsimage_0000000000000000002.md5
VERSION
```

SecondaryNameNode的namesecondary/current目录和主namenode的current目录的布局相同。

好处：在主namenode发生故障时（假设没有及时备份数据），可以从SecondaryNameNode恢复数据。

- 方法一：将SecondaryNameNode中数据拷贝到namenode存储数据的目录；
- 方法二：使用-importCheckpoint选项启动namenode守护进程，从而将SecondaryNameNode中数据拷贝到namenode目录中。

1. 案例实操（一）：

   模拟namenode故障，并采用方法一，恢复namenode数据

   - kill -9 namenode进程

   - 删除namenode存储的数据（/opt/module/hadoop-2.8.4/data/dfs/name）

     ```
     rm -rf /opt/module/hadoop-2.8.4/data/dfs/name/*
     ```

   - 拷贝SecondaryNameNode中数据到原namenode存储数据目录

     ```
     cp -R /opt/module/hadoop-2.8.4/data/dfs/namesecondary/* /opt/module/hadoop-2.7.2/data/dfs/name/
     ```

   - 重新启动namenode

     ```
     sbin/hadoop-daemon.sh start namenode
     ```

2. 案例实操（二）：

   模拟namenode故障，并采用方法二，恢复namenode数据(配置一台即可)

   1. 修改hdfs-site.xml中的

      ```
      <property>
        <name>dfs.namenode.checkpoint.period</name>
        <value>120</value>
      </property>
      
      <property>
        <name>dfs.namenode.name.dir</name>
        <value>/opt/module/hadoop-2.8.4/data/dfs/name</value>
      </property>
      ```

   2. kill -9 namenode进程

   3. 删除namenode存储的数据（/opt/module/hadoop-2.7.2/data/dfs/name）

      ```
      rm -rf /opt/module/hadoop-2.7.2/data/dfs/name/*
      ```

   4. 如果SecondaryNameNode不和Namenode在一个主机节点上，需要将SecondaryNameNode存储数据的目录拷贝到Namenode存储数据的平级目录。

      ```
      [root@bigdata111 dfs]$ pwd
      /opt/module/hadoop-2.7.2/data/dfs
      [root@bigdata111 dfs]$ ls
      data  name  namesecondary
      ```

   5. 导入检查点数据（等待一会ctrl+c结束掉）

      ```
      bin/hdfs namenode -importCheckpoint
      ```

   6. 启动namenode

      ```
      sbin/hadoop-daemon.sh start namenode
      ```

   7. 如果提示文件锁了，可以删除in_use.lock

      ```
      rm -rf /opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary/in_use.lock
      ```

### 3.6 集群安全模式操作

1. 概述

   - Namenode启动时，首先将映像文件（fsimage）载入内存，并执行编辑日志（edits）中的各项操作。一旦在内存中成功建立文件系统元数据的映像，则创建一个新的fsimage文件和一个空的编辑日志。此时，namenode开始监听datanode请求。但是此刻，namenode运行在安全模式，即namenode的文件系统对于客户端来说是只读的。
   - 系统中的数据块的位置并不是由namenode维护的，而是以块列表的形式存储在datanode中。在系统的正常操作期间，namenode会在内存中保留所有块位置的映射信息。在安全模式下，各个datanode会向namenode发送最新的块列表信息，namenode了解到足够多的块位置信息之后，即可高效运行文件系统。
   - 如果满足“最小副本条件”，namenode会在30秒钟之后就退出安全模式。所谓的最小副本条件指的是在整个文件系统中99.9%的块满足最小副本级别（默认值：dfs.replication.min=1）。在启动一个刚刚格式化的HDFS集群时，因为系统中还没有任何块，所以namenode不会进入安全模式。

2. 基本语法

   集群处于安全模式，不能执行重要操作（写操作）。集群启动完成后，自动退出安全模式。

   - `bin/hdfs dfsadmin -safemode get	`	（功能描述：查看安全模式状态
   - `bin/hdfs dfsadmin -safemode enter`  （功能描述：进入安全模式状态）
   - `bin/hdfs dfsadmin -safemode leave`	（功能描述：离开安全模式状态）
   - `bin/hdfs dfsadmin -safemode wait`（功能描述：等待安全模式状态）

3. 案例

   模拟等待安全模式

   1. 先进入安全模式

      `bin/hdfs dfsadmin -safemode enter`

   2. 执行下面的脚本

      编辑一个脚本

      ```shell
      #!/bin/bash
      bin/hdfs dfsadmin -safemode wait
      bin/hdfs dfs -put ~/hello.txt /root/hello.txt
      ```

   3. 再打开一个窗口，执行

      ```
      bin/hdfs dfsadmin -safemode leave
      ```

### 3.7 Namenode多目录配置

1. namenode的本地目录可以配置成多个，且每个目录存放内容相同，增加了可靠性。

2. 具体配置如下（仅配置在namenode节点的一台即可）

   hdfs-site.xml

   ```xml
   <property>
       <name>dfs.namenode.name.dir</name>
   <value>file:///${hadoop.tmp.dir}/dfs/name1,file:///${hadoop.tmp.dir}/dfs/name2</value>
   </property>
   ```
   
3. 格式化

   - 停止集群 删除data 和 logs(集群同步)

     `  rm -rf data/* logs/*`

   - 执行`hdfs namenode -format`

   - 开启集群测试`start-dfs.sh`

### 3.8 namenode理解

1. NameNode的作用

   - NameNode在内存中保存着整个文件系统的名称、空间和文件数据块的地址映射
   - 整个HDFS可存储的文件数受限于NameNode的内存大小

2. NameNode元数据信息

   文件名，文件目录结构，文件属性（生成时间，副本数，权限）每个文件的块列表。以及列表中的块与块所在的DataNode之间的地址映射关系，在内存中加载文件系统中每个文件和每个数据块的引用关系（文件，block，datanode之间的映射信息）数据会定期保存到本地磁盘（fsimage和edits文件）

3. NameNode文件操作

   NameNode负责文件元数据的操作DataNode负责处理文件内容的读写请求，数据流不经过NameNode，会询问它和那个DataNode联系

4. NameNode副本

   文件数据块到底存放到那些DataNode上，是由NameNode决定的，NN根据全局情况作出放置副本的决定

5. NameNode心跳机制

   全权管理数据块的赋值，周期性接收心跳和块的状态报告信息（包含该DataNod上所有数据块的列表）若接收到心跳信息，NameNode认为DataNode工作正常，如果在10分钟以后还接收不到DN的心跳，那么NameNode认为DataNode已经宕机，这时候NN准备要把DN上的数据块进行重新的复制。块的状态报告包含了一个DN上所有数据块的列表，blocks report 每小时发送一次

## 4.DataNode工作机制

DataNode工作机制

![](\img\DataNode工作机制.png)

解读：

- 一个数据库块在datanode上以文件形式存储在磁盘上，包括两个文件，一个是数据文件，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳
- DataNode启动后向NameNode注册，通过后，周期性（1小时）的向namenode上报告所有的块信息
- 心跳是么3秒一次，心跳返回结果带有namenode给该datanode的命令，如复制块数据到另一台机器上，或删除某个数据块。如果超过10分钟没有收到某个datanode的心跳，则认为该节点不可用
- 集群运行中可以安全加入和退出一些机器



### 4.1DataNode的作用：

提供真实文件数据的存储服务

- DataNode以数据块的形式存储HDFS文件
- DataNode响应HDFS客户端读写请求
- DataNode周期性的向NameNode汇报心跳信息
- DataNode周期性的向NameNode汇报数据块信息
- DataNode周期性的向NameNode汇报缓存数据块信息

### 4.2数据完整性

1. 当DataNode读取block的时候，它会计算checksum
2. 如果计算后的checksum，与block创建时值不一样，说明block损坏
3. client读取其他DataNode上的block
4. datanode在其文件创建后周期性验证checksum

### 4.3掉线时限参数设置

- datanode进程死亡或者网络故障造成datanode无法与namenode通信，namenode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称作超时时长。

- HDFS默认的超时时长为10分钟+30秒。如果定义超时时间为timeout，则超时时长的计算公式为：timeout  = 2 * dfs.namenode.heartbeat.recheck-interval + 10 * dfs.heartbeat.interval。
- 而默认的dfs.namenode.heartbeat.recheck-interval 大小为5分钟，dfs.heartbeat.interval默认为3秒。
- 需要注意的是hdfs-site.xml 配置文件中的heartbeat.recheck.interval的单位为毫秒。

```xml
<property>
    <name>dfs.namenode.heartbeat.recheck-interval</name>
    <value>300000</value>
</property>
<property>
    <name> dfs.heartbeat.interval </name>
    <value>3</value>
</property>
```

### 4.4DataNode的目录结构

和namenode不同的是，datanode的存储目录是初始阶段自动创建的，不需要额外格式化

1. 在/opt/module/hadoop-2.8.4/data/tmp/dfs/data/current这个目录下查看版本号

   ```
   [root@bigdata111 current]$ cat VERSION 
   storageID=DS-1b998a1d-71a3-43d5-82dc-c0ff3294921b
   clusterID=CID-1f2bf8d1-5ad2-4202-af1c-6713ab381175
   cTime=0
   datanodeUuid=970b2daf-63b8-4e17-a514-d81741392165
   storageType=DATA_NODE
   layoutVersion=-56
   ```

2. 具体解释

   - storageID：存储ID号
   - clusterID：集群ID，全局唯一
   - cTime：该属性标记了datanode存储系统的创建时间，对于刚刚格式化的存储系统，这个属性为0，但是在文件系统升级之后，该值会更新到新的时间戳
   - datanodeUuid：datanode的唯一识别码
   - storageType：存储类型
   - layoutVersion：是一个负整数。通常只有HDFS增加新特性时才会更新这个版本号

3. 在/opt/module/hadoop-2.8.4/data/tmp/dfs/data/current/BP-97847618-192.168.10.102-1493726072779/current这个目录下查看该数据块的版本号

   ```
   [root@bigdata111 current]$ cat VERSION 
   #Mon May 08 16:30:19 CST 2017
   namespaceID=1933630176
   cTime=0
   blockpoolID=BP-97847618-192.168.10.102-1493726072779
   layoutVersion=-56
   ```

4. 具体解释

   - namespaceID：是datanode首次访问namenode的时候从namenode处获取的storageID对每个datanode来说是唯一的（但对于单个datanode中所有存储目录来说则是相同的），namenode可用这个属性来区分不同datanode。
   - cTime属性标记了datanode存储系统的创建时间，对于刚刚格式化的存储系统，这个属性为0；但是在文件系统升级之后，该值会更新到新的时间戳。
   - blockpoolID：一个block pool id标识一个block pool，并且是跨集群的全局唯一。当一个新的Namespace被创建的时候(format过程的一部分)会创建并持久化一个唯一ID。在创建过程构建全局唯一的BlockPoolID比人为的配置更可靠一些。NN将BlockPoolID持久化到磁盘中，在后续的启动过程中，会再次load并使用。
   - layoutVersion是一个负整数。通常只有HDFS增加新特性时才会更新这个版本号。

### 4.5 Datanode多目录配置

1. datanode也可以配置成多个目录，每个目录存储的数据不一样。即：数据不是副本。

2. 具体配置：（每台都需配置）

   hdfs-site.xml

   ```xml
   <property>
   <name>dfs.datanode.data.dir</name>        				<value>file:///${hadoop.tmp.dir}/dfs/data1,file:///${hadoop.tmp.dir}/dfs/data2</value>
   </property>
   ```
   
3. 格式化

   - 停止集群 删除data 和 logs(集群同步)

     `  rm -rf data/* logs/*`

   - 执行`hdfs namenode -format`

   - 开启集群测试`start-dfs.sh`



## 5.HDFS数据流

**HDFS写数据流程**

![](C:\Users\宋天\Desktop\大数据\img\HDFS写数据流程.png)

1. 客户端向NameNode请求上传文件，NameNode检查目标文件是否存在，父目录是否存在
2. namenode返回是否可以上传。
3. 客户端请求第一个block上传到哪几个datanode服务器上
4. namenode返回3个datanode节点，分别为dn1、dn2、dn3。
5. 客户端请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成
6. dn1、dn2、dn3逐级应答客户端
7. 客户端开始往dn1上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位，dn1收到一个packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答
8. 当一个block传输完成之后，客户端再次请求namenode上传第二个block的服务器。（重复执行3-7步）

**HDFS读数据流程**

![](C:\Users\宋天\Desktop\大数据\img\HDFS读数据流程.png)

1. 客户端向namenode请求下载文件，namenode通过查询元数据，找到文件块所在的datanode地址。
2. 挑选一台datanode（就近原则，然后随机）服务器，请求读取数据。
3. datanode开始传输数据给客户端（从磁盘里面读取数据放入流，以packet为单位来做校验）。
4. 客户端以packet为单位接收，先在本地缓存，然后写入目标文件。

### 5.1网络拓扑概念

在本地网络中，两个节点被称为“彼此近邻”是什么意思？在海量数据处理中，其主要限制因素是节点之间数据的传输速率——带宽很稀缺。这里的想法是将两个节点间的带宽作为距离的衡量标准。

节点距离：两个节点到达最近的共同祖先的距离总和。

例如：假设有数据中心d1机架r1中的节点n1。该节点可以表示为/d1/r1/n1。利用这种标记，这里给出四种距离描述。

Distance(/d1/r1/n1, /d1/r1/n1)=0（同一节点上的进程）

Distance(/d1/r1/n1, /d1/r1/n2)=2（同一机架上的不同节点）

Distance(/d1/r1/n1, /d1/r3/n2)=4（同一数据中心不同机架上的节点）

Distance(/d1/r1/n1, /d2/r4/n2)=6（不同数据中心的节点）

![](C:\Users\宋天\Desktop\大数据\img\机架感知.png)



### 5.2机架感知（副本节点选择）

1. 低版本Hadoop副本节点选择

   第一个副本在client所处的节点上。如果客户端在集群外，随机选一个。

   第二个副本和第一个副本位于不相同机架的随机节点上。

   第三个副本和第二个副本位于相同机架，节点随机。

   ![](C:\Users\宋天\Desktop\大数据\img\低版本机架感知.png)

2. Hadoop2.7.2副本节点选择

   ​	第一个副本在client所处的节点上。如果客户端在集群外，随机选一个。

   ​	第二个副本和第一个副本位于相同机架，随机节点。

   ​	第三个副本位于不同机架，随机节点。

   ![](C:\Users\宋天\Desktop\大数据\img\机架感知2.7.2.png)

### 5.3一致性模型

1. debug调试如下代码

   ```java
   @Test
   	public void writeFile() throws Exception{
   		// 1 创建配置信息对象
   		Configuration configuration = new Configuration();
   		fs = FileSystem.get(configuration);
   		
   		// 2 创建文件输出流
   		Path path = new Path("hdfs://bigdata111:9000/user/itstar/hello.txt");
   		FSDataOutputStream fos = fs.create(path);
   		
   		// 3 写数据
   		fos.write("hello".getBytes());
           // 4 一致性刷新
   		fos.hflush();
   		
   		fos.close();
   	}
   ```

2. 总结

   写入数据时，如果希望数据被其他client立即可见，调用如下方法

   ```java
   FsDataOutputStream. hflush ();		//清理客户端缓冲区数据，被其他client立即可见
   ```



## 6.HDFS其他功能

### 6.1集群间数据拷贝

1. scp实现两个远程主机之间的文件复制

   ```
   scp -r hello.txt root@bigdata112:/user/itstar/hello.txt		// 推 push
   scp -r root@bigdata112:/user/itstar/hello.txt  hello.txt		// 拉 pull
   scp -r root@bigdata112:/user/itstar/hello.txt root@bigdata113:/user/itstar  
   //是通过本地主机中转实现两个远程主机的文件复制；如果在两个远程主机之间ssh没有配置的情况下可以使用该方式。
   ```

2. 采用discp命令实现两个hadoop集群之间的递归数据复制

   ```
   bin/hadoop distcp hdfs://192.168.1.51:9000/LICENSE.txt hdfs://192.168.1.111:9000/HAHA
   ```

### 6.2Hadoop存档

1. 理论概述

   - 每个文件均按块存储，每个块的元数据存储在namenode的内存中，因此hadoop存储小文件会非常低效。因为大量的小文件会耗尽namenode中的大部分内存。但注意，存储小文件所需要的磁盘容量和存储这些文件原始内容所需要的磁盘空间相比也不会增多。例如，一个1MB的文件以大小为128MB的块存储，使用的是1MB的磁盘空间，而不是128MB。
   - Hadoop存档文件或HAR文件，是一个更高效的文件存档工具，它将文件存入HDFS块，在减少namenode内存使用的同时，允许对文件进行透明的访问。具体说来，Hadoop存档文件可以用作MapReduce的输入。

2. 案例实操

   1. 需要启动yarn进程

      `start-yarn.sh`

   2. 归档文件

      归档成一个叫做xxx.har的文件夹，该文件夹下有相应的数据文件。Xx.har目录是一个整体，该目录看成是一个归档文件即可。

      ```
      bin/hadoop archive -archiveName myhar.har -p /user/itstar   /user/my
      ```

   3. 查看归档

      ```
      hadoop fs -lsr /user/my/myhar.har
      hadoop fs -lsr har:///myhar.har
      ```

   4. 解归档文件
   
      取消存档：`hadoop fs -cp har:/// user/my/myhar.har /* /user/itstar`
   
      并行解压缩：`hadoop distcp har:/foo.har /001`

### 6.3 快照管理

快照相当于对目录做一个备份。并不会立即复制所有文件，而是指向同一个文件。当写入发生时，才会产生新文件。

1. 基本语法

   ```
   	（1）hdfs dfsadmin -allowSnapshot 路径   （功能描述：开启指定目录的快照功能）
   	（2）hdfs dfsadmin -disallowSnapshot 路径 （功能描述：禁用指定目录的快照功能，默认是禁用）
   	（3）hdfs dfs -createSnapshot 路径        （功能描述：对目录创建快照）
   	（4）hdfs dfs -createSnapshot 路径 名称   （功能描述：指定名称创建快照）
   	（5）hdfs dfs -renameSnapshot 路径 旧名称 新名称 （功能描述：重命名快照）
   	（6）hdfs lsSnapshottableDir         （功能描述：列出当前用户所有可快照目录）
   	（7）hdfs snapshotDiff 路径1 路径2 （功能描述：比较两个快照目录的不同之处）
   	（8）hdfs dfs -deleteSnapshot <path> <snapshotName>  （功能描述：删除快照）
   ```

2. 案例实操

   1. 开启/禁用指定目录的快照功能

      ```
      hdfs dfsadmin -allowSnapshot /user/itstar/data		
      hdfs dfsadmin -disallowSnapshot /user/itstar/data	
      ```

   2. 对目录创建快照

      ```
      hdfs dfs -createSnapshot /user/itstar/data		// 对目录创建快照
      ```

      通过web访问hdfs://bigdata111:9000/user/itstar/data/.snapshot/s…..// 快照和源文件使用相同数据块

      ```
      hdfs dfs -lsr /user/itstar/data/.snapshot/
      ```

   3. 指定名称创建快照

      ```
      hdfs dfs -createSnapshot /user/itstar/data miao170508		
      ```

   4. 重命名快照

      ```
      hdfs dfs -renameSnapshot /user/itstar/data/ miao170508 itstar170508		
      ```

   5. 列出当前用户所有可快照目录

      ```
      hdfs lsSnapshottableDir	
      ```

   6. 比较两个快照目录的不同之处

      ```
      hdfs snapshotDiff /user/itstar/data/  .  .snapshot/itstar170508	
      ```

   7. 恢复快照

      ```
      hdfs dfs -cp /user/itstar/input/.snapshot/s20170708-134303.027 /user
      ```

### 6.4 回收站

1. 默认回收站

   默认值fs.trash.interval=0，0表示禁用回收站，可以设置删除文件的存活时间。

   默认值fs.trash.checkpoint.interval=0，检查回收站的间隔时间。

   要求fs.trash.checkpoint.interval<=fs.trash.interval。

   ![](C:\Users\宋天\Desktop\大数据\img\HDFS回收站.png)

2. 启用回收站

   修改core-site.xml，配置垃圾回收时间为1分钟。

   ```xml
   <property>
       <name>fs.trash.interval</name>
       <value>1</value>
   </property>
   ```

3. 查看回收站

   回收站在集群中的；路径：/user/itstar/.Trash/….

4. 修改访问垃圾回收站用户名称

   进入垃圾回收站用户名称，默认是dr.who，修改为itstar用户

   core-site.xml

   ```xml
   <property>
     <name>hadoop.http.staticuser.user</name>
     <value>itstar</value>
   </property>
   ```

5. 通过程序删除的文件不会经过回收站，需要调用moveToTrash()才进入回收站

   ```
   Trash trash = New Trash(conf);
   
   trash.moveToTrash(path);
   ```

6. 恢复回收站数据

   ```
   hadoop fs -mv /user/itstar/.Trash/Current/user/itstar/input   /user/itstar/input
   ```

7. 清空回收站

   ```
   hdfs dfs -expunge
   ```


## 7.HDFS命令行基础操作

注：以下`hadoop fs`和`hdfs dfs`都可以使用

1. -help：输出某个命令的参数

   ```
   bin/hdfs dfs -help rm
   ```

2. -ls：显示目录信息

   ```
   hdfs dfs -ls /
   ```

3. -mkdir：在hdfs上创建目录

   ```
   hdfs dfs -mkdir -p /路径
   ```

   参数-p：递归创建

4. -moveFromLocal从本地剪切粘贴到hdfs

   ```
   hdfs dfs -moveFromLocal 本地路径 /hdfs路径
   ```

5. -appendToFile：追加一个文件到已经存在的文件末尾

   ```
   hdfs dfs -appendToFile 本地路径 /hdfs路径
   ```

6. -cat：显示文件内容

   ```
   hdfs dfs -cat /hdfs路径
   ```

7. -tail -f 监控文件

   ```
   hdfs dfs -tail -f /hdfs路径
   ```

8. -chmod、-chown：linux文件系统中的用法一样，修改文件所属权限

   ```
   hdfs dfs -chmod 777 /hdfs路径
   hdfs dfs -chown someuser:somegrp /hdfs路径
   ```

9. -cp：从hdfs的一个路径拷贝到hdfs的另一个路径

   ```
   hdfs dfs -cp /hdfs路径1 /hdfs路径2
   ```

10. -mv：在hdfs目录中移动、重命名文件

    ```
    hdfs dfs -mv /hdfs路径 /hdfs路径
    ```

11. -get：等同于copyToLocal，就是从hdfs下载文件到本地

    ```
    hdfs dfs -get /hdfs路径 ./本地路径
    ```

12. -getmerge：合并下载多个文件到linux本地，比如hdfs的目录/a下有多个文件：log1,log2...（注：是合成下载到本地）

    ```
    hdfs dfs -getmerge /a/log.* ./log.sum
    ```

    合成到不同的目录：

    ```
    hdfs dfs -getmerge /hdfs1路径 /hdfs2路径 /
    ```

13. -put：等同于copyFromLocal

    ```
    hdfs dfs -put /本地路径 /hdfs路径
    ```

14. -rm：删除文件或文件夹

    ```
    hdfs dfs -rm -r /hdfs路径
    ```

    参数-r：递归删除

15. -df：统计文件系统的可用空间信息

    ```
    hdfs dfs -df -h /hdfs路径
    ```

    示例：

    ```
    [root@bigdata111 file]# hdfs dfs -df -h /a
    Filesystem                 Size     Used  Available  Use%
    hdfs://bigdata111:9000  149.9 G  709.8 M    131.7 G    0%
    ```

16. -du：统计文件夹大小信息

    ```
    [root@bigdata111 file]# hdfs dfs -du /a
    0  /a/b
    ```

17. -count：统计一个指定目录下的文件节点数量

    ```
    hdfs dfs -count /a
    
    [root@bigdata111 file]# hdfs dfs -count /a
               2            0                  0 /a
               嵌套文件层级；  包含文件的总数
    ```

18. -setrep：设置hdfs中文件的副本数量：3是副本数，可改

    ```
    hdfs dfs -setrep 3 /hdfs路径
    ```

    注：这里设置的副本数只是记录在namenode的元数据中，是否真的会有这么多副本，还得看datanode的数量。因为目前只有3台设备，最多也就3个副本，只有节点数的增加到10台时，副本数才能达到10。



## 8.HDFS高级命令

1. HDFS文件限额配置

   概念：

   - 在多人共用HDFS的环境下，配置设置非常重要。特别是在Hadoop处理大量资料的环境，果没有配额管理，很容易把所有的空间用完造成别人无法存取。Hdfs的配额设定是针对目而不是针对账号，可以 让每个账号仅操作某一个目录，然后对目录设置配置。
   -  hdfs文件的限额配置允许我们以文件个数，或者文件大小来限制我们在某个目录下上传的件数量或者文件内容总量，以便达到我们类似百度网盘网盘等限制每个用户允许上传的最的文件的

   ```
   hdfs dfs -count -q -h /1.txt  #查看配额信息
   ```

   ```
   [root@bigdata111 file]# hdfs dfs -count -q -h /1.txt
    none         inf      none       inf      0      1     16 /1.txt
   ```

2. 数量限额

   ```
   hdfs dfs  -mkdir -p /user/root/dir    #创建hdfs文件夹
   hdfs dfsadmin -setQuota 2 dir      # 给该文件夹下面设置最多上传两个文件，发现只能
   上传一个文件,准确上传数量为(n-1)次
   hdfs dfsadmin -clrQuota /user/root/dir  # 清除文件数量
   ```

3. 空间大小限额

   在设置空间配额时，设置的空间至少是block_size * 3大小

   ```
   hdfs dfsadmin -setSpaceQuota 4k /user/root/dir   # 限制空间大小4KB
   hdfs dfs -put /root/a.txt /user/root/dir
   ```

   

   ```
   [root@bigdata111 file]# hdfs dfsadmin -setSpaceQuota 4k /a
   [root@bigdata111 file]# hdfs dfs -put /opt/software/zookeeper-3.4.10.tar.gz /a
   put: The DiskSpace quota of /a is exceeded: quota = 4096 B = 4 KB but diskspace consumed = 402653232 B = 384.00 MB
   ```

   生成任意大小文件的命令:

   ```
   dd if=/dev/zero of=1.txt  bs=1M count=2     #生成2M的文件
   ```

   示例：

   ```
   在当前目录下创建一个文件名为file的10M的空文件
   
   dd if=/dev/zero of=./file.txt bs=1M count=10    
   ```

   清除空间配额

   ```
   hdfs dfsadmin -clrSpaceQuota /a
   ```



## 9.hdfs安全模式

概念：

安全模式杀死hadoop的一种保护机制，用于保证集群中的数据块的完整性。当集群启动的时候，会首先进入安全模式。当系统处于安全模式的时候会检查数据块的完整性

假设我们设置的副本数（即参数dfs.replication）是3，那么在datanode上就应该有3个副本存在，假设只存在2个副本，那么比例就是2/3=0.666。hdfs默认的副本率0.999。我们的副本率0.666明显小于0.999，因此系统会自动的复制副本到其他dataNode，使得副本率不小于0.999。如果系统中有5个副本，超过我们设定的3个副本，那么系统也会删除多于的2个副本

**注：**在安全模式状态下，文件系统只接受读数据请求，而不接受删除、修改等变更请求。在，当
整个系统达到安全标准时，HDFS自动离开安全模式。

安全模式操作命令

```
hdfs dfsadmin  -safemode  get #查看安全模式状态
hdfs dfsadmin  -safemode enter #进入安全模式
hdfs dfsadmin  -safemode leave #离开安全模式
```



## 10.HDFS基准测试

概念：

实际生产环境当中，hadoop的环境搭建完成之后，第一件事就是进行压力测试，测试我们的集群的读取和写入速度，测试我们的网络宽带是否足够等一些基准测试

### 10.1测试写入速度

向HDFS文件系统中写入数据,10个文件,每个文件10MB,文件存放到HDFS文件系统的/benchmarks/TestDFSIO中

```
hadoop jar /opt/module/hadoop-2.8.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.8.4.jar TestDFSIO -write -nrFiles 10 -fileSize 10MB
```

注：上述路径为hadoop的安装路径下的一个测试基准使用的jar包，每个hadoop都有自己对应的jar包

完成之后查看写入速度结果

```
hdfs dfs -text /benchmarks/TestDFSIO/io_write/part-000
```

在执行写入命令的文件夹下会生成一个TestDFSIO_results.log的文件，也可以查看结果，

```
[root@bigdata111 file]# vim TestDFSIO_results.log 

----- TestDFSIO ----- : write
Date & time: Sun Oct 13 17:44:13 CST 2019 日期和时间：2019年10月13日星期日17:44:13
Number of files: 10   文件数
Total MBytes processed: 100 总大小
Throughput mb/sec: 3.2   吞吐量
Average IO rate mb/sec: 3.53 平均IO速率
IO rate std deviation: 0.91   IO速率标准偏差
Test exec time sec: 73.27    测试执行时间秒
```



### 10.2 测试读取速度

测试hdfs的读取文件性能，在HDFS文件系统中读入10个文件,每个文件10M

```
hadoop jar /opt/module/hadoop-2.8.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.8.4.jar TestDFSIO -read -nrFiles 10 -fileSize 10MB
```

注：上述路径为hadoop的安装路径下的一个测试基准使用的jar包，每个hadoop都有自己对应的jar包

查看读取果

```
hdfs dfs -text /benchmarks/TestDFSIO/io_read/part-000
```

在执行写入命令的文件夹下会生成一个TestDFSIO_results.log的文件，也可以查看结果，

**注：**如果存在该文件则会追加到文件末尾

```
----- TestDFSIO ----- : read
            Date & time: Sun Oct 13 17:54:56 CST 2019
        Number of files: 10    文件数
 Total MBytes processed: 100 处理的总兆字节数
      Throughput mb/sec: 52.69    吞吐量
 Average IO rate mb/sec: 98.76 平均IO速率
  IO rate std deviation: 94.88   IO速率标准偏差
     Test exec time sec: 55.47 测试执行时间秒
```



### 10.3清除测试数据

```
[root@bigdata111 file]# hadoop jar /opt/module/hadoop-2.8.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-2.8.4.jar TestDFSIO -clean
19/10/13 17:58:07 INFO fs.TestDFSIO: TestDFSIO.1.8
19/10/13 17:58:07 INFO fs.TestDFSIO: nrFiles = 1
19/10/13 17:58:07 INFO fs.TestDFSIO: nrBytes (MB) = 1.0
19/10/13 17:58:07 INFO fs.TestDFSIO: bufferSize = 1000000
19/10/13 17:58:07 INFO fs.TestDFSIO: baseDir = /benchmarks/TestDFSIO
19/10/13 17:58:08 INFO fs.TestDFSIO: Cleaning up test files
```

注：清除数据之后，HDFS文件系统的 benchmarks目录存在，只是其中的数据被删除了，当前目录下的TestDFSIO_results.log文件也同样会被保留