# 18_yarn

## 1. yarn介绍

yarn是hadoop集群当中的资源管理系统模块，从hadoop2.0开始引入yarn模块，yarn可为各类计算框架提供资源的管理和调度，主要用于管理集群当中的资源（主要是服务器的各种硬件资源，包括cpu、内存、磁盘、网络IO等）以及调度运行在yarn上面的各种任务

yarn核心出发点是为了分离资源管理与作业监控，实现分离的做法是拥有一个全局的资源管理（ResourceManger，RM），以及每个应用程序对应一个的应用管理器（ApplicationMaster，AM）

总结一句话就是说：yarn主要就是为了调度资源，管理任务等

其调度分为两个层级来说：

- 以及调度管理

  计算机资源管理（cpu、内存、网络IO、磁盘）

- 二级调度管理

  任务内部的计算模型管理（AppMaster的任务精细化管理）

yarn的官网文档说明：

http://hadoop.apache.org/docs/r2.7.5/hadoop-yarn/hadoop-yarn-site/YARN.ht

yarn集群的监控管理界面：

http://node01:8088/clusterjobHistoryServer

查看界面：

http://node01:19888/jobhisto

## 2.yarn的主要组件介绍与作用

yarn总体上是Master/Slave结构，主要由ResourceManage、NodeManager、ApplicationMaster和Container等几个组件构成

1. ResourceManager(RM) 

   负责处理客户端请求,对各NM上的资源进行统一管理和调度。给ApplicationMaster分配空
   闲的Container 运行并监控其运行状态。主要由两个组件构成：调度器和应用程序管理
   器

2. 调度器（Scheduler）：

   调度器根据容量、队列等限制条件，将系统中的资源分配给各个正在运行的应用程序。调度器仅根据各个应用程序的资源需求进行资源分配，而资源分配单位是Container。Shceduler不负责监控或者跟踪应用程序的状态。总之，调度器根据应用程序的资源要求，以及集群机器的资源情况，为应用程序分配封装在Container中的资源

3. 应用程序管理器（Applications Manager）：

   应用程序管理器负责管理整个系统中所有应用程序，包括应用程序提交、与调度器协商资源以启动ApplicationMaster、监控ApplicationMaster运行状态并在失败时重新启动等，跟踪分给的Container的进度、状态也是其职责

4. NodeManger（NM）：

   NodeManager是每个节点上的资源和任务管理器。他会定时地向ResourceManager汇报本节点上的资源使用情况和各个Container的运行状态；同时会接收并处理来自ApplicationMaster的Container启动/停止等请求

5. ApplicationMaster（AM）：

   用户提交的应用程序均包含一个ApplicationMaster，负责应用的监控，跟踪应用执行状态，重启失败任务等。ApplicationMaster是应用框架，他负责向ResourceManager协调资源，并且与NodeManager协同工作完成Tash的执行与监控

6. Container：

   Container是yarn中的资源抽象，它封装了某个节点上的多维度资源，如内粗怒、cpu、磁盘、网络等，当ApplicationMaster向ResourceManager申请资源时，ResourceManager为ApplicationMaster返回的资源便是用Container表示的

## 3.yarn的架构和工作流程

![](C:\Users\宋天\Desktop\大数据\大数据笔记\img\yarn\yarn.png)



### 3.1简化版工作机制：

1. 用户使用客户端向 RM 提交一个任务job，同时指定提交到哪个队列和需要多少资源。用户可以通过每个计算引擎的对应参数设置，如果没有特别指定，则使用默认设置。
2.  RM 在收到任务提交的请求后，先根据资源和队列是否满足要求选择一个 NM，通知它启动一个特殊的 container，称为 ApplicationMaster（AM），后续流程由它发起。
3.  AM 向 RM 注册后根据自己任务的需要，向 RM 申请 container，包括数量、所需资源量、所在位置等因素。
4. 如果队列有足够资源，RM 会将 container 分配给有足够剩余资源的 NM，由 AM 通知 NM 启动 container。
5.  container 启动后执行具体的任务，处理分给自己的数据。NM 除了负责启动 container，还负责监控它的资源使用状况以及是否失败退出等工作，如果 container 实际使用的内存超过申请时指定的内存，会将其杀死，保证其他 container 能正常运行。
6.  各个 container 向 AM 汇报自己的进度，都完成后，AM 向 RM 注销任务并退出，RM 通知 NM 杀死对应的 container，任务结束。

**container设置多少资源合适？**

