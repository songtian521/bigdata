# 08_Zookeeper安装和基本操作

## 1. 安装JDK

前面已经安装，步骤不再重复

## 2.上传zookeeper至software目录

## 3.修改tar包权限

```
[root@bigData111 software]# chmod u+x zookeeper-3.4.10.tar.gz
```

## 4.解压到指定目录

```
 [root@bigData111 software]# tar -zxvf zookeeper-3.4.10.tar.gz -C /opt/module/
```

## 5.配置环境变量

```
[root@bigData111 software]# vi /etc/profile
```

添加如下内容：

```
export ZOOKEEPER_HOME=/opt/module/zookeeper-3.4.10
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```

## 6.配置修改

1. 将/opt/module/zookeeper-3.4.10/conf这个路径下的zoo_sample.cfg修改为zoo.cfg；

2. 进入zoo.cfg文件：`vim zoo.cfg`

   修改dataDir路径为

   ```
   dataDir=/opt/module/zookeeper-3.4.10/zkData
   ```

3. 在/opt/module/zookeeper-3.4.10/这个目录上创建zkData文件夹

   ```
   mkdir zkData
   ```

## 7.操作zookeeper

1. 启动zookeeper

   ```
   bin/zkServer.sh start
   ```

2. 查看进程是否启动

   ```
   [root@bigData111 zookeeper-3.4.10]jps
   4020 Jps
   4001 QuorumPeerMain
   ```

3. 查看状态：

   ```
   [root@bigdata111 zookeeper-3.4.10]# bin/zkServer.sh status
   JMX enabled by default
   Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
   Mode: follower
   [root@bigdata222 zookeeper-3.4.10]# bin/zkServer.sh status
   JMX enabled by default
   Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
   Mode: leader
   [root@bigdata333 zookeeper-3.4.5]# bin/zkServer.sh status
   JMX enabled by default
   Using config: /opt/module/zookeeper-3.4.10/bin/../conf/zoo.cfg
   Mode: follower
   ```

4. 启动客户端：

   ```
   [root@bigData111 zookeeper-3.4.10]# bin/zkCli.sh
   ```

5. 退出客户端：

   ```
   [zk: localhost:2181(CONNECTED) 0] quit
   ```

6. 停止zookeeper

   ```
   [root@bigData111 zookeeper-3.4.10]# bin/zkServer.sh stop
   ```


## 8.配置参数解读

解读zoo.cfg文件中参数含义

1. **tickTime=2000：**通信心跳数，Zookeeper服务器心跳时间，单位毫秒

   Zookeeper使用的基本时间，服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个tickTime时间就会发送一个心跳，时间单位为毫秒。

   它用于心跳机制，并且设置最小的session超时时间为两倍心跳时间。(session的最小超时时间是2*tickTime)

2. **initLimit=10：**Leader和Follower初始通信时限

   集群中的follower跟随者服务器与leader领导者服务器之间初始连接时能容忍的最多心跳数（tickTime的数量），用它来限定集群中的Zookeeper服务器连接到Leader的时限。

   投票选举新leader的初始化时间

   Follower在启动过程中，会从Leader同步所有最新数据，然后确定自己能够对外服务的起始状态。

   Leader允许Follower在initLimit时间内完成这个工作。

3. **syncLimit=5：**Leader和Follower同步通信时限

   集群中Leader与Follower之间的最大响应时间单位，假如响应超过syncLimit * tickTime，Leader认为Follwer死掉，从服务器列表中删除Follwer。

   在运行过程中，Leader负责与ZK集群中所有机器进行通信，例如通过一些心跳检测机制，来检测机器的存活状态。

   如果L发出心跳包在syncLimit之后，还没有从F那收到响应，那么就认为这个F已经不在线了。

4. dataDir：数据文件目录+数据持久化路径

   保存内存数据库快照信息的位置，如果没有其他说明，更新的事务日志也保存到数据库。

5. **clientPort=2181：**客户端连接端口

   监听客户端连接的端口

## 9.分布式安装部署

### 9.1集群规划

在bigdata111、bigdata222和bigdata3三33个节点上部署Zookeeper。

### 9.2 安装

重复上面2-6的操作（每台机器，在上面的基础上新增如下操作）

- 配置zoo.cfg文件

  新增内容：

  ```
  server.1=bigdata111:2888:3888
  server.2=bigdata222:2888:3888
  server.3=bigdata333:2888:3888
  ```

  注：更改为自己的主机名即可，其他可以不修改

