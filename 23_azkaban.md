# 23_azkaban

## 1.概述

### 1.1 概念

Azkaban是由Linkedin（领英）公司推出的一个批量工作流任务调度器，主要用于在一个工作流内以一个特定的顺序运行一组工作和流程，它的配置是通过简单的key:value对的方式，通过配置中的dependencies 来设置依赖关系。**Azkaban使用job配置文件建立任务之间的依赖关系**，并提供一个易于使用的web用户界面维护和跟踪你的工作流。

### 1.2 特点

1. 兼容任何版本的hadoop
2. 易于使用的web用户界面
3. 简单的工作流的上传
4. 方便设置任务之间的关系
5. 调度工作流
6. 模块化和可插拔的插件机制
7. 认证/授权（权限的工作）
8. 能够杀死并重新启动工作流
9. 有关失败和成功的电子邮件提醒



### 1.3 工作流产生背景

工作流（Workflow），指“**业务过程的部分或整体在计算机应用环境下的自动化**”。是对工作流程及其各操作步骤之间业务规则的抽象、概括描述。工作流解决的主要问题是：**为了实现某个业务目标，利用计算机软件在多个参与者之间按某种预定规则自动传递文档、信息或者任务。**

一个完整的数据分析系统通常都是由多个前后依赖的模块组合构成的：数据采集、数据预处理、数据分析、数据展示等。各个模块单元之间存在时间先后**依赖关系**，且**存在着周期性重复。**

为了很好地组织起这样的复杂执行计划，需要一个工作流调度系统来调度执行。

### 1.4 工作流调度实现方式

1. 简单的任务调度：直接使用linux的crontab来定义,但是缺点也是比较明显，无法设置依赖。
2. 复杂的任务调度自主开发调度平台，使用开源调度系统，比如**azkaban**、Apache Oozie、Cascading、Hamake等。
3. 其中知名度比较高的是Apache Oozie，但是其配置工作流的过程是编写大量的XML配置，而且代码复杂度比较高，不易于二次开发。 

### 1.5为什么需要工作流调度系统

1. 一个完整的数据分析系统通常都是由大量的任务单元组成

   shell脚本程序，java程序，mapreduce程序，hive脚本等

2. 各任务单元之间存在时间先后及前后依赖关系

   如：我们可能有这样一个需求，某业务系统每天产生20G原始数据，我们每天都要对其进行处理，处理步骤如下所示

   1. 通过Hadoop先将原始数据上传到HDFS上（HDFS的操作）；
   2. 使用MapReduce对原始数据进行清洗（MapReduce的操作）；
   3. 将清洗后的数据导入到hive表中（hive的导入操作）；
   4. 对Hive中多个表的数据进行JOIN处理，得到一张hive的明细表（创建中间表）；
   5.  通过对明细表的统计和分析，得到结果报表信息（hive的查询操作）；

### 1.6 azkaban的适用场景

根据以上业务场景： （2）任务依赖（1）任务的结果，（3）任务依赖（2）任务的结果，（4）任务依赖（3）任务的结果，（5）任务依赖（4）任务的结果。一般的做法是，先执行完（1）再执行（2），再一次执行（3）（4）（5）。

这样的话，整个的执行过程都需要人工参加，并且得盯着各任务的进度。但是我们的很多任务都是在深更半夜执行的，通过写脚本设置crontab执行。其实，整个过程类似于一个有向无环图（DAG）。每个子任务相当于大任务中的一个节点，也就是，我们需要的就是一个工作流的调度器，而Azkaban就是能解决上述问题的一个调度器。

### 1.7工作流调度工具之间对比

下面的表格对上述四种hadoop工作流调度器的关键特性进行了比较，尽管这些工作流调度器能够解决的需求场景基本一致，但在**设计理念，目标用户，应用场景等方面还是存在显著的区别**，在做技术选型的时候，可以提供参考

| 特性               | Hamake               | Oozie             | Azkaban                        | Cascading |
| ------------------ | -------------------- | ----------------- | ------------------------------ | --------- |
| 工作流描述语言     | XML                  | XML (xPDL based)  | text file with key/value pairs | Java API  |
| 依赖机制           | data-driven          | explicit          | explicit                       | explicit  |
| 是否要web容器      | No                   | Yes               | Yes                            | No        |
| 进度跟踪           | console/log messages | web page          | web page                       | Java API  |
| Hadoop job调度支持 | no                   | yes               | yes                            | yes       |
| 运行模式           | command line utility | daemon            | daemon                         | API       |
| Pig支持            | yes                  | yes               | yes                            | yes       |
| 事件通知           | no                   | no                | no                             | yes       |
| 需要安装           | no                   | yes               | yes                            | no        |
| 支持的hadoop版本   | 0.18+                | 0.20+             | currently unknown              | 0.18+     |
| 重试支持           | no                   | workflownode evel | yes                            | yes       |
| 运行任意命令       | yes                  | yes               | yes                            | yes       |
| Amazon EMR支持     | yes                  | no                | currently unknown              | yes       |

### 1.8 azkaban的架构

Azkaban由三个关键组件构成：

![](img\azkaban\azkaban架构.png)

