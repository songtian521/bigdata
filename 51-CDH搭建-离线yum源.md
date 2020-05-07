# 51-CDH搭建-离线yum源

# 1.初始环境准备

1. 搭建三台基于centos6虚拟机

   注：三台服务器的硬盘可以设置的大一些，主节点的内存高一些

2. 设置网络

   vim /etc/sysconfig/network-scripts/ifcfg-eth0 

   ```shell
   IPADDR=192.168.64.140
   GATEWAY=192.168.64.2
   NETMASK =255.255.255.0
   DNS1=8.8.8.8
   DNS2=8.8.4.4
   BOOTPROTO=static
   ONBOOT=yes
   ```

   ```shell
   service network restart # 重启网卡
   ```

3. 关闭防火墙

   ```shell
   service iptables stop # 关闭防火墙
   service iptables status # 验证是否关闭
   chkconfig iptables off # 关闭防火墙自动运行
   ```

4. 修改主机名

   vim /etc/sysconfig/network

   ```shell
   HOSTNAME=cdh01
   ```

5. 设置主机名映射

   vim /etc/hosts

   ```shell
   192.168.64.130 cdh01
   192.168.64.131 cdh02
   192.168.64.132 cdh03
   ```

6. 关闭selinux安全模块

   vim /etc/selinux/config

   ```shell
   SELINUX=disabled
   ```

7. 设置ssh免密登录

   ```shell
   ssh-keygen -t rsa  # 生成密钥文件，一路回车就行了
   
   # 分别发送给另外两台节点
   ssh-copy-id cdh01
   ssh-copy-id cdh02
   ssh-copy-id cdh03
   ```

8. 配置时间同步服务

   ```shell
   yum install -y ntp # yum安装ntp
   service ntpd start # 启动ntp服务
   chkconfig ntpd on  # 设置开机自启
   ntpstat # 查看ntp同步状态
   ```

9. 安装JDK

   ```shell
   rpm -qa | grep java # 查看是否有留存的JDK
   rpm -e --nodeps 包名 # 如果有, 则使用该命令卸载
   
   #上传JDK，并解压至/usr/java目录中，该目录需要自行创建
   tar -zxvf jdk-8u211-linux-x64.tar.gz
   mv jdk1.8.0_211 /usr/java/
   
   #修改 /etc/profile 配置环境变量
   export JAVA_HOME=/usr/java/jdk1.8.0_192
   export PATH=$PATH:$JAVA_HOME/bin
   ```

10. 重启节点

    ```
    shutdown -r now
    ```

注：以上操作基于centos6进行操作，并且每台节点都需要进行配置

另外说明一点：每台节点我都是单独配置的而不是采用克隆后修改相关参数，原因在于克隆修改参数后无法联网

# 2.创建本地Yum仓库

创建本地 `Yum` 仓库的目的是因为从远端的 `Yum` 仓库下载东西的速度实在是太渣, 然而 `CDH` 的所有组件几乎都要从 `Yum` 安装, 所以搭建一个本地仓库会加快下载速度

1. 下载 `CDH` 的安装包需要使用 `CDH` 的一个工具, 要安装 `CDH` 的这个工具就要先导入 `CDH` 的 `Yum` 源

   ```shell
   wget https://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/cloudera-cdh5.repo
   mv cloudera-cdh5.repo /etc/yum.repos.d/ 
   ```

2. 安装 `CDH` 安装包同步工具

   ```shell
   yum install -y yum-utils createrepo
   ```

3. 同步 `CDH` 的安装包

   ```shell
   reposync -r cloudera-cdh5
   ```

   注：该步骤执需要下载很多包，速度太慢，所以我们这里将准备好的zip包上传至任意目录，并解压

4. 安装 `Http` 服务器软件

   ```shell
   yum install -y httpd
   service httpd start
   chkconfig httpd on # 开机自动加载httpd服务
   ```

5. 创建 `Yum` 仓库的 `Http` 目录

   ```shell
   mkdir -p /var/www/html/cdh/5 # 创建目录
   cp -r cloudera-cdh5/RPMS /var/www/html/cdh/5/ # 拷贝解压后的cloudera包下的RPMS至目标目录
   cd /var/www/html/cdh/5 # 进入该目录
   createrepo . # 生成新的软件路径目录repodata  
   ```