- 配置参数解读

  - Server.A=B:C:D。
    - A是一个数字，代表这个是第几号服务器
    - B是这个服务器的IP地址
    - C是这个服务器与集群中的leader服务器交换信息的接口
    - D是万一集群的leader服务挂了，需要一个端口来重新选举，选出一个Leader，而这个端口就是用来执行选举是服务器相互通信的端口
  - 集群模式下配置一个文件myid，这个文件在dataDir目录下，这个文件里面有一个数据就是A的值，Zookeeper启动时读取此文件，拿到里面的数据与zoo.cfg里面的配置信息比较从而判断到底是哪个server。

- 在/opt/module/zookeeper-3.4.10/zkData目录下创建一个myid的文件

  - 编辑myid文件
  - 在文件中添加与server对应的编号：如2（看上面zoo.cfg配置文件中server后面的那个1、2、3，如第一台主机对应1，那么就在myid文件中添加数字1，然后保存即可）

## 10.客户端命令行操作

| 命令基本语法     | 功能描述                                                   |
| ---------------- | ---------------------------------------------------------- |
| help             | 显示所有操作命令                                           |
| ls path [watch]  | 使用 ls 命令来查看当前znode中所包含的内容                  |
| ls2 path [watch] | 查看当前节点数据并能看到更新次数等数据                     |
| create           | 普通创建(永久节点)-s  含有序列-e  临时（重启或者超时消失） |
| get path [watch] | 获得节点的值                                               |
| set              | 设置节点的具体值                                           |
| stat             | 查看节点状态                                               |
| delete           | 删除节点                                                   |
| rmr              | 递归删除节点                                               |

1. 启动客户端

   ```
   bin/zkCli.sh
   ```

2. 显示所有操作命令

   ```
   [zk: localhost:2181(CONNECTED) 0] help
   ZooKeeper -server host:port cmd args
   	stat path [watch]
   	set path data [version]
   	ls path [watch]
   	delquota [-n|-b] path
   	ls2 path [watch]
   	setAcl path acl
   	setquota -n|-b val path
   	history 
   	redo cmdno
   	printwatches on|off
   	delete path [version]
   	sync path
   	listquota path
   	rmr path
   	get path [watch]
   	create [-s] [-e] path data acl
   	addauth scheme auth
   	quit 
   	getAcl path
   	close 
   	connect host:port
   ```

3. 查看当前znode中所包含的内容

   ```
   [zk: localhost:2181(CONNECTED) 1] ls/
   ZooKeeper -server host:port cmd args
   	stat path [watch]
   	set path data [version]
   	ls path [watch]
   	delquota [-n|-b] path
   	ls2 path [watch]
   	setAcl path acl
   	setquota -n|-b val path
   	history 
   	redo cmdno
   	printwatches on|off
   	delete path [version]
   	sync path
   	listquota path
   	rmr path
   	get path [watch]
   	create [-s] [-e] path data acl
   	addauth scheme auth
   	quit 
   	getAcl path
   	close 
   	connect host:port
   ```

4. 查看当前节点数据并能看到更新次数等数据

   ```
   [zk: localhost:2181(CONNECTED) 1] ls2 /
   [zookeeper]
   cZxid = 0x0
   ctime = Thu Jan 01 08:00:00 CST 1970
   mZxid = 0x0
   mtime = Thu Jan 01 08:00:00 CST 1970
   pZxid = 0x0
   cversion = -1
   dataVersion = 0
   aclVersion = 0
   ephemeralOwner = 0x0
   dataLength = 0
   numChildren = 1
   ```

5. 创建普通节点

   ```
   [zk: localhost:2181(CONNECTED) 2] create /app1 "hello app1"
   Created /app1
   ```

   ```
   [zk: localhost:2181(CONNECTED) 2] create /app1/server101 "192.168.64.129"
   Created /app1/server101
   ```

6. 获得节点的值

   ```
   [zk: localhost:2181(CONNECTED) 3] get /app1
   hello app1
   cZxid = 0x100000004
   ctime = Fri Oct 04 19:41:14 CST 2019
   mZxid = 0x100000004
   mtime = Fri Oct 04 19:41:14 CST 2019
   pZxid = 0x100000004
   cversion = 0
   dataVersion = 0
   aclVersion = 0
   ephemeralOwner = 0x0
   dataLength = 10
   numChildren = 0
   ```

   ```
   [zk: localhost:2181(CONNECTED) 7] get /app1/server101
   192.168.64.129
   cZxid = 0x100000005
   ctime = Fri Oct 04 19:44:03 CST 2019
   mZxid = 0x100000005
   mtime = Fri Oct 04 19:44:03 CST 2019
   pZxid = 0x100000005
   cversion = 0
   dataVersion = 0
   aclVersion = 0
   ephemeralOwner = 0x0
   dataLength = 14
   numChildren = 0
   ```

