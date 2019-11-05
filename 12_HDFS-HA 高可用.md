# 12_HDFS-HA 高可用

## 1.HA概述

相关概念：

- 所谓HA（high available），即高可用（7*24消失不中断服务）。
- 实现高可用最关键的策略是消除单点故障。HA严格来说应该分成各个组件的HA机制：
  - HAFS的HA和YARN的HA
- Hadoop2.0之前，在HDFS集群中NameNode存在单点故障（SPOF）。
- NameNode主要在以下两个方面影响HDFS集群
  - NameNode机器发生意外，如：宕机，集群将无法使用，知道管理员重启
  - NameNode机器需要升级，包括软件、硬件升级，此时集群也将无法使用
  - HDFS HA功能通过配置Avtive/Standby连那个歌nameNode实现集群中对NameNode的操作来解决上述问题
  - 如果出现故障，如：机器崩溃或机器需要升级维护，此时可通过此种方式将NameNode很快的切换到另一台机器

### 1.1HDFS-HA工作机制

1. 通过双namenode消除单点故障

### 1.2 HDFS-HA 工作要点

1. 元数据管理方式需要改变

   - 内存中各自保存一份元数据
   - Edits日志只有Active状态的namenode节点可以做写操作
   - 两个namenode都可以读取edits
   - 共享的edits放在一个共享存储中管理（qjournal和NFS两个主流实现）；

2. 需要一个状态管理功能模块

   实现了一个zkfailover，常驻在每一个namdenode所在节点，每一个zkfailover负责监控自己所在的namenode节点，利用zk进行 状态表示，当需要进行状态切换时，由zkfailover来负责切换，切换时需要方式brain split(脑裂)现象的发生

3. 必须保证两个namenode之间能够ssh无密码登录

4. 隔离（Fence），即同一时刻仅仅只有一个NameNode对外提供服务

### 1.3 HDFS-HA 自动故障转移工作机制

概念：自动故障转移为HDFS部署增加了两个新组件：zookeeper和ZKFailoverController(ZKFC)进程。zookeeper是维护少量协调数据，通知客户端这些数据的改变和监视客户端故障的高可用服务。HA的自动故障转移依赖于Zookeeper的以下功能

1. 故障检测

   集群中的每一个NameNode在zookeeper中维护了一个持久会话，如果机器奔溃，zookeeper中的会话将终止，zookeeper通知另一个nameNode需要触发故障转移

2. 现役NameNode选择

   zookeeper提供了一个简单的机制用于唯一的选择一个节点为active状态。如果目前现役NameNode崩溃，另一个节点可能从zookeeper获得特殊的排外锁以表明它应该成为现役的NameNode

ZKFC是自动故障转移中的另一个新组件，是zookeeper的客户端，也监视和管理NameNode的状态。每个运行NameNode的主机也运行一个ZKFC的进程，ZKFC负责：

1. 健康监测

   ZKFC使用一个健康检查命令定期地ping与之在相同主机的NameNode，只要该NameNode及时地回复健康状态，ZKFC认为该节点是健康的。如果该节点崩溃，冻结或进入不健康的状态，健康监测器标识该节点为非健康的

2. Zookeeper会话管理

   当本地NameNode是健康的，zkfc保持在一个zookeeper中打开的会话。如果本地NameNode处于active状态，zkfc也保持一个特殊的znoe锁，该锁使用了zookeeper对短暂节点的支持，如果会话终止，锁节点将自动删除

3. 基于Zookeeper的选择

   如果本地NameNode是健康的，且ZKFC发现没有其他的节点当前持有znode锁，它将自己获取该锁。如果成功，则它已经赢得了选择，并负责运行故障转移进程以使它的本地NameNode为active



![](C:\Users\宋天\Desktop\大数据\img\HDFS-HA故障自动转移机制.png)

## 2.搭建

**说明：**HA较为耗费资源，所以本次安装在特定环境下，目的只是学习HA高可用搭建方法其原理

**注：**本次配置基于已经搭建完成hadoop完全分布式集群及zookeeper的情况下配置

**集群规划：**

```
bigdata111  			bigdata112   			bigdata113		
NameNode				NameNode
JournalNode				JournalNode				JournalNode		
DataNode				DataNode				DataNode		
ZK						ZK						ZK
ResourceManager
NodeManager				NodeManager				NodeManager		
```

1. 在目录`/opt`下创建HA文件夹，并将`/opt/moudle/hadoop-2.8.4`下所有文件复制一份至HA文件夹

   - 创建文件夹HA

     ```
     mkdir HA
     ```

   - 复制文件至HA目录

     ```
     cp -r /opt/module/hadoop-2.8.4/ /opt/HA/
     ```