6. 在三台主机上配置 `Yum` 源

   最后一步便是向 `Yum` 增加一个新的源, 指向我们在 `cdh01` 上创建的 `Yum` 仓库, 但是在这个环节的第一步中, 已经下载了一个 `Yum` 的源, 只需要修改这个源的文件, 把 `URL` 替换为 `cdh01` 的地址即可

   ```shell
   vim /etc/yum.repos.d/cloudera-cdh5.repo
   # 修改以下内容
   baseurl=http://cdh01/cdh/5/
   ```

   另外两台节点，同样需要下载并配置

   ```shell
   wget https://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/cloudera-cdh5.repo
   mv cloudera-cdh5.repo /etc/yum.repos.d/ 
   
   vim /etc/yum.repos.d/cloudera-cdh5.repo
   # 修改以下内容
   baseurl=http://cdh01/cdh/5/
   ```

# 3.安装zookeeper

集群规划

| 主机名  | 是否有 `Zookeeper` |
| :------ | :----------------- |
| `cdh01` | 有                 |
| `cdh02` | 有                 |
| `cdh03` | 有                 |

1. 安装zookeeper

   ```
   yum install -y zookeeper zookeeper-server
   ```

2. 所以需要先创建 `Zookeeper` 的数据目录，并且所有者指定给 `Zookeeper` 所使用的用户

   ```shell
   mkdir -p /var/lib/zookeeper
   chown -R zookeeper /var/lib/zookeeper/
   ```

3. 配置  `myid` 参数, 在不同节点要修改 `myid`的值

   ```
   service zookeeper-server init --myid=1
   ```

4. 修改配置文件

   CDH 版本的 `Zookeeper` 默认配置文件在 `/etc/zookeeper/conf/zoo.cfg`，修改以下内容

   ```
   server.1=cdh01:2888:3888
   server.2=cdh02:2888:3888
   server.3=cdh03:2888:3888
   ```

5. 启动zookeeper并检查

   ```
   service zookeeper-server start
   zookeeper-server status
   ```

注：以上步骤需要三台节点都要执行并进行配置

> `CDH` 版本的组件有一个特点, 默认情况下配置文件在 `/etc` 对应目录, 日志在 `/var/log` 对应目录, 数据在 `/var/lib` 对应目录, 例如说 `Zookeeper`, 配置文件放在 `/etc/zookeeper` 中, 日志在 `/var/log/zookeeper` 中, 其它的组件也遵循这样的规律

# 3.安装hadoop

集群规划

| 主机名  | 职责                                                         |
| :------ | :----------------------------------------------------------- |
| `cdh01` | `Yarn ResourceManager`, `HDFS NameNode`, `HDFS SecondaryNamenode`, `MapReduce HistroyServer`, `Hadoop Clients` |
| `cdh02` | `Yarn NodeManager`, `HDFS DataNode`                          |
| `cdh03` | `Yarn NodeManager`, `HDFS DataNode`                          |

1. 下载软件安装包

   cdh01中下载以下内容

   ```shell
   yum -y install hadoop hadoop-yarn-resourcemanager hadoop-hdfs-secondarynamenode hadoop-hdfs-namenode hadoop-mapreduce hadoop-mapreduce-historyserver hadoop-client
   ```

   cdh02，cdh03中下载以下内容

   ```shell
   yum -y install hadoop hadoop-yarn-nodemanager hadoop-hdfs-datanode hadoop-mapreduce hadoop-client
   ```