如果 container 内存设置得过低，而实际使用的内存较多，则可能会被 YARN 在运行过程中杀死，无法正常运行。而如果 container 内部线程并发数较多而 vcore 设置的较少，则可能会被分配到一个 load 已经比较高的机器上，导致运行缓慢。所以需要预估单个 container 处理的数据量对应的内存，同时 vcore 数设置的不应该比并发线程数低。

### 3.2复杂版运行机制

![](img\yarn\yarn复杂版运行机制.png)

工作机制详解

1. Mr程序提交到客户端所在的节点。
2. Yarnrunner向Resourcemanager申请一个Application。
3. rm将该应用程序的资源路径返回给yarnrunner。
4. 该程序将运行所需资源提交到HDFS上。
5. 程序资源提交完毕后，申请运行mrAppMaster。
6. RM将用户的请求初始化成一个task。
7. 其中一个NodeManager领取到task任务。
8. 该NodeManager创建容器Container，并产生MRAppmaster。
9. Container从HDFS上拷贝资源到本地。
10. MRAppmaster向RM 申请运行maptask资源。
11. RM将运行maptask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。
12. MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动maptask，maptask对数据分区排序。
13. MrAppMaster等待所有maptask运行完毕后，向RM申请容器，运行reduce task。
14. reduce task向maptask获取相应分区的数据。
15. 程序运行完毕后，MR会向RM申请注销自己。

## 4.作业提交全过程

1. 作业提交

   第0步：client调用job.waitForCompletion方法，向整个集群提交MapReduce作业。

   第1步：client向RM申请一个作业id。

   第2步：RM给client返回该job资源的提交路径和作业id。

   第3步：client提交jar包、切片信息和配置文件到指定的资源提交路径。

   第4步：client提交完资源后，向RM申请运行MrAppMaster。

2. 作业初始化

   第5步：当RM收到client的请求后，将该job添加到容量调度器中。

   第6步：某一个空闲的NM领取到该job。

   第7步：该NM创建Container，并产生MRAppmaster。

   第8步：下载client提交的资源到本地。

3. 任务分配

   第9步：MrAppMaster向RM申请运行多个maptask任务资源。

   第10步：RM将运行maptask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。

4. 任务运行

   第11步：MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动maptask，maptask对数据分区排序。

   第12步：MrAppMaster等待所有maptask运行完毕后，向RM申请容器，运行reduce task。

   第13步：reduce task向maptask获取相应分区的数据。

   第14步：程序运行完毕后，MR会向RM申请注销自己。

5. 进度和状态更新

   YARN中的任务将其进度和状态(包括counter)返回给应用管理器, 客户端每秒(通过mapreduce.client.progressmonitor.pollinterval设置)向应用管理器请求进度更新, 展示给用户。

6. 作业完成

   除了向应用管理器请求作业进度外, 客户端每5分钟都会通过调用waitForCompletion()来检查作业是否完成。时间间隔可以通过mapreduce.client.completion.pollinterval来设置。作业完成之后, 应用管理器和container会清理工作状态。作业的信息会被作业历史服务器存储以备之后用户核查。

## 5.yarn的调度器

yarn我们都知道主要用于做资源调度，任务分配等功能的，那么在hadoop当中，究竟使用什么算法来进行任务调度就需要我们关注了，hadoop支持好几种任务的调度方式，不同的场景使用不同的任务调度器

1. FIFO Scheduler（队列调度）

   ![](img\yarn\先进先出调度器.png)

   

   把任务按提交的顺序排成一个队列，这是一个先进先出队列，在进行资源分配的时候，先给队列中最头上的任务进行分配资源，带最头上任务需求满足后再给下一个分配，以此类推。

   FIFO Scheduler是最简单也是最容易理解的调度器，也不需要任何配置，但它并不适用与共享集群。大的任务可能会占用所有集群资源，这就导致其他任务被阻塞

   **优点：**调度算法简单，JobTracker（job提交任务后发送得地方）工作负担轻。

   **缺点：**忽略了不同作业的需求差异。例如如果类似对海量数据进行统计分析的作业长期占据计算资源，那么在其后提交的交互型作业有可能迟迟得不到处理，从而影响到用户的体验。