2. 打开目录`/opt/HA/hadoop-2.8.4/etc/hadoop`下的hadoop-env.sh文件，配置java_home

   ```
   export JAVA_HOME=/opt/module/jdk1.8.0_144
   ```

   注：注意自己的JDK版本

3. 打开第二步目录下的`hdfs-site.xml`文件，将我们之前写入`<configuration></configuration>`标签内的所有内容删除，并添加以下内容

   ```xml
   <configuration>
   	<!-- 完全分布式集群名称 -->
   	<property>
   		<name>dfs.nameservices</name>
   		<value>mycluster</value>
   	</property>
   
   	<!-- 集群中NameNode节点都有哪些 -->
   	<property>
   		<name>dfs.ha.namenodes.mycluster</name>
   		<value>nn1,nn2</value>
   	</property>
   
   	<!-- nn1的RPC通信地址 -->
   	<property>
   		<name>dfs.namenode.rpc-address.mycluster.nn1</name>
   		<value>bigdata111:9000</value>
   	</property>
   
   	<!-- nn2的RPC通信地址 -->
   	<property>
   		<name>dfs.namenode.rpc-address.mycluster.nn2</name>
   		<value>bigdata112:9000</value>
   	</property>
   
   	<!-- nn1的http通信地址 -->
   	<property>
   		<name>dfs.namenode.http-address.mycluster.nn1</name>
   		<value>bigdata111:50070</value>
   	</property>
   
   	<!-- nn2的http通信地址 -->
   	<property>
   		<name>dfs.namenode.http-address.mycluster.nn2</name>
   		<value>bigdata112:50070</value>
   	</property>
   
   	<!-- 指定NameNode元数据在JournalNode上的存放位置 -->
   	<property>
   		<name>dfs.namenode.shared.edits.dir</name>
   	 <value>qjournal://bigdata111:8485;bigdata112:8485;bigdata113:8485/mycluster</value>
   	</property>
   
   	<!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 -->
   	<property>
   		<name>dfs.ha.fencing.methods</name>
   		<value>sshfence</value>
   	</property>
   
   	<!-- 使用隔离机制时需要ssh无秘钥登录-->
   	<property>
   		<name>dfs.ha.fencing.ssh.private-key-files</name>
   		<value>/root/.ssh/id_rsa</value>
   	</property>
   
   	<!-- 声明journalnode服务器存储目录-->
   	<property>
   		<name>dfs.journalnode.edits.dir</name>
   		<value>/opt/HA/hadoop-2.8.4/data/jn</value>
   	</property>
   
   	<!-- 关闭权限检查-->
   	<property>
   		<name>dfs.permissions.enable</name>
   		<value>false</value>
   	</property>
   
   	<!-- 访问代理类：client，mycluster，active配置失败自动切换实现方式-->
   	<property>
     		<name>dfs.client.failover.proxy.provider.mycluster</name>
   	<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
   	</property>
   </configuration>
   ```

   注：以上内容根据自己主机情况进行配置

4. 配置`core-site.xml`文件，删除原本`<configuration></configuration>`标签的内容，并添加入以下内容

   ```xml
   <configuration>
   <!-- 把两个NameNode）的地址组装成一个集群mycluster -->
   		<property>
   			<name>fs.defaultFS</name>
           	<value>hdfs://mycluster</value>
   		</property>
   
   		<!-- 指定hadoop运行时产生文件的存储目录 -->
   		<property>
   			<name>hadoop.tmp.dir</name>
   			<value>/opt/HA/hadoop-2.8.4/data</value>
   		</property>
   </configuration>
   ```

   注：根据自己主机进行配置

5. 使用命令分别复制HA目录到其他节点

   ```
   scp -r /opt/HA root@bigdata111:/opt
   ```

## 3.启动HDFS-HA集群

注：切记在HA目录下的hadoop里面使用以下命令

1. 在各个JournalNode节点上输入以下命令启动JournalNode服务

   ```
   sbin/hadoop-daemon.sh start journalnode
   ```

2. 在[nn1]上，对其进行格式化，并启动

   ```
   bin/hdfs namenode -format
   ```

   ```
   sbin/hadoop-daemon.sh start namenode
   ```

   查看一下进程

   ```
   [root@bigdata111 hadoop-2.8.4]# jps
   4224 NameNode
   3784 JournalNode
   4301 Jps
   ```