2. 配置HDFS

   - 说明：

     - 在 `CDH` 版本的组件中, 配置文件是可以动态变更的
     - 本质上, `CDH` 各组件的配置文件几乎都分布在 `/etc` 目录中, 例如 `Hadoop` 的配置文件就在 `/etc/hadoop/conf` 中, 这个 `conf` 目录是 `Hadoop` 当前所使用的配置文件目录, 但是这个目录其实是一个软链接, 当希望更改配置的时候, 只需要在 `/etc/hadoop` 中创建一个新的目录, 然后将 `conf` 指向这个新目录即可
     - 但是因为各个组件的 `conf` 目录对应了多个目录, 还需要修改其指向, 管理起来很麻烦, 所以 `CDH` 使用了 `Linux` 一个非常厉害的功能, 可以配置一个目录可以指向的多个目录, 同时可以根据优先级确定某个目录指向谁, 这个工具叫做 `alternatives`, 有如下几个常见操作

     ```shell
     alternatives --install # 讲一个新目录关联进来, 并指定其 ID 和优先级
     alternatives --set # 设置其指向哪个目录
     alternatives --display # 展示其指向哪个目录
     ```

   - 在所有节点中复制原始配置文件并生成新的配置目录, 让 `Hadoop` 使用使用新的配置目录

     这样做的目的是尽可能的保留原始配置文件, 以便日后恢复, **所以在所有节点中执行如下操作**

     - 创建新的配置目录

       ```
       cp -r /etc/hadoop/conf.empty /etc/hadoop/conf.itcast
       ```

     - 链接过去, 让 `Hadoop` 读取新的目录

       ```shell
       # 关联新的目录和 conf
       alternatives --install /etc/hadoop/conf hadoop-conf /etc/hadoop/conf.itcast 50
       # 设置指向
       alternatives --set hadoop-conf /etc/hadoop/conf.itcast
       # 显式当前指向
       alternatives --display hadoop-conf
       ```

3. 所有节点的新配置目录 `/etc/hadoop/conf.itcast` 中, 修改配置文件

   - core-site.xml

     ```xml
     <property>
         <name>fs.defaultFS</name>
         <value>hdfs://cdh01:8020</value>
     </property>
     ```

   - hdfs-site.xml

     ```xml
     <property>
         <name>dfs.namenode.name.dir</name>
         <value>file:///var/lib/hadoop-hdfs/cache/hdfs/dfs/name</value>
     </property>
     <property>
         <name>dfs.datanode.data.dir</name>
         <value>file:///var/lib/hadoop-hdfs/cache/hdfs/dfs/data</value>
     </property>
     <property>
         <name>dfs.permissions.superusergroup</name>
         <value>hadoop</value>
     </property>
     <property>
         <name>dfs.namenode.http-address</name>
         <value>cdh01:50070</value>
     </property>
     <property>
         <name>dfs.permissions.enabled</name>
         <value>false</value>
     </property>
     ```

4. **在所有节点中, 创建配置文件**指定的 `HDFS` 的 `NameNode` 和 `DataNode` 存放数据的目录, 并处理权限

   - 如下创建所需要的目录

     ```
     mkdir -p /var/lib/hadoop-hdfs/cache/hdfs/dfs/name
     mkdir -p /var/lib/hadoop-hdfs/cache/hdfs/dfs/data
     ```

   - 因为 `CDH` 比较特殊, 其严格按照 `Linux` 用户来管理和启动各个服务, 所以 `HDFS` 启动的时候使用的是 `hdfs` 用户组下的用户 `hdfs`, 需要创建文件后进行权限配置

     ```shell
     chown -R hdfs:hdfs /var/lib/hadoop-hdfs/cache/hdfs/dfs/name
     chown -R hdfs:hdfs /var/lib/hadoop-hdfs/cache/hdfs/dfs/data
     chmod 700 /var/lib/hadoop-hdfs/cache/hdfs/dfs/name
     chmod 700 /var/lib/hadoop-hdfs/cache/hdfs/dfs/data
     ```

5. **在cdh01节点**格式化 `NameNode`

   ```shell
   sudo -u hdfs hdfs namenode -format
   ```

6. 启动 `HDFS`

   - `cdh01` 上和 `HDFS` 有关的服务有 `NameNode`, `SecondaryNameNode`, 使用如下命令启动这两个组件

     ```shell
     service hadoop-hdfs-namenode start
     service hadoop-hdfs-secondarynamenode start
     ```

   - 在 `cdh02` 和 `cdh03` 上执行如下命令

     ```shell
     service hadoop-hdfs-datanode start
     ```