7. 创建短暂节点

   ```
   [zk: localhost:2181(CONNECTED) 8] create -e /app-emphemeral 8888
   Created /app-emphemeral
   ```

   1. 在当前客户端是能够看到的

      ```
      [zk: localhost:2181(CONNECTED) 9] ls /
      [zookeeper, app1, app-emphemeral]
      ```

   2. 退出客户端然后再重启客户端，

      ```
      [zk: localhost:2181(CONNECTED) 10] quit
      [root@bigdata111 zkData]# zkCli.sh 
      ```

   3. 再次查看根目录下短暂节点已经删除

      ```
      [zk: localhost:2181(CONNECTED) 0] ls /
      [zookeeper, app1]
      ```

8. 创建带序号的节点

   1. 先创建一个普通的根节点 app2

      ```
      [zk: localhost:2181(CONNECTED) 1] create /app2 "app2"
      Created /app2
      ```

   2. 创建带序号的节点

      ```
      [zk: localhost:2181(CONNECTED) 2] create -s /app2/aa 888
      Created /app2/aa0000000000
      [zk: localhost:2181(CONNECTED) 3] create -s /app2/bb 888
      Created /app2/bb0000000001
      [zk: localhost:2181(CONNECTED) 5] create -s /app2/cc 888
      Created /app2/cc0000000002
      ```

      如果原节点下有1个节点，则再排序时从1开始，以此类推。

      ```
      [zk: localhost:2181(CONNECTED) 16] create -s /app1/aa 888
      Created /app1/aa0000000001
      ```

9. 修改节点数据值

   ```
   [zk: localhost:2181(CONNECTED) 6] set /app1 999
   cZxid = 0x100000004
   ctime = Fri Oct 04 19:41:14 CST 2019
   mZxid = 0x10000000e
   mtime = Fri Oct 04 19:51:10 CST 2019
   pZxid = 0x100000005
   cversion = 1
   dataVersion = 1
   aclVersion = 0
   ephemeralOwner = 0x0
   dataLength = 3
   numChildren = 1
   ```

10. 节点的值变化监听

    1. 在第一台主机监听/app1 节点数据变化

       ```
       [zk: localhost:2181(CONNECTED) 7] get /app1 watch
       999
       cZxid = 0x100000004
       ctime = Fri Oct 04 19:41:14 CST 2019
       mZxid = 0x10000000e
       mtime = Fri Oct 04 19:51:10 CST 2019
       pZxid = 0x100000005
       cversion = 1
       dataVersion = 1
       aclVersion = 0
       ephemeralOwner = 0x0
       dataLength = 3
       numChildren = 1
       ```

    2. 在其他主机上修改/app1节点的数据

       ```
       [zk: localhost:2181(CONNECTED) 0] set /app1 777
       cZxid = 0x100000004
       ctime = Fri Oct 04 19:41:14 CST 2019
       mZxid = 0x100000010
       mtime = Fri Oct 04 19:52:43 CST 2019
       pZxid = 0x100000005
       cversion = 1
       dataVersion = 2
       aclVersion = 0
       ephemeralOwner = 0x0
       dataLength = 3
       numChildren = 1
       ```

    3. 观察第一台主机收到的数据变化监听

       ```
       [zk: localhost:2181(CONNECTED) 8] 
       WATCHER::
       
       WatchedEvent state:SyncConnected type:NodeDataChanged path:/app1
       ```

11. 节点的子节点变化监听（路径变化）

    1. 在第一台主机上监听 /app1节点的子节点变化

       ```
       [zk: localhost:2181(CONNECTED) 8] ls /app1 watch
       [server101]
       ```

    2. 在其他主机/app1节点上创建子节点

       ```
       [zk: localhost:2181(CONNECTED) 2] create /app1/cc 444
       Created /app1/cc
       ```

    3. 观察第一台主机收到得到子节点变化的监听

       ```
       WATCHER::
       WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/app1
       ```

12. 删除节点

    ```
    [zk: localhost:2181(CONNECTED) 11] delete /app1/bb
    
    WATCHER::
    
    WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/app1
    
    [zk: localhost:2181(CONNECTED) 13] ls /app1
    [cc, server101]
    ```

13. 递归删除节点

    ```
    [zk: localhost:2181(CONNECTED) 14] rmr /app2
    ```

14. 查看节点状态

    ```
    [zk: localhost:2181(CONNECTED) 17] stat /app1
    cZxid = 0x100000004
    ctime = Fri Oct 04 19:41:14 CST 2019
    mZxid = 0x100000010
    mtime = Fri Oct 04 19:52:43 CST 2019
    pZxid = 0x100000014
    cversion = 4
    dataVersion = 2
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 3
    numChildren = 2
    ```

    