2. Capacity Scheduler（容量调度器，apache版本默认使用的调度器）

   ![](img\yarn\容量调度器.png)

   Capacity调度器允许多个组织共享整个集群，每个组织可以获得集群的一部分计算能力。通过为每个组织分配专门的队列，然后再为每个队列分配一定的集群资源，这样整个集群就可以通过设置多个队列的方式给多个组织提供服务了。除此之外，队列内部又可以垂直划分，这样一个组织内部的多个成员就可以共享这个队列资源了，在一个队列内部，资源的调度是采用先进先出(FIFO)策略

   1. 多队列支持，每个队列采用FIFO
   2. 为了防止同一个用户的作业独占队列中的资源，该调度器会对同一个用户提交多的作业所占资源量进行限定
   3. 首先，计算每个队列中正在运行的任务数与其应该分得的计算资源之间的比值，选择一个该比值最小的队列
   4. 其次，根据作业的优先级和提交时间顺序，同时考虑用户资源量限制和内存限制对队列内任务排序
   5. 三个队列同时按照任务的先后顺序依次执行，比如，job1，job21和job31分别排在队列最前面，是最先运行，也是同时运行

   该调度**默认情况下不支持优先级**，但是可以在配置文件中开启此选项，如果支持优先级，调度算法就是带有优先级的FIFO。

   **不支持优先级抢占**，一旦一个作业开始执行，在执行完之前它的资源不会被高优先级作业所抢占。

   对队列中同一用户提交的作业能够获得的资源百分比进行了限制以使同属于一用户的作业不能出现独占资源的情况。

3. Fair Scheduler(公平调度器，CDH版本的hadoop默认使用的调度器)

   ![](img\yarn\公平调度器.png)

   Fair调度器的设计目标是为所有的应用分配公平的资源（对公平的定义可以通过参数来设置）。公平调度也可以在多个队列间工作。举个例子，假设有两个用户A和B，他们分别拥有一个队列。大概A启动一个job而B没有任务时，A会获得全部集群资源；当B启动一个job后，A的job会继续运行，不过一会儿之后两个任务会各自获得一半的集群资源。如果此时B在启动第二个job并且其他job还在运行，则他将会和B的第一个job共享B这个队列的资源，也就是B的两个job会用于四分之一的集群资源，而A的job仍然用于集群一半的资源，结果就是资源最终在两个用户之间平等的共享

   1. 支持多队列多用户，每个队列中的资源量可以配置，同一个队列中的作业公平共享队列中所有资源
   2. 比如有三个队列A，B，C.每个队列中的job按照优先级分配资源，优先级越高分配的资源越多，但是每个job都分配到资源以确保公平。在资源有限的情况下，每个job理想情况下，获得的计算资源与实际获得的计算资源存在一种差距，这个差距叫做缺额。同一个队列，job的资源缺额越大，越先获得的资源优先执行，作业是按照缺额的高低来先后执行的，而且可以看到上图有多个作业同时运行

具体设置详见：yarn-default.xml文件

```xml
<property>
    <description>The class to use as the resource scheduler.</description>
    <name>yarn.resourcemanager.scheduler.class</name>
<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```

## 6.任务的推测执行

推测执行(Speculative Execution)是指在集群环境下运行MapReduce，可能是程序Bug，负载不均或者其他的一些问题，导致在一个JOB下的多个TASK速度不一致，比如有的任务已经完成，但是有些任务可能只跑了10%，根据木桶原理，这些任务将成为整个JOB的短板，如果集群启动了推测执行，这时为了最大限度的提高短板，Hadoop会为该task启动备份任务，让speculative task与原始task同时处理一份数据，哪个先运行完，则将谁的结果作为最终结果，并且在运行完成后Kill掉另外一个任务。

1. 作业完成时间取决于最慢的任务完成时间

   一个作业由若干个Map任务和Reduce任务构成。因硬件老化、软件Bug等，某些任务可能运行非常慢。

   典型案例：系统中有99%的Map任务都完成了，只有少数几个Map老是进度很慢，完不成，怎么办？

2. 推测执行机制：

   发现拖后腿的任务，比如某个任务运行速度远慢于任务平均速度。为拖后腿任务启动一个备份任务，同时运行。谁先运行完，则采用谁的结果。

3. 执行推测任务的前提条件

   1. 每个task只能有一个备份任务；
   2. 当前job已完成的task必须不小于0.05（5%）
   3. 开启推测执行参数设置，mapred-site.xml文件中默认是打开的。

   ```xml
   <property>
     <name>mapreduce.map.speculative</name>
     <value>true</value>
     <description>If true, then multiple instances of some map tasks                may be executed in parallel.</description>
   </property>
   
   <property>
     <name>mapreduce.reduce.speculative</name>
     <value>true</value>
     <description>If true, then multiple instances of some reduce tasks 
                  may be executed in parallel.</description>
   </property>
   ```

4. 不能启用推测执行机制情况

   1. 任务间存在严重的负载倾斜；
   2. 特殊任务，比如任务向数据库中写数据。