7. **在所有节点上**配置 `Yarn` **和** `MapReduce`

   前面已经完成配置目录创建等一系列任务了, 所以在配置 `Yarn` 的时候, 只需要去配置以下配置文件即可

   路径：`/etc/hadoop/conf.itcast/`

   - mapred-site.xml

     ```xml
     <property>
         <name>mapreduce.framework.name</name>
         <value>yarn</value>
     </property>
     <property>
         <name>mapreduce.jobhistory.address</name>
         <value>cdh01:10020</value>
     </property>
     <property>
         <name>mapreduce.jobhistory.webapp.address</name>
         <value>cdh01:19888</value>
     </property>
     <property>
         <name>hadoop.proxyuser.mapred.groups</name>
         <value>*</value>
     </property>
     <property>
         <name>hadoop.proxyuser.mapred.hosts</name>
         <value>*</value>
     </property>
     <property>
         <name>yarn.app.mapreduce.am.staging-dir</name>
         <value>/user</value>
     </property>
     ```

   - yarn-site.xml

     ```xml
     <property>
         <name>yarn.resourcemanager.hostname</name>
         <value>cdh01</value>
     </property>
     <property>
         <name>yarn.application.classpath</name>
         <value>
             $HADOOP_CONF_DIR,
             $HADOOP_COMMON_HOME/*,$HADOOP_COMMON_HOME/lib/*,
             $HADOOP_HDFS_HOME/*,$HADOOP_HDFS_HOME/lib/*,
             $HADOOP_MAPRED_HOME/*,$HADOOP_MAPRED_HOME/lib/*,
             $HADOOP_YARN_HOME/*,$HADOOP_YARN_HOME/lib/*
         </value>
     </property>
     <property>
         <name>yarn.nodemanager.aux-services</name>
         <value>mapreduce_shuffle</value>
     </property>
     <property>
         <name>yarn.nodemanager.local-dirs</name>
         <value>file:///var/lib/hadoop-yarn/cache/${user.name}/nm-local-dir</value>
     </property>
     <property>
         <name>yarn.nodemanager.log-dirs</name>
         <value>file:///var/log/hadoop-yarn/containers</value>
     </property>
     <property>
         <name>yarn.log.aggregation-enable</name>
         <value>true</value>
     </property>
     <property>
         <name>yarn.nodemanager.remote-app-log-dir</name>
         <value>hdfs:///var/log/hadoop-yarn/apps</value>
     </property>
     ```

8. 在所有节点上, 创建配置文件指定的存放数据的目录

   - 创建 `Yarn` 所需要的数据目录

     ```
     mkdir -p /var/lib/hadoop-yarn/cache
     mkdir -p /var/log/hadoop-yarn/containers
     mkdir -p /var/log/hadoop-yarn/apps
     ```

   - 赋予 `Yarn` 用户这些目录的权限

     ```
     chown -R yarn:yarn /var/lib/hadoop-yarn/cache /var/log/hadoop-yarn/containers /var/log/hadoop-yarn/apps
     ```

9. 为 `MapReduce` 准备 `HDFS` 上的目录, 因为是操作 `HDFS`, **只需要在一个节点执行**即可

   大致上是需要两种文件夹, 一种用做于缓存, 一种是用户目录

   - 为 `MapReduce` 缓存目录赋权

     ```
     sudo -u hdfs hadoop fs -mkdir /tmp
     sudo -u hdfs hadoop fs -chmod -R 1777 /tmp
     sudo -u hdfs hadoop fs -mkdir -p /user/history
     sudo -u hdfs hadoop fs -chmod -R 1777 /user/history
     sudo -u hdfs hadoop fs -chown mapred:hadoop /user/history
     ```

   - 为 `MapReduce` 创建用户目录

     ```
     sudo -u hdfs hadoop fs -mkdir /user/$USER
     sudo -u hdfs hadoop fs -chown $USER /user/$USER
     ```

10. 启动 `Yarn`

    - 在 `cdh01` 上启动 `ResourceManager` 和 `HistoryServer`

      ```
      service hadoop-yarn-resourcemanager start
      service hadoop-mapreduce-historyserver start
      ```

    - 在 `cdh02` 和 `cdh03` 上启动 `NodeManager`

      ```
      service hadoop-yarn-nodemanager start
      ```