3. 在[nn2]上，同步nn1的元数据信息

   ```
   bin/hdfs namenode -bootstrapStandby
   ```

4. 启动[nn2]

   ```
   sbin/hadoop-daemon.sh start namenode
   ```

5. 查看web页面（暂未配置自动故障转移，所以都为standby）

   ![](C:\Users\宋天\Desktop\大数据\img\bigdata111节点HA状态.PNG)

   ![](C:\Users\宋天\Desktop\大数据\img\bigdata222节点HA状态.PNG)

6. 在[nn1]上，启动所有的datanode

   ```
   sbin/hadoop-daemons.sh start datanode
   ```

7. 查看是否Active

   ```
   bin/hdfs haadmin -getServiceState nn1
   ```

   测试状态：

   ```
   [root@bigdata111 hadoop-2.8.4]# bin/hdfs haadmin -getServiceState nn1
   standby
   ```

   

8. 将[nn1]切换为Active

   - 切换为Active:

     ```
     bin/hdfs haadmin -transitionToActive nn1
     ```

   - 切换为Standby

     ```
     bin/hdfs haadmin -transitionToStandby nn1
     ```

## 4.配置HDFS-HA自动转移

1. 配置`hdfs-site.xm`文件(集群同时配置)

   添加如下内容

   ```xml
   <property>
   	<name>dfs.ha.automatic-failover.enabled</name>
   	<value>true</value>
   </property>
   ```

2. 配置core-site.xml文件(集群同时配置)

   添加如下内容

   ```xml
   <property>
   	<name>ha.zookeeper.quorum</name>
   	<value>bigdata111:2181,bigdata222:2181,bigdata333:2181</value>
   </property>
   ```

3. 启动

   1. 关闭所有HDFS服务：

      ```
      sbin/stop-dfs.sh
      ```

   2. 启动zookeeper集群（集群同时启动）

      ```
      zkServer.sh start
      ```

      查看状态

      ```‘
      zkServer.sh status
      ```

   3. 初始化HA在zookeeper中的状态

      ```
      bin/hdfs zkfc -formatZK
      ```

   4. 启动HDFS服务

      ```
      sbin/start-dfs.sh
      ```

      查看进程

      ```
      节点1
      [root@bigdata111 hadoop-2.8.4]# jps
      5312 Jps
      5105 QuorumPeerMain
      4662 DataNode
      4553 NameNode
      4841 JournalNode
      5214 DFSZKFailoverController
      节点2
      [root@bigdata222 hadoop-2.8.4]# jps
      3665 Jps
      3270 JournalNode
      3606 DFSZKFailoverController
      3191 DataNode
      3116 NameNode
      3469 QuorumPeerMain
      节点3
      [root@bigdata333 hadoop-2.8.4]# jps
      2384 QuorumPeerMain
      2581 Jps
      2283 JournalNode
      2204 DataNode
      ```

      启动zookeeper客户端查看，可以发现多出一个 `hadoop-ha`进程

      ```
      进入zookeeper客户端
      zkCli.sh
      查看
      [zk: localhost:2181(CONNECTED) 0] ls /
      [zookeeper, hadoop-ha]
      
      ```

      

   5. 在各个NameNode节点上启动DFSZK Failover Controller，先在哪台机器启动，哪个机器的NameNode就是Active NameNode

      ```
      sbin/hadoop-daemon.sh start zkfc
      ```

      注：在启动HDFS服务的时候已经启动了zkfc进程，所以无需再启动

   6. 验证

      1. 将Active NameNode进程kill

         ```
         kill -9 namenode的进程id
         ```

      2. 将Active NameNode机器断开网络

         ```
         service network stop
         ```

   注意：

   本机实验kill - 9 namenode进程靠测试失败

   原因：缺少ssh相关配置；导致防脑裂机制的流程有误因此standby没有被拉起来

   在hdfs-site.xml文件中增加以下内容，测试成功

   ```xml
   <property>
   <name>dfs.ha.fencing.methods</name>
   <value>
   	sshfence
   	shell(/bin/true)
   	</value>
   </property>
   ```


## 5.yarn-HA配置

1. YARN-HA 工作机制

   ![](C:\Users\宋天\Desktop\大数据\img\yarn-ha工作机制.png)

2. 规划集群

   ```
   bigdata111  			bigdata112   			bigdata113		
   NameNode				NameNode
   JournalNode				JournalNode				JournalNode		
   DataNode				DataNode				DataNode		
   ZK						ZK						ZK
   ResourceManager			ResourceManager
   NodeManager				NodeManager				NodeManager	
   ```