1.  AzkabanWebServer：AzkabanWebServer是整个Azkaban工作流系统的主要管理者，它用户登录认证、负责project管理、定时执行工作流、跟踪工作流执行进度等一系列任务。
2.  AzkabanExecutorServer：负责具体的工作流的提交、执行，它们通过mysql数据库来协调任务的执行。
3. 关系型数据库（MySQL）：存储大部分执行流状态，AzkabanWebServer和AzkabanExecutorServer都需要访问数据库。

## 2. azkaban2.5.0安装部署

### 2.1 安装前准备

1.  将Azkaban Web服务器、Azkaban执行服务器、Azkaban的sql执行脚本及MySQL安装包拷贝到bigdata111虚拟机/opt/software目录下
2.  选择**Mysql**作为Azkaban数据库，因为Azkaban建立了一些Mysql连接增强功能，以方便Azkaban设置，并增强服务可靠性。

### 2.2 安装azkaban

1. 在/opt/module目录下创建azkaban目录

2. 解压三个压缩包至azkaban目录

   ```
   tar -zxvf azkaban-web-server-2.5.0.tar.gz -C /opt/module/azkaban/
   tar -zxvf azkaban-executor-server-2.5.0.tar.gz -C /opt/module/azkaban/
   tar -zxvf azkaban-sql-script-2.5.0.tar.gz -C /opt/module/azkaban/
   ```

3. 重命名

   ```
   mv azkaban-web-2.5.0/ server
   mv azkaban-executor-2.5.0/ executor
   ```

4. azkaban脚本导入

   进入mysql，创建azkaban数据库，并将解压的脚本导入到azkaban数据库。

   ```
   mysql> create database azkaban;
   mysql> use azkaban;
   mysql> source /opt/module/azkaban/azkaban-2.5.0/create-all-sql-2.5.0.sql
   ```

   注：source后跟.sql文件，用于批量处理.sql文件中的sql语句。

### 2.3 生成密钥库

Keytool是java数据证书的管理工具，使用户能够管理自己的公/私钥对及相关证书。

```
-keystore    指定密钥库的名称及位置(产生的各类信息将不在.keystore文件中)
-genkey      在用户主目录中创建一个默认文件".keystore" 
-alias  	对我们生成的.keystore 进行指认别名；如果没有默认是mykey
-keyalg  	指定密钥的算法 RSA/DSA 默认是DSA
```

1. 生成 keystore的密码及相应信息的密钥库

   ```
   [root@bigdata111 azkaban]keytool -keystore keystore -alias jetty -genkey -keyalg RSA
   
   输入密钥库口令:  000000
   再次输入新口令:  000000
   您的名字与姓氏是什么?
     [Unknown]:  
   您的组织单位名称是什么?
     [Unknown]:  
   您的组织名称是什么?
     [Unknown]:  
   您所在的城市或区域名称是什么?
     [Unknown]:  
   您所在的省/市/自治区名称是什么?
     [Unknown]:  
   该单位的双字母国家/地区代码是什么?
     [Unknown]:  
   CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown是否正确?
     [否]:  y
   
   输入 <jetty> 的密钥口令    000000
           (如果和密钥库口令相同, 按回车):  
   再次输入新口令:            000000
   ```

   **注意：**密钥库的密码至少必须6个字符，可以是纯数字或者字母或者数字和字母的组合等等

   密钥库的密码最好和<jetty> 的密钥相同，方便记忆

2. 将keystore 拷贝到 azkaban web服务器根目录中

   ```
   mv keystore /opt/module/azkaban/server/
   ```

### 2.4 时间同步配置

先配置好服务器节点上的时区

1. 如果在/usr/share/zoneinfo/这个目录下不存在时区配置文件Asia/Shanghai，就要用 tzselect 生成

   ```
   [root@bigdata111 azkaban]$ tzselect
   ```

   > Please identify a location so that time zone rules can be set correctly.
   > Please select a continent or ocean.
   >  1) Africa
   >  2) Americas
   >  3) Antarctica
   >  4) Arctic Ocean
   >  5) Asia
   >  6) Atlantic Ocean
   >  7) Australia
   >  8) Europe
   >  9) Indian Ocean
   > 10) Pacific Ocean
   > 11) none - I want to specify the time zone using the Posix TZ format.
   > **#? 5**
   > Please select a country.
   >  1) Afghanistan           18) Israel                35) Palestine
   >  2) Armenia               19) Japan                 36) Philippines
   >  3) Azerbaijan            20) Jordan                37) Qatar
   >  4) Bahrain               21) Kazakhstan            38) Russia
   >  5) Bangladesh            22) Korea (North)         39) Saudi Arabia
   >  6) Bhutan                23) Korea (South)         40) Singapore
   >  7) Brunei                24) Kuwait                41) Sri Lanka
   >  8) Cambodia              25) Kyrgyzstan            42) Syria
   >  9) China                 26) Laos                  43) Taiwan
   > 10) Cyprus                27) Lebanon               44) Tajikistan
   > 11) East Timor            28) Macau                 45) Thailand
   > 12) Georgia               29) Malaysia              46) Turkmenistan
   > 13) Hong Kong             30) Mongolia              47) United Arab Emirates
   > 14) India                 31) Myanmar (Burma)       48) Uzbekistan
   > 15) Indonesia             32) Nepal                 49) Vietnam
   > 16) Iran                  33) Oman                  50) Yemen
   > 17) Iraq                  34) Pakistan
   > **#? 9**
   > Please select one of the following time zone regions.
   > 1) Beijing Time
   > 2) Xinjiang Time
   > **#? 1**
   >
   > The following information has been given:
   >
   > ​    China
   > ​    Beijing Time
   >
   > Therefore TZ='Asia/Shanghai' will be used.
   > Local time is now:      Thu Oct 18 16:24:23 CST 2018.
   > Universal Time is now:  Thu Oct 18 08:24:23 UTC 2018.
   > Is the above information OK?
   > 1) Yes
   > 2) No
   > **#? 1**
   >
   > You can make this change permanent for yourself by appending the line
   >         TZ='Asia/Shanghai'; export TZ
   > to the file '.profile' in your home directory; then log out and log in again.
   >
   > Here is that TZ value again, this time on standard output so that you
   > can use the /usr/bin/tzselect command in shell scripts:
   > Asia/Shanghai