# 4.安装MySql

安装 `MySQL` 有很多方式, 可以直接准备压缩包上传解压安装, 也可以通过 `Yum` 来安装, 从方便和是否主流两个角度来看, 通过 `Yum` 来安装会比较舒服, `MySQL` 默认是单机的, 所以在一个主机上安装即可, 我们选择在 `cdh01` 上安装,。

1. 安装

   因为要从 `Yum` 安装, 但是默认的 `Yum` 源是没有 `MySQL` 的, 需要导入 `Oracle` 的源, 然后再安装

   - 下载 `Yum` 源配置

     ```shell
     wget http://repo.mysql.com/mysql-community-release-el6-5.noarch.rpm
     rpm -ivh mysql-community-release-el6-5.noarch.rpm
     ```

   - 安装 MySQL

     ```
     yum install -y mysql-server
     ```

     注：此步执行相当漫长

2. 启动和配置

   现在 `MySQL` 的安全等级默认很高, 所以要通过一些特殊的方式来进行密码设置, 在启动 `MySQL` 以后要单独的进行配置

   - 启动mysql

     ```
     service mysqld start
     ```

   - 修改配置文件，关闭mysql的密码强验证

     vim /etc/my.cnf

     ```
     validate_password=OFF
     ```

     修改完成之后需要重启mysql

     ```
     service mysqld restart
     ```

   - 通过 `MySQL` 提供的工具, 设置 `root` 密码

     ```
     mysql_secure_installation
     ```

     ```
     Enter current password for root (enter for none): #直接回车即可
     Set root password? [Y/n] Y
     New password:000000
     Re-enter new password:000000 
     Remove anonymous users? [Y/n] Y
     Disallow root login remotely? [Y/n] N
     Remove test database and access to it? [Y/n] y
     Reload privilege tables now? [Y/n] y
     ```

     

# 5.安装Hive

因为 `Hive` 需要使用 `MySQL` 作为元数据库, 所以需要在 `MySQL` 为 `Hive` 创建用户, 创建对应的表

1. 安装hive软件包

   - 安装 `Hive` 依然使用 `CDH` 的 `Yum` 仓库

     ```
     yum install -y hive hive-metastore hive-server2
     ```

   - 如果想要 `Hive` 使用 `MySQL` 作为元数据库, 那需要给 `Hive` 一个 `MySQL` 的 `JDBC` 包

     ```
     yum install -y mysql-connector-java
     ln -s /usr/share/java/mysql-connector-java.jar /usr/lib/hive/lib/mysql-connector-java.jar
     ```

2. mysql中增加hive用户

   - 进入mysql

     ```
     mysql -u root -p000000
     ```

   - 为 `Hive` 创建数据库

     ```
     CREATE DATABASE metastore;
     USE metastore;
     ```

   - 创建 `Hive` 用户

     ```
     CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';
     ```

   - 为 `Hive` 用户赋权

     ```shell
     REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'hive'@'%';
     GRANT ALL PRIVILEGES ON metastore.* TO 'hive'@'%';
     FLUSH PRIVILEGES;
     ```

