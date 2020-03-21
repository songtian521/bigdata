# 10_Hadoop完全分布式搭建

## 1.环境配置

1. 关闭防火墙

   - 关闭防火墙：systemctl stop firewalld.service
   - 禁用防火墙：systemctl disable firewalld.service
   - 查看防火墙：systemctl status firewalld.service

2. 关闭Selinux：

   - vi /etc/selinux/config

     将SELINUX=enforcing改为SELINUX=disabled

3. 修改IP

   - vi /etc/sysconfig/network-scripts/ifcfg-ens33

   - 修改如下内容

     ```
     BOOTPROTO=static
     ONBOOT=yes
     
     IPADDR=192.168.X.51
     GATEWAY=192.168.X.2
     DNS1=8.8.8.8
     DNS2=8.8.4.4
     NETMASK=255.255.255.0
     ```

   - 注意：IPADDRh和GATEWAY需要修改为自己的虚拟机IP地址，其中GATEWAY只需要修改`X`的内容为IPADDR的第三个网段即可

4. 修改DNS客户机配置文件

    vim /etc/resolv.conf

   ```
   nameserver 8.8.8.8
   nameserver 8.8.4.4
   ```

5. 重启网卡

   ```
   systemctl restart network
   ```

6. 修改主机名

   ```
   hostnamectl set-hostname 主机名
   ```

   使用命令 `hostname`可查看当前主机名

7. IP和主机名关系映射

   - 虚拟机中`vi /etc/hosts`

     - 添加如下内容

       ```
       192.168.64.129 bigdata111
       192.168.64.130 bigdata222
       192.168.64.131 bigdata333
       ```

       注：修改为自己的虚拟机IP

   - 在windows中的C:\Windows\System32\drivers\etc目录下找到hosts

     - 添加如下内容

       ```
       192.168.64.129 bigdata111
       192.168.64.130 bigdata222
       192.168.64.131 bigdata333
       ```

       注：修改为自己的虚拟机IP



## 2.Hadoop搭建（伪分布）

配置集群节点分布

|      | bigdata111                            | bigdata112  | bigdata113  |
| ---- | ------------------------------------- | ----------- | ----------- |
| HDFS | NameNode、SecondaryNameNode、DataNode | DataNode    | DataNode    |
| YARN | ResourceManager、NodeManager          | NodeManager | NodeManager |

1. 上传`hadoop-2.8.4.tar.gz`至opt/software

2. 解压`hadoop-2.8.4.tar.gz`至opt/module

   ```
   tar -zxvf hadoop-2.8.4.tar.gz -C /opt/module
   ```

3. 配置环境变量

   `vi /etc/profile`

   添加如下内容

   ```
   export HADOOP_HOME=/opt/module/hadoop-2.8.4
   export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
   ```

   注：需要根据自己的路径设置

4. 重启配置文件

   `source /etc/profile`