2. 拷贝该时区文件，覆盖系统本地时区配置

   ```
   [root@bigdata111 azkaban]$ cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime  
   ```

3. 集群时间同步（同时发给三个窗口）

   ```
   [root@bigdata111 azkaban]$ sudo date -s '2019-10-18 16:39:30'
   ```

### 2.5 配置文件

#### 2.5.1 web服务器配置

1. 进入azkaban web服务器安装目录 conf目录，打开azkaban.properties文件

   ```
   [root@bigdata111 conf]$ pwd
   /opt/module/azkaban/server/conf
   [root@bigdata111 conf]$ vim azkaban.properties
   ```

2. 按照如下配置修改azkaban.properties文件。

   > \#Azkaban Personalization Settings
   >
   > \#服务器UI名称,用于服务器上方显示的名字
   >
   > azkaban.name=Test
   >
   > \#描述
   >
   > azkaban.label=My Local Azkaban
   >
   > \#UI颜色
   >
   > azkaban.color=#FF3601
   >
   > azkaban.default.servlet.path=/index
   >
   > **\#默认web server存放web文件的目录**
   >
   > **web.resource.dir=/opt/module/azkaban/server/web/**
   >
   > **\#默认时区,已改为亚洲/上海 默认为美国**
   >
   > **default.timezone.id=Asia/Shanghai**
   >
   >  
   >
   > \#Azkaban UserManager class
   >
   > user.manager.class=azkaban.user.XmlUserManager
   >
   > **\#用户权限管理默认类（绝对路径）**
   >
   > **user.manager.xml.file=/opt/module/azkaban/server/conf/azkaban-users.xml**
   >
   > 
   >
   > \#Loader for projects
   >
   > **\#global配置文件所在位置（绝对路径）**
   >
   > **executor.global.properties=/opt/module/azkaban/executor/conf/global.properties**
   >
   > **azkaban.project.dir=projects**
   >
   >  
   >
   > \#数据库类型
   >
   > database.type=mysql
   >
   > \#端口号
   >
   > mysql.port=3306
   >
   > \#数据库连接IP
   >
   > **mysql.host=bigdata111**
   >
   > \#数据库实例名
   >
   > mysql.database=azkaban
   >
   > **\#数据库用户名**
   >
   > **mysql.user=root**
   >
   > **\#数据库密码**
   >
   > **mysql.password=000000**
   >
   > \#最大连接数
   >
   > mysql.numconnections=100
   >
   >  
   >
   > \# Velocity dev mode
   >
   > velocity.dev.mode=false
   >
   >  
   >
   > \# Azkaban Jetty server properties.
   >
   > \# Jetty服务器属性.
   >
   > \#最大线程数
   >
   > jetty.maxThreads=25
   >
   > \#Jetty SSL端口
   >
   > jetty.ssl.port=8443
   >
   > \#Jetty端口
   >
   > jetty.port=8081
   >
   > **\#SSL文件名（绝对路径）**
   >
   > **jetty.keystore=/opt/module/azkaban/server/keystore**
   >
   > **\#SSL文件密码**
   >
   > **jetty.password=000000**
   >
   > **\#Jetty主密码与keystore文件相同**
   >
   > **jetty.keypassword=000000**
   >
   > **\#SSL文件名（绝对路径）**
   >
   > **jetty.truststore=/opt/module/azkaban/server/keystore**
   >
   > **\#SSL文件密码**
   >
   > **jetty.trustpassword=000000**
   >
   >  
   >
   > \# Azkaban Executor settings
   >
   > executor.port=12321
   >
   >  
   >
   > \# mail settings
   >
   > mail.sender=
   >
   > mail.host=
   >
   > job.failure.email=
   >
   > job.success.email=
   >
   >  
   >
   > lockdown.create.projects=false
   >
   >  
   >
   > cache.directory=cache

3. Web服务器用户配置

   在azkaban web服务器安装目录 conf目录，按照如下配置修改azkaban-users.xml 文件，增加管理员用户。

   ```
   [root@bigdata111 conf]$ vim azkaban-users.xml
   ```

   > <azkaban-users>
   > 	<user username="azkaban" password="azkaban" roles="admin" groups="azkaban" />
   > 	<user username="metrics" password="metrics" roles="metrics"/>
   > 	**<user username="admin" password="admin" roles="admin,metrics" />**
   > 	<role name="admin" permissions="ADMIN" />
   > 	<role name="metrics" permissions="METRICS"/>
   > </azkaban-users>

#### 2.5.2 执行服务器配置

1. 进入执行服务器安装目录conf，打开azkaban.properties

   ```
   [root@bigdata111 conf]$ pwd
   /opt/module/azkaban/executor/conf
   [root@bigdata111 conf]$ vim azkaban.properties
   ```

2.  按照如下配置修改azkaban.properties文件。

   > \#Azkaban
   >
   > \#时区
   >
   > **default.timezone.id=Asia/Shanghai**
   >
   >  
   >
   > \# Azkaban JobTypes Plugins
   >
   > \#jobtype 插件所在位置
   >
   > azkaban.jobtype.plugin.dir=plugins/jobtypes
   >
   >  
   >
   > \#Loader for projects
   >
   > **executor.global.properties=/opt/module/azkaban/executor/conf/global.properties**
   >
   > azkaban.project.dir=projects
   >
   >  
   >
   > database.type=mysql
   >
   > mysql.port=3306
   >
   > **mysql.host=bigdata111**
   >
   > **mysql.database=azkaban**
   >
   > **mysql.user=root**
   >
   > **mysql.password=000000**
   >
   > mysql.numconnections=100
   >
   >  
   >
   > \# Azkaban Executor settings
   >
   > \#最大线程数
   >
   > executor.maxThreads=50
   >
   > \#端口号(如修改,请与web服务中一致)
   >
   > executor.port=12321
   >
   > \#线程数
   >
   > executor.flow.threads=30

### 2.6 启动executor服务器

在executor服务器目录下执行启动命令

```
[root@bigdata111 executor]$ pwd

/opt/module/azkaban/executor

[root@bigdata111 executor]$ bin/azkaban-executor-start.sh
```

### 2.7 启动web服务器

在azkaban web服务器目录下执行启动命令

```
[root@bigdata111 server]$ pwd
/opt/module/azkaban/server
[root@bigdata111 server]$ bin/azkaban-web-start.sh
```

**注意：**

先执行executor，再执行web，避免Web Server会因为找不到执行器启动失败。

jps查看进程

```
[root@bigdata111 server]$ jps
3601 AzkabanExecutorServer
5880 Jps
3661 AzkabanWebServer
```

启动完成后，在浏览器(建议使用谷歌浏览器)中输入**https://服务器IP地址:8443**，即可访问azkaban服务了。

在登录中输入刚才在azkaban-users.xml文件中新添加的户用名及密码,**即admin和admin**，点击 login。

## 3.azkaban3.51.0安装部署

### 3.1 azkaban 3.x版本三种部署模式

1. solo server mode

   该模式中webServer和executorServer运行在同一个进程中，进程名是AzkabanSingleServer。使用**自带的H2数据库**。这种模式包含Azkaban的所有特性，但一般用来**学习和测试**。

2. two-server mode

   该模式使用MySQL数据库， Web Server和Executor Server运行在**不同的进程中。**

3. multiple-executor mode

   该模式使用MySQL数据库， Web Server和Executor Server运行在**不同的机器**中。且有**多个Executor Server**。该模式**适用于大规模**应用。

### 3.2 安装maven

1. 上传maven安装包至software目录，并解压至module目录

2. 更名

   ```
   mv apache-maven-3.6.3/ maven-3.6.3
   ```

3. 配置环境变量

   vim /etc/profile

   ```
   #maven
   MAVEN_HOME=/opt/module/maven-3.6.3
   PATH=$PATH:$MAVEN_HOME/bin
   ```

4. 重启配置文件

   source /etc/profile

5. 测试

   ```
   mvn -v
   ```

### 3.3 安装ant

1. 下载apache-ant-1.9.14-bin.tar.gz并上传至software目录，然后解压至module目录

2. 更名

   ```
   mv apache-ant-1.9.14/ ant-1.9.14
   ```

3. 配置环境变量

   ```
   #ant
   export ANT_HOME=/opt/module/ant-1.9.14
   export PATH=$PATH:$ANT_HOME/bin
   ```

4. 重启配置文件

   source /etc/profile

5. 检查

   ```
   ant -v
   ```

### 3.4 安装mysql

之前已经安装过，这里不再重复

1. 创建azkaban的数据库

   ```
   mysql> create database azkaban;
   ```

2. 设置mysql的信息包大小，默认的太小，改大一点(允许上传较大压缩包)

   修改/etc/my.cnf配置文件

   ```
   [mysqld]
   ...
   max_allowed_packet=1024M
   ```

3. 重启mysql

   ```
   service mysqld restart
   ```

   

### 3.5 安装node

1. 下载并上传安装包（node-8.9.1）至software，解压至module目录

   ```
   tar -zxvf node-v8.9.1-linux-x64_.tar.gz -C /opt/module/
   ```

2. 配置环境变量

   ```
   #node
   export NODE_HOME=/opt/module/node-8.9.1
   export PATH=$PATH:$NODE_HOME/bin
   ```

3. 重启配置文件

   ```
   source /etc/profile
   ```

4. 检查

   ```
   [root@bigdata222 module]# node -v
   v8.9.1
   [root@bigdata222 module]# npm -v
   5.5.1
   ```

### 3.6 azkaban 源码编译

Azkaban3.x在安装前需要自己编译成二进制包。

并且提前安装好Maven、Ant、Node等软件。

1. 编译环境

   ```
   yum install –y git
   yum install –y gcc-c++
   yum -y install wget
   ```

2. 下载源码解压

   ```
   wget https://github.com/azkaban/azkaban/archive/3.51.0.tar.gz
   tar -zxvf 3.51.0.tar.gz 
   ```

3. 编译源码

   先进入azkaban根目录，然后执行命令

   ```
   ./gradlew build installDist -x test
   ```

   Gradle是一个基于Apache Ant和Apache Maven的项目自动化构建工具。-x test 跳过测试。（注意联网下载jar可能会失败、慢）

4. 编译后安装包路径

   编译成功之后就可以在指定的路径下取得对应的安装包了。

   ```
   azkaban-db-0.1.0-SNAPSHOT.tar.gz
   azkaban-exec-server-0.1.0-SNAPSHOT.tar.gz
   azkaban-solo-server-0.1.0-SNAPSHOT.tar.gz
   azkaban-web-server-0.1.0-SNAPSHOT.tar.gz
   ```

5. 分别解压上述安装包到目标目录

6. 导入sql

   ```
   use azkaban;
   source /opt/module/azkaban-3.51.0/azkaban-db-0.1.0-SNAPSHOT/create-all-sql-0.1.0-SNAPSHOT.sql;
   ```

7. 生成keystore

   进入/opt/module/azkaban-3.51.0/zkaban-web-server-0.1.0-SNAPSHOT目录

   执行下面的命令生成密钥

   ```
   keytool -keystore keystore -alias jetty -genkey -keyalg RSA
   ```

   > [root@bigdata222 azkaban-web-server-0.1.0-SNAPSHOT]# keytool -keystore keystore -alias jetty -genkey -keyalg RSA
   > Enter keystore password:  [root@bigdata222 azkaban-web-server-0.1.0-SNAPSHOT]# keytool -keystore keystore -alias jetty -genkey -keyalg RSA
   > Enter keystore password:  
   > Re-enter new password: 
   > What is your first and last name?
   >   [Unknown]:  
   > What is the name of your organizational unit?
   >   [Unknown]:  
   > What is the name of your organization?
   >   [Unknown]:  
   > What is the name of your City or Locality?
   >   [Unknown]:  
   > What is the name of your State or Province?
   >   [Unknown]:  
   > What is the two-letter country code for this unit?
   >   [Unknown]:  
   > Is CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown correct?
   >
   > [no]:  y
   >
   > Enter key password for <jetty>
   > 	(RETURN if same as keystore password):  
   > Re-enter new password: 

   密码统一皆为：000000

8. 配置conf/azkaban.properties

   如果在azkaban-web-server-0.1.0-SNAPSHOT下没有conf目录，可以从azkaban-solo-server-0.1.0-SNAPSHOT下把conf目录拷贝过来。

### 3.7 solo-server模式部署

#### 3.7.1 节点规划

| HOST       | 角色                                |
| ---------- | ----------------------------------- |
| bigdata222 | Web Server和Executor Server同一进程 |

#### 3.7.2 配置

1. 解压至目标目录

   ```
   mkdir /export/servers/azkaban
   tar -zxvf azkaban-solo-server-0.1.0-SNAPSHOT.tar.gz –C /opt/module/azkaban-3.51.0/
   ```

2. 编写配置文件

   ```
   vim conf/azkaban.properties
   default.timezone.id=Asia/Shanghai #修改时区
   ```

   ```
   vim  plugins/jobtypes/commonprivate.properties
   添加：memCheck.enabled=false
   ```

3. 启动验证

   ```
   cd azkaban-solo-server-0.1.0-SNAPSHOT/
   bin/start-solo.sh
   ```

   注:启动/关闭必须进到azkaban-solo-server-0.1.0-SNAPSHOT/目录下

   AzkabanSingleServer(对于Azkaban solo‐server模式，Exec Server和Web Server在同一个进程中)

4. 登录web页面

   访问Web Server=>http://bigdata222:8081/ 默认用户名密码azkaban

注意：azkaban默认需要3G的内存，剩余内存不足则会报异常

### 3.8 two-server模式部署

1. 节点规划

   | HOST       | 角色                            |
   | ---------- | ------------------------------- |
   | bigdata222 | MySQL                           |
   | bigdata222 | web‐server和exec‐server不同进程 |

   条件有限，就在同一台机器上布置，

   two-server模式可将mysql配置在其他服务器上，只需要在配置文件中更改相关配置即可

2. mysql配置初始化

   - 解压 azkaban-db-0.1.0-SNAPSHOT.tar.gz 安装包

     ```
     tar -zxvf azkaban-db-0.1.0-SNAPSHOT.tar.gz –C /opt/module/azkaban-3.51.0/
     ```

   - Mysql上创建对应的库、增加权限、创建表

     ```
     CREATE DATABASE azkaban;
     
     use azkaban;
     
     /opt/module/azkaban-3.51.0/azkaban-db-0.1.0-SNAPSHOT/create-all-sql-0.1.0-SNAPSHOT.sql; 
     ```

3. web-server服务器配置

   - 解压azkaban-web-server-0.1.0-SNAPSHOT.tar.gz 和 azkaban-exec-server-0.1.0-SNAPSHOT.tar.gz 到目标目录

     ```
     tar -zxvf azkaban-web-server-0.1.0-SNAPSHOT.tar.gz –C opt/module/azkaban-3.51.0/
     tar -zxvf azkaban-exec-server-0.1.0-SNAPSHOT.tar.gz –C opt/module/azkaban-3.51.0/
     ```

   - 生成ssl证书：

     在web服务器根目录下执行

     ```
     keytool -keystore keystore -alias jetty -genkey -keyalg RSA
     ```

     运行此命令后,会提示输入当前生成keystore的密码及相应信息,输入的密码请记住(**所有密码统一以000000输入**)。

   - 配置conf/azkaban.properties

     > #Azkaban Personalization Settings
     >
     > azkaban.name=Test
     > azkaban.label=My Local Azkaban
     > azkaban.color=#FF3601
     > azkaban.default.servlet.path=/index
     > **web.resource.dir=/opt/module/azkaban-3.51.0/azkaban-web-server-0.1.0-SNAPSHOT/web**
     > **default.timezone.id=Asia/Shanghai**
     >
     > #Azkaban UserManager class
     >
     > user.manager.class=azkaban.user.XmlUserManager
     > **user.manager.xml.file=/opt/module/azkaban-3.51.0/azkaban-exec-server-0.1.0-SNAPSHOT/conf/azkaban-users.xml**
     >
     > #Loader for projects
     >
     > **executor.global.properties=/opt/module/azkaban-3.51.0/azkaban-exec-server-0.1.0-SNAPSHOT/conf/global.properties**
     > azkaban.project.dir=projects
     >
     > #Velocity dev mode
     >
     > velocity.dev.mode=false
     >
     > #Azkaban Jetty server properties.
     >
     > jetty.use.ssl=true
     > jetty.maxThreads=25
     > jetty.port=8443
     > #SSL文件名（绝对路径）
     > **jetty.keystore=/opt/module/azkaban-3.51.0/azkaban-web-server-0.1.0-SNAPSHOT/keystore**
     > #SSL文件密码
     > **jetty.password=000000**
     > #Jetty主密码与keystore文件相同
     > **jetty.keypassword=000000**
     > #SSL文件名（绝对路径）
     > **jetty.truststore=/opt/module/azkaban-3.51.0/azkaban-web-server-0.1.0-SNAPSHOT/keystore**
     > #SSL文件密码
     > **jetty.trustpassword=000000**
     >
     > #Azkaban Executor settings
     >
     > **executor.host=bigdata222**
     > **executor.port=12321**
     >
     > #mail settings
     >
     > mail.sender=
     > mail.host=
     >
     > #User facing web server configurations used to construct the user facing server URLs. They are useful when there is a reverse proxy between Azkaban web servers and users.
     >
     > #enduser -> myazkabanhost:443 -> proxy -> localhost:8081
     >
     > #when this parameters set then these parameters are used to generate email links.
     >
     > #if these parameters are not set then jetty.hostname, and jetty.port(if ssl configured jetty.ssl.port) are used.
     >
     > #azkaban.webserver.external_hostname=myazkabanhost.com
     >
     > #azkaban.webserver.external_ssl_port=443
     >
     > #azkaban.webserver.external_port=8081
     >
     > job.failure.email=
     > job.success.email=
     > lockdown.create.projects=false
     > cache.directory=cache
     >
     > #JMX stats
     >
     > jetty.connector.stats=true
     > executor.connector.stats=true
     >
     > #Azkaban mysql settings by default. Users should configure their own username and password.
     >
     > **database.type=mysql**
     > **mysql.port=3306**
     > **mysql.host=bigdata222**
     > **mysql.database=azkaban**
     > **mysql.user=root**
     > **mysql.password=000000**
     > mysql.numconnections=100
     > #Multiple Executor
     > azkaban.use.multiple.executors=true
     > azkaban.executorselector.filters=StaticRemainingFlowSize,CpuStatus
     > azkaban.executorselector.comparator.NumberOfAssignedFlowComparator=1
     > azkaban.executorselector.comparator.Memory=1
     > azkaban.executorselector.comparator.LastDispatched=1
     > azkaban.executorselector.comparator.CpuUsage=1

   - 添加azkaban.native.lib=false 和 execute.as.user=false属性：

     ```
     mkdir -p plugins/jobtypes
     vim commonprivate.properties
     
     azkaban.native.lib=false
     execute.as.user=false
     memCheck.enabled=false
     ```

     

4. exec-server 服务器配置

   配置conf/azkaban.properties：

   > #Azkaban Personalization Settings
   >
   > azkaban.name=Test
   > azkaban.label=My Local Azkaban
   > azkaban.color=#FF3601
   > azkaban.default.servlet.path=/index
   > web.resource.dir=web/
   > **default.timezone.id=Asia/Shanghai**
   >
   > #Azkaban UserManager class
   >
   > user.manager.class=azkaban.user.XmlUserManager
   > user.manager.xml.file=conf/azkaban-users.xml
   >
   > #Loader for projects
   >
   > **executor.global.properties=/opt/module/azkaban-3.51.0/azkaban-exec-server-0.1.0-SNAPSHOT/conf/global.properties**
   > azkaban.project.dir=projects
   >
   > #Velocity dev mode
   >
   > velocity.dev.mode=false
   >
   > #Azkaban Jetty server properties.
   >
   > jetty.use.ssl=false
   > jetty.maxThreads=25
   > jetty.port=8081
   >
   > #Where the Azkaban web server is located
   >
   > **azkaban.webserver.url=https://bigdata222:8443**
   >
   > #mail settings
   >
   > mail.sender=
   > mail.host=
   >
   > #User facing web server configurations used to construct the user facing server URLs. They are useful when there is a reverse proxy between Azkaban web servers and users.
   >
   > enduser -> myazkabanhost:443 -> proxy -> localhost:8081
   >
   > #when this parameters set then these parameters are used to generate email links.
   >
   > #if these parameters are not set then jetty.hostname, and jetty.port(if ssl configured jetty.ssl.port) are used.
   >
   > #azkaban.webserver.external_hostname=myazkabanhost.com
   >
   > #azkaban.webserver.external_ssl_port=443
   >
   > #azkaban.webserver.external_port=8081
   >
   > #enduser -> myazkabanhost:443 -> proxy -> localhost:8081
   >
   > #when this parameters set then these parameters are used to generate email links.
   >
   > #if these parameters are not set then jetty.hostname, and jetty.port(if ssl configured jetty.ssl.port) are used.
   >
   > #azkaban.webserver.external_hostname=myazkabanhost.com
   >
   > #azkaban.webserver.external_ssl_port=443
   >
   > #azkaban.webserver.external_port=8081
   >
   > job.failure.email=
   > job.success.email=
   > lockdown.create.projects=false
   > cache.directory=cache
   >
   > #JMX stats
   >
   > jetty.connector.stats=true
   > executor.connector.stats=true
   >
   > #Azkaban plugin settings
   >
   > azkaban.jobtype.plugin.dir=plugins/jobtypes
   >
   > #Azkaban mysql settings by default. Users should configure their own username and password.
   >
   > **database.type=mysql**
   > **mysql.port=3306**
   > **mysql.host=bigdata222**
   > **mysql.database=azkaban**
   > **mysql.user=root**
   > **mysql.password=000000**
   > mysql.numconnections=100
   >
   > #Azkaban Executor settings
   >
   > executor.maxThreads=50
   > executor.port=12321
   > executor.flow.threads=30

5. 集群启动

   - 先启动exec-server
   - 再启动web-server

   启动webServer之后进程失败消失，可通过安装包根目录下对应启动日志进行排查。

   ```
   Caused by: azkaban.executor.ExecutorManagerException: No active executor found
   	at azkaban.executor.ExecutorManager.setupExecutors(ExecutorManager.java:253)
   	at azkaban.executor.ExecutorManager.<init>(ExecutorManager.java:131)
   	at azkaban.executor.ExecutorManager$$FastClassByGuice$$e1c1dfed.newInstance(<generated>)
   ```

   需要手动激活executor

   ```
   cd  /opt/module/azkaban-3.51.0/azkaban-exec-server-0.1.0-SNAPSHOT
   
   curl -G "bigdata222:$(<./executor.port)/executor?action=activate" && echo
   
   [root@bigdata222 azkaban-exec-server-0.1.0-SNAPSHOT]# curl -G "bigdata222:$(<./executor.port)/executor?action=activate" && echo
   {"status":"success"}
   ```

   然后重新启动webServer就可以了

### 3.9 multiple-executor模式部署

multiple-executor模式是多个executor Server分布在不同服务器上，只需要将azkaban-exec-server安装包拷贝到不同机器上即可组成分布式。

#### 3.9.1 节点规划

| HOST       | 角色                    |
| ---------- | ----------------------- |
| bigdata222 | mysql                   |
| bigdata222 | web-server、exec-server |
| bigdata333 | exec-server             |

#### 3.9.2 scp executor server安装包到bigdata333

```
scp -r azkaban-exec-server-0.1.0-SNAPSHOT root@bigdata333:/opt/module/
```

#### 3.9.4 启动azkaban

启动之后，需要手动激活executor

```
cd  /opt/module/azkaban-3.51.0/azkaban-exec-server-0.1.0-SNAPSHOT

curl -G "bigdata333:$(<./executor.port)/executor?action=activate" && echo
```



## 4.azkaban实战

Azkaba内置的任务类型支持command、java

### 4.1 单一job案例

1. 创建job描述文件

   vim first.job

   ```
   #first.job
   type=command
   command=echo 'this is my first job'
   ```

2. 将job资源文件打包成zip文件

   ```
   [root@bigdata111 jobs]$ zip first.zip first.job 
     adding: first.job (deflated 15%)
   [root@bigdata111 jobs]$ ll
   总用量 8
   -rw-rw-r--. 1 itstar itstar  60 10月 18 17:42 first.job
   -rw-rw-r--. 1 itstar itstar 219 10月 18 17:43 first.zip
   ```

   注意：

   目前，Azkaban上传的工作流文件**只支持xxx.zip文件**。zip应包含xxx.job运行作业所需的文件和任何文件（文件名后缀必须以.job结尾，否则无法识别）。作业名称在项目中必须是唯一的。

3. 通过azkaban的web管理平台创建project并上传job的zip包

   首先创建project

   ![](img\azkaban\创建job.png)

   上传zip包

   ![](img\azkaban\上传zip包.png)

   启动执行该job

   ![](img\azkaban\启动job.png)

   点击执行工作流

   ![](img\azkaban\执行.png)

   点击继续

   ![](img\azkaban\继续.png)

   Job执行成功

   ![](img\azkaban\执行成功.png)

   点击查看job日志

   ![](img\azkaban\查看日志.png)

### 4.2 多job工作流案例

1. 创建有依赖关系的多个job描述

   第一个job：1.job

   ```
   [root@bigdata111 jobs]$ vi 1.job
   type=command
   command=/opt/module/hadoop-2.8.4/bin/hadoop fs -put /opt/module/datas/word.txt /
   ```

   第二个job：2.job依赖1.job

   ```
   [root@bigdata111 jobs]$ vi 2.job
   type=command
   command=/opt/module/hadoop-2.8.4/bin/hadoop jar /opt/module/hadoop-2.8.4/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.4.jar wordcount /word.txt /out
   dependencies=1
   ```

2. 注意：将所有job资源文件打到一个zip包中

3. 在azkaban的web管理界面创建工程并上传zip包

4. 查看结果

**思考：**

将student.txt文件上传到hdfs，根据所传文件创建外部表，再将表中查询到的结果写入到本地文件

### 4.3 java操作任务

使用Azkaban调度java程序

1. 编写java程序

   ```java
   import java.io.FileOutputStream;
   import java.io.IOException;
   
   
   public class AzkabanTest {
   public void run() throws IOException {
   // 根据需求编写具体代码
   FileOutputStream fos = new FileOutputStream("/opt/module/azkaban/output.txt");
   fos.write("this is a java progress".getBytes());
   fos.close();
   }
   
   
   public static void main(String[] args) throws IOException {
   AzkabanTest azkabanTest = new AzkabanTest();
   azkabanTest.run();
   }
   }
   ```

2. 将java程序打成jar包，创建lib目录，将jar放入lib内

   自行在azkaban下创建lib目录

3. 编写job文件

   [root@bigdata111 jobs]$ vim azkabanJava.job

   ```
   #azkabanJava.job
   type=javaprocess
   java.class=AzkabanTest(全类名)
   classpath=/opt/module/azkaban/lib/*
   ```

4. 将job文件打成zip包

   ```
   [root@bigdata111 jobs]$ zip azkabanJava.zip azkabanJava.job 
   adding: azkabanJava.job (deflated 19%)
   ```

5. 通过azkaban的web管理平台创建project并上传job压缩包，启动执行该job

   查看生成的文件：

   ```
   [root@bigdata111 azkaban]$ cat /opt/module/azkaban/output.txt 
   ```

### 3.4 HDFS操作任务

1. 创建job描述文件

   [root@bigdata111 jobs]$ vi hdfs.job

   ```
   #hdfs job
   type=command
   command=/opt/module/hadoop-2.8.4/bin/hadoop fs -mkdir /azkaban
   ```

2. 将job资源文件打包成zip文件

   ```
   [root@bigdata111 jobs]$ zip fs.zip fs.job 
    adding: fs.job (deflated 12%)
   ```

3. 通过azkaban的web管理平台创建project并上传job压缩包

4. 启动执行该job

5. 查看结果

### 4.5 mapreduce任务

mapreduce任务依然可以使用azkaban进行调度

1. 创建job描述文件，及mr程序jar包

   [root@bigdata111 jobs]$ vim mapreduce.job

   ```
   #mapreduce job
   type=command
   command=/opt/module/hadoop-2.8.4/bin/hadoop jar /opt/module/hadoop-2.8.4/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.4.jar wordcount /wordcount/input /wordcount/output
   ```

2. 将所有job资源文件打到一个zip包中

   ```
   [root@bigdata111 jobs]$ zip mapreduce.zip mapreduce.job 
   adding: mapreduce.job (deflated 43%)
   ```

3. 在azkaban的web管理界面创建工程并上传zip包

4. 启动job

5. 查看结果

### 4.6 Hive脚本任务

1. 创建job描述文件和hive脚本

   - Hive脚本：student.sql

     [root@bigdata111 jobs]$ vim student.sql

     ```
     use default;
     drop table student;
     create table student(id int, name string)
     row format delimited fields terminated by '\t';
     load data local inpath '/opt/module/datas/student.txt' into table student;
     insert overwrite local directory '/opt/module/datas/student'
     row format delimited fields terminated by '\t'
     select * from student;
     ```

   - Job描述文件：hive.job

     [root@bigdata111 jobs]$ vim hive.job

     ```
     #hive job
     type=command
     command=/opt/module/hive/bin/hive -f /opt/module/azkaban/jobs/student.sql
     ```

2. 将所有job资源文件打到一个zip包中

3. 在azkaban的web管理界面创建工程并上传zip包

4. 启动job

5. 查看结果

   ```
   [root@bigdata111 student]$ cat /opt/module/datas/student/000000_0 
   ```

   