3. 配置 `Hive`

   在启动 `Hive` 之前, 要配置 `Hive` 一些参数, 例如使用 `MySQL` 作为数据库之类的配置

   删除原来property的配置，只保留以下内容即可

   vim /etc/hive/conf/hive-site.xml

   ```xml
   <!-- /usr/lib/hive/conf/hive-site.xml -->
   <property>
       <name>javax.jdo.option.ConnectionURL</name>
       <value>jdbc:mysql://cdh01/metastore</value>
   </property>
   <property>
       <name>javax.jdo.option.ConnectionDriverName</name>
       <value>com.mysql.jdbc.Driver</value>
   </property>
   <property>
       <name>javax.jdo.option.ConnectionUserName</name>
       <value>hive</value>
   </property>
   <property>
       <name>javax.jdo.option.ConnectionPassword</name>
       <value>hive</value>
   </property>
   <property>
       <name>datanucleus.autoCreateSchema</name>
       <value>false</value>
   </property>
   <property>
       <name>datanucleus.fixedDatastore</name>
       <value>true</value>
   </property>
   <property>
       <name>datanucleus.autoStartMechanism</name>
       <value>SchemaTable</value>
   </property>
   <property>
       <name>hive.metastore.uris</name>
       <value>thrift://cdh01:9083</value>
   </property>
   <property>
       <name>hive.metastore.schema.verification</name>
       <value>true</value>
   </property>
   <property>
       <name>hive.support.concurrency</name>
       <description>Enable Hive's Table Lock Manager Service</description>
       <value>true</value>
   </property>
   <property>
       <name>hive.support.concurrency</name>
       <value>true</value>
   </property>
   <property>
       <name>hive.zookeeper.quorum</name>
       <value>cdh01</value>
   </property>
   
   ```

4. 初始化表结构

   使用 `Hive` 之前, `MySQL` 中还没有任何内容, 所以需要先为 `Hive` 初始化数据库, 创建必备的表和模式. `Hive` 提供了方便的工具, 提供 `MySQL` 的连接信息, 即可帮助我们创建对应的表

   ```shell
   /usr/lib/hive/bin/schematool -dbType mysql -initSchema -passWord hive -userName hive -url jdbc:mysql://cdh01/metastore
   ```

5. 启动hive

   默认版本的 `Hive` 只提供了一个 `Shell` 命令, 通过这一个单独的 `Shell` 命令以指定参数的形式启动服务, 但是 `CDH` 版本将 `Hive` 抽取为两个独立服务, 方便通过服务的形式启动 `Hive`, `hive-metastore` 是元数据库, `hive-server2` 是对外提供连接的服务

   ```shell
   service hive-metastore start
   service hive-server2 start
   ```

   通过 `beeline` 可以连接 `Hive` 验证是否启动成功, 启动 `beeline` 后, 通过如下字符串连接 `Hive`

   ```shell
   !connect jdbc:hive2://cdh01:10000 username password org.apache.hive.jdbc.HiveDriver
   ```

# 6.安装kudu

安装 `Kudu` 依然使用我们已经配置好的 `Yum` 仓库来进行, 整体步骤非常简单, 但是安装上分为 `Master` 和 `Tablet server`

集群规划

| 节点    | 职责            |
| :------ | :-------------- |
| `cdh01` | `Master server` |
| `cdh02` | `Tablet server` |
| `cdh03` | `Tablet server` |

1. 安装 Master server 的软件包

   根据集群规划, 尽量让 `cdh01` 少一些负载, 所以只在 `cdh01` 上安装 `Master server`, 命令如下

   ```shell
   yum install -y kudu kudu-master kudu-client0 kudu-client-devel
   ```

2. 配置 Master server

   `Kudu` 的 `Master server` 没有太多可以配置的项目, 默认的话日志和数据都会写入到 `/var` 目录下, 只需要修改一下 `BlockManager` 的方式即可, 在虚拟机上使用 `Log` 方式可能会带来一些问题, 改为 `File` 方式

   vim /etc/kudu/conf/master.gflagfile

   ```
   --block_manager=file
   ```

   但是有一点需要注意, 一定确保 `ntp` 服务是开启的, 可以使用 `ntpstat` 来查看, 因为 `Kudu` 对时间同步的要求非常高, `ntp` 必须可以自动同步

   ```
   # 查看时间是否是同步状态
   ntpstat
   ```

3. 运行 Master server

   - 运行 `Master server`

     ```
     service kudu-master start
     ```

   - 查看 `Web ui` 确认 `Master server` 已经启动

     ```
     http://cdh01:8051/masters
     ```

4. 安装 Tablet server 的软件包

   根据集群规划, 在 `cdh02`, `cdh03` 中安装 `Tablet server`, 负责更为繁重的工作

   ```
   yum install -y kudu kudu-tserver kudu-client0 kudu-client-devel
   ```