5. 修改四个配置文件

   先在`/opt/module/hadoop-2.8.4/`目录下创建logs和data目录，分别用于存储日志文件和数据文件

   进入目录`/opt/module/hadoop-2.8.4/etc/hadoop`

   注：以下添加内容都需要写在`<configuration></configuration>`标签之内

   1. 修改文件core-site.xml，添加如下内容

      ```
      <!-- 指定HDFS中NameNode的地址 -->
              <property>
                      <name>fs.defaultFS</name>
              <value>hdfs://bigdata111:9000</value>
              </property>
      
      <!-- 指定hadoop运行时产生文件的存储目录 -->
              <property>
                      <name>hadoop.tmp.dir</name>
                      <value>/opt/module/hadoop-2.8.4/data</value>
              </property>
      ```

      注：目录和主机名需要修改为自己的

   2. 修改文件hdfs-site.xml，添加如下内容

      ```
      <!--数据冗余数-->
              <property>
                      <name>dfs.replication</name>
                      <value>3</value>
              </property>
      <!--secondary的地址-->
              <property>
                      <name>dfs.namenode.secondary.http-address</name>
                      <value>bigdata111:50090</value>
              </property>
      <!--关闭权限-->
              <property>
                      <name>dfs.permissions</name>
                      <value>false</value>
              </property>
      
      ```

      注：主机名需要修改为自己虚拟机自己的

   3. 修改文件yarn-site.xml，添加如下内容

      ```
      <!-- reducer获取数据的方式 -->
              <property>
                       <name>yarn.nodemanager.aux-services</name>
                       <value>mapreduce_shuffle</value>
              </property>
      
      <!-- 指定YARN的ResourceManager的地址 -->
              <property>
                      <name>yarn.resourcemanager.hostname</name>
                      <value>bigdata111</value>
              </property>
      <!-- 日志聚集功能使能 -->
              <property>
                      <name>yarn.log-aggregation-enable</name>
                      <value>true</value>
              </property>
      <!-- 日志保留时间设置7天(秒) -->
              <property>
                      <name>yarn.log-aggregation.retain-seconds</name>
                      <value>604800</value>
              </property>
      
      ```

      注：主机名需要修改为自己的

   4. 修改文件mapred-site.xml，添加如下内容

      注：只有一个mapred-site.xml.template的临时文件，可使用命令`mv mapred-site.xml.template mapred-site.xml`修改

      ```
      <!-- 指定mr运行在yarn上-->
              <property>
                      <name>mapreduce.framework.name</name>
                      <value>yarn</value>
              </property>
      <!--历史服务器的地址-->
              <property>
                  <name>mapreduce.jobhistory.address</name>
                  <value>bigdata111:10020</value>
              </property>
      <!--历史服务器页面的地址-->
              <property>
                      <name>mapreduce.jobhistory.webapp.address</name>
                      <value>bigdata111:19888</value>
              </property>
      ```

      注：主机名需要修改为自己的

   5. 分别修改：hadoop-env.sh、yarn-env.sh、mapred-env.sh

      分别在最末尾添加配置的JAVA_HOME路径，如下：

      ```
      export JAVA_HOME=/opt/module/jdk1.8.0_144
      ```

   6. 修改slaves 文件

      覆盖localhost

      ```
      bigdata111
      bigdata222
      bigdata333
      ```

6. 格式化Namenode

   `hdfs namenode -format`

   无报错就算成功，如：

   最下面的一些内容：

   ```
   19/10/03 16:29:40 INFO namenode.FSNamesystem: dfs.namenode.safemode.extension     = 30000
   19/10/03 16:29:40 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.window.num.buckets = 10
   19/10/03 16:29:40 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.num.users = 10
   19/10/03 16:29:40 INFO metrics.TopMetrics: NNTop conf: dfs.namenode.top.windows.minutes = 1,5,25
   19/10/03 16:29:40 INFO namenode.FSNamesystem: Retry cache on namenode is enabled
   19/10/03 16:29:40 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis
   19/10/03 16:29:40 INFO util.GSet: Computing capacity for map NameNodeRetryCache
   19/10/03 16:29:40 INFO util.GSet: VM type       = 64-bit
   19/10/03 16:29:40 INFO util.GSet: 0.029999999329447746% max memory 966.7 MB = 297.0 KB
   19/10/03 16:29:40 INFO util.GSet: capacity      = 2^15 = 32768 entries
   19/10/03 16:29:40 INFO namenode.FSImage: Allocated new BlockPoolId: BP-1701936615-192.168.64.129-1570091380659
   19/10/03 16:29:40 INFO common.Storage: Storage directory /opt/module/hadoop-2.8.4/data/dfs/name has been successfully formatted.
   19/10/03 16:29:40 INFO namenode.FSImageFormatProtobuf: Saving image file /opt/module/hadoop-2.8.4/data/dfs/name/current/fsimage.ckpt_0000000000000000000 using no compression
   19/10/03 16:29:40 INFO namenode.FSImageFormatProtobuf: Image file /opt/module/hadoop-2.8.4/data/dfs/name/current/fsimage.ckpt_0000000000000000000 of size 321 bytes saved in 0 seconds.
   19/10/03 16:29:40 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
   19/10/03 16:29:40 INFO util.ExitUtil: Exiting with status 0
   19/10/03 16:29:40 INFO namenode.NameNode: SHUTDOWN_MSG: 
   /************************************************************
   SHUTDOWN_MSG: Shutting down NameNode at bigdata111/192.168.64.129
   ************************************************************/
   ```

   