3. 具体配置

   1. yarn-site.sh

      ```xml
      <configuration>
      
      <!-- Site specific YARN configuration properties -->
      
      
          <property>
      	<name>yarn.nodemanager.aux-services</name>
              <value>mapreduce_shuffle</value>
          </property>
      
          <!--启用resourcemanager ha-->
          <property>
              <name>yarn.resourcemanager.ha.enabled</name>
              <value>true</value>
          </property>
       
          <!--声明两台resourcemanager的地址-->
          <property>
              <name>yarn.resourcemanager.cluster-id</name>
              <value>cluster-yarn1</value>
          </property>
      
          <property>
              <name>yarn.resourcemanager.ha.rm-ids</name>
              <value>rm1,rm2</value>
          </property>
      
          <property>
              <name>yarn.resourcemanager.hostname.rm1</name>
              <value>bigdata111</value>
          </property>
      
          <property>
              <name>yarn.resourcemanager.hostname.rm2</name>
              <value>bigdata222</value>
          </property>
       
          <!--指定zookeeper集群的地址--> 
          <property>
              <name>yarn.resourcemanager.zk-address</name>
              <value>bigdata111:2181,bigdata222:2181,bigdata333:2181</value>
          </property>
      
          <!--启用自动恢复--> 
          <property>
              <name>yarn.resourcemanager.recovery.enabled</name>
              <value>true</value>
          </property>
       
          <!--指定resourcemanager的状态信息存储在zookeeper集群--> 
          <property>
              <name>yarn.resourcemanager.store.class</name>
      	<value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
          </property>
      
      </configuration>
      ```

      注：根据自己主机配置进行操作

   2. 同步至其他节点

4. 启动hdfs **(上面已经做过无需重复执行)**

   1. 在各个JournalNode节点上，输入以下命令启动journalnode服务

      ```
      sbin/hadoop-daemon.sh start journalnode
      ```

   2. 在[nn1]上，对其进行格式化，并启动：

      ```
      bin/hdfs namenode -format
      sbin/hadoop-daemon.sh start namenode
      ```

   3. 在[nn2]上，同步nn1的元数据信息：

      ```
      bin/hdfs namenode -bootstrapStandby
      ```

   4. 启动[nn2]：

      ```
      sbin/hadoop-daemon.sh start namenode
      ```

   5. 启动所有datanode

      ```
      sbin/hadoop-daemons.sh start datanode
      ```

   6. 将[nn1]切换为Active

      ```
      bin/hdfs haadmin -transitionToActive nn1
      ```

5. 在bigdata111中执行：

   ```
   sbin/start-yarn.sh
   ```

6. 在bigdata222中执行：

   ```
   sbin/yarn-daemon.sh start resourcemanager
   ```

7. 在各个节点上启动zookeeper

   ```
   zkServer.sh start
   ```

8. 在各个节点上启动JournalNode

   ```
   sbin/hadoop-daemon.sh start journalnode
   ```

9. 在节点1查看服务状态

   ```
   bin/yarn rmadmin -getServiceState rm1
   bin/yarn rmadmin -getServiceState rm2
   ```

   测试情况

   ```
   [root@bigdata111 hadoop-2.8.4]# bin/yarn rmadmin -getServiceState rm1
   active
   [root@bigdata111 hadoop-2.8.4]# bin/yarn rmadmin -getServiceState rm2
   standby
   ```

10. 三节点总进程查看

    1. 节点1

       ```
       [root@bigdata111 hadoop-2.8.4]# jps
       4144 Jps
       3603 ResourceManager
       3716 NodeManager
       3804 QuorumPeerMain
       4093 JournalNode
       ```

    2. 节点2

       ```
       [root@bigdata222 hadoop-2.8.4]# jps
       3508 Jps
       3333 ResourceManager
       3386 QuorumPeerMain
       3179 NodeManager
       3454 JournalNode
       ```

    3. 节点3

       ```
       [root@bigdata333 hadoop-2.8.4]# jps
       2482 NodeManager
       2723 Jps
       2619 QuorumPeerMain
       2669 JournalNode
       ```

       

11. 打开页面查看状况

    ![](C:\Users\宋天\Desktop\大数据\img\yarn-ha节点1.png)

    注：打开节点2界面的时候，因为节点1是active状态，所以会自动跳转至节点1的页面

12. 测试

    可以使用`kill -9  ResourceManager的进程号`，杀死active节点的ResourceManager进程，然后使用`bin/yarn rmadmin -getServiceState rm1`命令查看节点2是否变为active