5. 配置 Tablet server

   `Master server` 相对来说没什么需要配置的, 也无须知道各个 `Tablet server` 的位置, 但是对于 `Tablet server` 来说, 必须配置 `Master server` 的位置, 因为一般都是从向主注册自己

   在 `cdh02`, `cdh03` 修改 `/etc/kudu/conf/tserver.gflagfile` 为如下内容, 如果有多个 `Master server` 实例, 用逗号分隔地址即可

   ```
   --tserver_master_addrs=cdh01:7051
   ```

   同时 `Tablet server` 也需要设置 `BlockManager`

   ```
   --block_manager=file
   ```

6. 运行 `Tablet server`

   - 启动

     ```text
     service kudu-tserver start
     ```

   - 通过 `Web ui` 查看是否已经启动成功

     ```text
     http://cdh02:8050
     ```

# 7.安装Impala

`Kudu` 没有 `SQL` 解析引擎, 因为 `Cloudera` 准备使用 `Impala` 作为 `Kudu` 的 `SQL` 引擎, 所以既然使用 `Kudu` 了, `Impala` 几乎也是必不可少的, 安装 `Impala` 之前, 先了解以下 `Impala` 中有哪些服务

| 服务           | 作用                                                         |
| :------------- | :----------------------------------------------------------- |
| `Catalog`      | `Impala` 的元信息仓库, 但是不同的是这个 `Catalog` 强依赖 `Hive` 的 `MetaStore`, 会从 `Hive` 处获取元信息 |
| `StateStore`   | `Impala` 的协调节点, 负责异常恢复                            |
| `ImpalaServer` | `Impala` 是 `MPP` 架构, 这是 `Impala` 处理数据的组件, 会读取 `HDFS`, 所以一般和 `DataNode` 部署在一起, 提升性能, 每个 `DataNode` 配一个 `ImpalaServer` |

所以, `cdh01` 上应该有 `Catalog` 和 `StateStore`, 而不应该有 `ImpalaServer`, 因为 `cdh01` 中没有 `DataNode`

集群规划

| 节点    | 职责                                   |
| :------ | :------------------------------------- |
| `cdh01` | `impala-state-store`, `impala-catalog` |
| `cdh02` | `impala-server`                        |
| `cdh03` | `impala-server`                        |

1. 安装软件包

   - 安装主节点 `cdh01` 所需要的软件包

     ```text
     yum install -y impala impala-state-store impala-catalog impala-shell
     ```

   - 安装其它节点所需要的软件包

     ```text
     yum install -y impala impala-server
     ```

2. 针对所有节点进行配置

   - 软链接 `Impala` 所需要的 `Hadoop` 配置文件, 和 `Hive` 的配置文件

     `Impala` 需要访问 `Hive` 的 `MetaStore`, 所以需要 `hive-site.xml` 来读取其位置

     `Impala` 需要访问 `HDFS`, 所以需要读取 `hdfs-site.xml` 来获取访问信息, 同时也需要读取 `core-site.xml` 获取一些信息

     ```text
     ln -s /etc/hadoop/conf/core-site.xml /etc/impala/conf/core-site.xml
     ln -s /etc/hadoop/conf/hdfs-site.xml /etc/impala/conf/hdfs-site.xml
     ln -s /etc/hive/conf/hive-site.xml /etc/impala/conf/hive-site.xml
     ```

   - 配置 `Impala` 的主服务位置, 以供 `ImpalaServer(Impalad)` 访问, 修改 `Impala` 的默认配置文件 `/etc/default/impala`, `/etc/default` 往往放置 `CDH` 版本中各组件的环境变量类的配置文件

     ```text
     IMPALA_CATALOG_SERVICE_HOST=cdh01
     IMPALA_STATE_STORE_HOST=cdh01
     
     
     #在该文件的IMPALA_SERVER_ARGS="内部添加如下内容吗，需要注意一定要在这个参数内部
     --kudu_masters_hosts=cdh01:7051
     ```