7. 查看进程 jps

   ```
   [root@bigdata111 hadoop]# jps
   3890 NodeManager
   4003 Jps
   3606 ResourceManager
   3162 NameNode
   3292 DataNode
   3454 SecondaryNameNode
   ```

   注意：进程一个不可以缺少，如果缺少则代表配置错误

8. 启动Hadoop

   使用命令`start-all.sh`

   ```
   [root@bigdata111 hadoop]# start-all.sh
   This script is Deprecated. Instead use start-dfs.sh and start-yarn.sh
   Starting namenodes on [bigdata111]
   root@bigdata111's password: 
   bigdata111: starting namenode, logging to /opt/module/hadoop-2.8.4/logs/hadoop-root-namenode-bigdata111.out
   root@bigdata111's password: 
   bigdata111: starting datanode, logging to /opt/module/hadoop-2.8.4/logs/hadoop-root-datanode-bigdata111.out
   Starting secondary namenodes [bigdata111]
   root@bigdata111's password: 
   bigdata111: starting secondarynamenode, logging to /opt/module/hadoop-2.8.4/logs/hadoop-root-secondarynamenode-bigdata111.out
   starting yarn daemons
   starting resourcemanager, logging to /opt/module/hadoop-2.8.4/logs/yarn-root-resourcemanager-bigdata111.out
   root@bigdata111's password: 
   bigdata111: starting nodemanager, logging to /opt/module/hadoop-2.8.4/logs/yarn-root-nodemanager-bigdata111.out
   ```

   注：因为还没有配置免密登录，所以需要输入密码

## 3.Hadoop免密登录（单节点）

注：配置前需要确保Hadoop已经关闭，使用命令`stop-all.sh`关闭Hadoop

1. 生成公钥和私钥

   `ssh-keygen -t rsa`

2. 将公钥拷贝到要免密登录的目标机器上

   `ssh-copy-id 主机名1`

3. 检验

   - 进入目录/root/.ssh

   - 执行命令`ssh bigdata111`

     ```
     [root@bigdata111 .ssh]# ssh bigdata111
     Last failed login: Thu Oct  3 16:35:46 CST 2019 from bigdata111 on ssh:notty
     There were 5 failed login attempts since the last successful login.
     Last login: Thu Oct  3 15:49:54 2019 from 192.168.64.1
     ```

     如上则成功，如果失败会让你在输入密码的

   - 使用exit退出

     ```
     [root@bigdata111 ~]# exit
     logout
     Connection to bigdata111 closed.
     ```

4. 再次执行start-all.sh

   如果不需要输入密码则成功

   ```
   [root@bigdata111 .ssh]# start-all.sh
   This script is Deprecated. Instead use start-dfs.sh and start-yarn.sh
   Starting namenodes on [bigdata111]
   bigdata111: starting namenode, logging to /opt/module/hadoop-2.8.4/logs/hadoop-root-namenode-bigdata111.out
   bigdata111: starting datanode, logging to /opt/module/hadoop-2.8.4/logs/hadoop-root-datanode-bigdata111.out
   Starting secondary namenodes [bigdata111]
   bigdata111: starting secondarynamenode, logging to /opt/module/hadoop-2.8.4/logs/hadoop-root-secondarynamenode-bigdata111.out
   starting yarn daemons
   starting resourcemanager, logging to /opt/module/hadoop-2.8.4/logs/yarn-root-resourcemanager-bigdata111.out
   bigdata111: starting nodemanager, logging to /opt/module/hadoop-2.8.4/logs/yarn-root-nodemanager-bigdata111.out
   ```

5. ssh文件夹下的文件功能解释

   ​	（1）~/.ssh/known_hosts	：记录ssh访问过计算机的公钥(public key)

   ​	（2）id_rsa	：生成的私钥

   ​	（3）id_rsa.pub	：生成的公钥

   ​	（4）authorized_keys	：存放授权过得无秘登录服务器公钥

## 4.Hadoop集群搭建（完全分布）

1. 克隆主机

   注：克隆的时候需要选择完全克隆

2. 修改主机名

   ```
   hostnamectl set-hostname 主机名
   ```

   查看当前主机名

   ```
   hostname
   ```