3. 启动

   - 启动 cdh01

     ```text
     service impala-state-store start
     service impala-catalog start
     ```

   - 启动其它节点

     ```text
     service impala-server start
     ```

     注：可能会报错，可以查看一下对应的日志文件，一般来说是25000端口问题，只需等待一会重启即可

   - 通过 Web ui 查看是否启动成功

     ```text
     http://cdh02:25000
     ```
     
   - 使用命令impala-shell连接
   
     因为server服务安装在cdh02上，所以需要连接cdh02
   
     ```
      impala-shell -i cdh02
     ```
   
     

# 8.安装 Hue

`Hue` 其实就是一个可视化平台, 主要用于浏览 `HDFS` 的文件, 编写和执行 `Hive` 的 `SQL`, 以及 `Impala` 的 `SQL`, 查看数据库中数据等, 而且 `Hue` 一般就作为 `CDH` 数据平台的入口, 所以装了 `CDH` 而不装 `Hue` 会觉得少了点什么, 面试的时候偶尔也会问 `Hue` 的使用, 所以我们简单安装, 简单使用 `Hue` 让大家了解以下这个可视化工具

1. Hue组件安装

   使用 `Yum` 即可简单安装

   ```text
   yum -y install hue
   ```

2. 配置 Hue

   `Hue` 的配置就会稍微优点复杂, 因为 `Hue` 要整合其它的一些工具, 例如访问 `HDFS`, 所以配置要从两方面说, 一是 `HDFS` 要允许 `Hue` 访问, 二是配置给 `Hue` 如何访问 `HDFS` (以及如何访问其它程序)

   - 修改hue配置文件

      vim /etc/hue/conf/hue.ini 

     ```shell
     # 配置 Hue 所占用的Web端口，只读模式下输入 /HDFS 就可以找到
     fs_defaultfs=hdfs://cdh01:8020 # 在917行    
     webhdfs_url=http://cdh01:50070/webhdfs/v1 # 在925行   
     
     # 配置 Impala的访问方式，只读模式下输入 /impala 就可以找到
     server_host=cdh01 #在1099行
     
     # 配置 Hive 的访问方式，只读模式下输入 /beeswax 就可以找到
     hive_server_host=cdh01 #在1028行
     ```

   - 配置 `HDFS`, 允许 `Hue` 的访问

     修改 `hdfs-site.xml` 增加如下内容, 以便让 `Hue` 用户可以访问 `HDFS` 中的文件

     vim /etc/hadoop/conf/hdfs-site.xml 

     ```xml
     <property>
         <name>hadoop.proxyuser.hue.hosts</name>
         <value>*</value>
     </property>
     <property>
         <name>hadoop.proxyuser.hue.groups</name>
         <value>*</value>
     </property>
     <property>
         <name>hadoop.proxyuser.hosts</name>
         <value>*</value>
     </property>
     <property>
         <name>hadoop.proxyuser.httpfs.groups</name>
         <value>*</value>
     </property>
     ```

3. 启动 Hue

   ```
   service hue start
   ```

   浏览器访问http://cdh01:8888/

   默认的账户名和密码皆为admin

**开机要启动的服务**

| 服务                      | 命令                                           |
| :------------------------ | :--------------------------------------------- |
| `httpd`                   | `service httpd start`                          |
| `Zookeeper`               | `service zookeeper-server start`               |
| `hdfs-namenode`           | `service hadoop-hdfs-namenode start`           |
| `hdfs-datanode`           | `service hadoop-hdfs-datanode start`           |
| `hdfs-secondarynamenode`  | `service hadoop-hdfs-secondarynamenode start`  |
| `yarn-resourcemanager`    | `service hadoop-yarn-resourcemanager start`    |
| `mapreduce-historyserver` | `service hadoop-mapreduce-historyserver start` |
| `yarn-nodemanager`        | `service hadoop-yarn-nodemanager start`        |
| `hive-metastore`          | `service hive-metastore start`                 |
| `hive-server2`            | `service hive-server2 start`                   |
| `kudu-master`             | `service kudu-master start`                    |
| `kudu-tserver`            | `service kudu-tserver start`                   |
| `impala-state-store`      | `service impala-state-store start`             |
| `impala-catalog`          | `service impala-catalog start`                 |
| `impala-server`           | `service impala-server start`                  |
| `hue`                     | `service hue start`                            |