3. 修改IP

   ```
   vi /etc/sysconfig/network-scripts/ifcfg-ens33
   ```

   仅修改一条即可

   ```
   IPADDR=192.168.64.130
   ```

   修改为自己的IP

4. 重启网卡

   ```
   systemctl restart network
   ```

5. 检查是否联网

   ```
   ping www.baidu.com
   ```

6. 查看IP

   ```
   ip addr
   ```

7. 配置几台虚拟机，则在每台虚拟机中执行上述步骤

8. 删除之前的数据

   - 进入/opt/module/hadoop-2.8.4目录执行`rm -rf data/* logs/*`（三台都要执行）

   ```
   cd /opt/module/hadoop-2.8.4
   ```

9. 格式化111节点（只在111节点执行）

   ```
    hdfs namenode -format
   ```

   

10. jps查看进程

    - 111节点

      ```
      5569 Jps
      5254 SecondaryNameNode
      4235 ResourceManager
      4971 NameNode
      5099 DataNode
      5467 NodeManage
      ```

    - 222节点

      ```
      [root@bigdata222 hadoop-2.8.4]# jps
      3462 DataNode
      3562 NodeManager
      3674 Jps
      ```

    - 333节点

      ```
      [root@bigdata333 hadoop-2.8.4]# jps
      3874 NodeManager
      3986 Jps
      3774 DataNode
      ```

      

## 5.集群状态下SSH免密搭建

1. 生成公钥和私钥：

   ```
   ssh-keygen -t rsa
   ```

   

2. 将公钥拷贝到要免密登录的目标机器上

   `ssh-copy-id 目标1`

   `ssh-copy-id 目标2`

   `ssh-copy-id 目标3`

   注：在另外三台机器上分别执行，共执行9遍

3. 检测

   `ssh 主机名`

   如果可以跳入其他节点则成功，否则失败

   eg:

   ```
   [root@bigdata111 ~]# ssh bigdata222
   Last login: Thu Oct  3 17:39:11 2019 from bigdata333
   ```

   使用`exit`退出

## 6.Hadoop启动和停止命令

| 启动/停止历史服务器      | mr-jobhistory-daemon.sh start\|stop historyserver |
| ------------------------ | ------------------------------------------------- |
| 启动/停止总资源管理器    | yarn-daemon.sh start\|stop resourcemanager        |
| 启动/停止节点管理器      | yarn-daemon.sh start\|stop nodemanager            |
| 启动/停止 NN 和 DN       | start\|stop-dfs.sh                                |
| 启动/停止 RN 和 NM       | start\|stop-yarn.sh                               |
| 启动/停止 NN、DN、RN、NM | start\|stop-all.sh                                |
| 启动/停止 NN             | hadoop-daemon.sh start\|stop namenode             |
| 启动/停止 DN             | hadoop-daemon.sh start\|stop datanode             |

注：以上命令都在$HADOOP_HOME/sbin下，如果使用记得配置环境变量

## 7.更改节点（可选）

案例以更改ResourceManager到bigdata222节点为例

1. 进入目录/opt/module/hadoop-2.8.4/etc/hadoop，找到yarn-site.xml文件

2. 找到注释：**指定YARN的ResourceManager的地址**，更改节点为bigdata222（其他类似）（集群同步）

   ```xml
   <!-- 指定YARN的ResourceManager的地址 -->
           <property>
                   <name>yarn.resourcemanager.hostname</name>
                   <value>bigdata222</value>
           </property>
   ```

3. 更改后启动集群节点进程如下

   ```xml
   [root@bigdata111 hadoop]# jps
   1440 NameNode
   2036 Jps
   1546 DataNode
   1741 SecondaryNameNode
   1901 NodeManager
   
   [root@bigdata222 hadoop]# jps
   1921 Jps
   1491 ResourceManager
   1366 DataNode
   1613 NodeManager
   
   [root@bigdata333 hadoop]# jps
   1495 NodeManager
   1372 DataNode
   1629 Jps
   ```

   注意：Namenode和ResourceManger如果不是同一台机器，不能在NameNode上启动 yarn，应该在ResouceManager所在的机器上启动yarn。

