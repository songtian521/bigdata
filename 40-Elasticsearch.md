# 40-Elasticsearch

# 1.概述

## 1.1 什么是搜索

百度：我们比如说想找寻任何的信息的时候，就会上百度去搜索一下，比如说找一部自己喜欢的电影，或者说找一本喜欢的书，或者找一条感兴趣的新闻（提到搜索的第一印象）。百度 != 搜索

1. 互联网的搜索：电商网站，招聘网站，新闻网站，各种app
2. IT系统的搜索：OA软件，办公自动化软件，会议管理，日程管理，项目管理。

**搜索**，就是在任何场景下，找寻你想要的信息，这个时候，会输入一段你要搜索的关键字，然后就期望找到这个关键字相关的有些信息

## 1.2 全文检索和Lucene

1. 全文检索，倒排索引

   - 全文检索是指计算机索引程序通过扫描文章中的每一个词，对每一个词建立一个索引，指明该词在文章中出现的次数和位置，当用户查询时，检索程序就根据事先建立的索引进行查找，并将查找的结果反馈给用户的检索方式。这个过程类似于通过字典中的检索字表查字的过程。全文搜索引擎数据库中的数据。

   - 倒排索引

     传统数据库存储：

     | id   | 描述         |
     | ---- | ------------ |
     | 1    | 优秀员工     |
     | 2    | 销售冠军     |
     | 3    | 优秀团队领导 |
     | 4    | 优秀项目     |

     倒排索引处理步骤：

     1. 切词：

        优秀员工 —— 优秀 员工

        销售冠军 —— 销售 冠军

        优秀团队领导 —— 优秀 团队 领导

        优秀项目 —— 优秀 项目

     2. 建立倒排索引：

        关键词 id

        | 关键词 | id     |
        | ------ | ------ |
        | 优秀   | 1,3,4  |
        | 员工   | 1      |
        | 销售   | 2      |
        | 团队   | 3      |
        | 。。。 | 。。。 |

        ![](img\Elasticsearch\倒排索引原理.png)

2. lucene

   就是一个jar包，里面包含了封装好的各种建立倒排索引，以及进行搜索的代码，包括各种算法。我们就用java开发的时候，引入lucene jar，然后基于lucene的api进行去进行开发就可以了。

## 1.3 什么是Elasticsearch

Elasticsearch，基于lucene，隐藏复杂性，提供简单易用的restful api接口、java api接口（还有其他语言的api接口）。

关于elasticsearch的一个传说，有一个程序员失业了，陪着自己老婆去英国伦敦学习厨师课程。程序员在失业期间想给老婆写一个菜谱搜索引擎，觉得lucene实在太复杂了，就开发了一个封装了lucene的开源项目，compass。后来程序员找到了工作，是做分布式的高性能项目的，觉得compass不够，就写了elasticsearch，让lucene变成分布式的系统。

**Elasticsearch是一个实时分布式搜索和分析引擎。它用于全文搜索、结构化搜索、分析。**

- 全文检索

  将非结构化数据中的一部分信息提取出来,重新组织,使其变得有一定结构,然后对此有一定结构的数据进行搜索,从而达到搜索相对较快的目的。

- 结构化检索

  我想搜索商品分类为日化用品的商品都有哪些，select * from products where category_id='日化用品'。

- 数据分析

  电商网站，最近7天牙膏这种商品销量排名前10的商家有哪些；新闻网站，最近1个月访问量排名前3的新闻版块是哪些。

## 1.4 Elasticsearch适用场景

1. 维基百科，类似百度百科，牙膏，牙膏的维基百科，全文检索，高亮，搜索推荐。
2. The Guardian（国外新闻网站），类似搜狐新闻，用户行为日志（点击，浏览，收藏，评论）+ 社交网络数据（对某某新闻的相关看法），数据分析，给到每篇新闻文章的作者，让他知道他的文章的公众反馈（好，坏，热门，垃圾，鄙视，崇拜）。
3. Stack Overflow（国外的程序异常讨论论坛），IT问题，程序的报错，提交上去，有人会跟你讨论和回答，全文检索，搜索相关问题和答案，程序报错了，就会将报错信息粘贴到里面去，搜索有没有对应的答案。
4. GitHub（开源代码管理），搜索上千亿行代码。
5. 国内：站内搜索（电商，招聘，门户，等等），IT系统搜索（OA，CRM，ERP，等等），数据分析（ES热门的一个使用场景）。

## 1.5 Elasticsearch特点

1. 可以作为一个大型分布式集群（数百台服务器）技术，处理PB级数据，服务大公司；也可以运行在单机上，服务小公司；树莓派
2. Elasticsearch不是什么新技术，主要是将全文检索、数据分析以及分布式技术，合并在了一起，才形成了独一无二的ES；lucene（全文检索），商用的数据分析软件（也是有的），分布式数据库（mycat）；
3. 对用户而言，是开箱即用的，非常简单，作为中小型的应用，直接3分钟部署一下ES，就可以作为生产环境的系统来使用了，数据量不大，操作不是太复杂；
4. 数据库的功能面对很多领域是不够用的（事务，还有各种联机事务型的操作）；特殊的功能，比如全文检索，同义词处理，相关度排名，复杂数据分析，海量数据的近实时处理；Elasticsearch作为传统数据库的一个补充，提供了数据库所不能提供的很多功能。

## 1.6 Elasticsearch核心概念

1. 近实时

   近实时，两个意思，从写入数据到数据可以被搜索到有一个小延迟（大概1秒）；基于es执行搜索和分析可以达到秒级。

2. Cluster（集群）
   集群包含多个节点，每个节点属于哪个集群是通过一个配置（集群名称，默认是elasticsearch）来决定的，对于中小型应用来说，刚开始一个集群就一个节点很正常。

3. Node（节点）
   集群中的一个节点，节点也有一个名称（默认是随机分配的），节点名称很重要（在执行运维管理操作的时候），默认节点会去加入一个名称为“elasticsearch”的集群，如果直接启动一堆节点，那么它们会自动组成一个elasticsearch集群，当然一个节点也可以组成一个elasticsearch集群。

4.  Index（索引-数据库）
   索引包含一堆有相似结构的文档数据，比如可以有一个客户索引，商品分类索引，订单索引，索引有一个名称。一个index包含很多document，一个index就代表了一类类似的或者相同的document。比如说建立一个product index，商品索引，里面可能就存放了所有的商品数据，所有的商品document。

5. Type（类型-表）
   每个索引里都可以有一个或多个type，type是index中的一个逻辑数据分类，一个type下的document，都有相同的field，比如博客系统，有一个索引，可以定义用户数据type，博客数据type，评论数据type。
   商品index，里面存放了所有的商品数据，商品document
   但是商品分很多种类，每个种类的document的field可能不太一样，比如说电器商品，可能还包含一些诸如售后时间范围这样的特殊field；生鲜商品，还包含一些诸如生鲜保质期之类的特殊field
   type，日化商品type，电器商品type，生鲜商品type
   日化商品type：product_id，product_name，product_desc，category_id，category_name
   电器商品type：product_id，product_name，product_desc，category_id，category_name，service_period
   生鲜商品type：product_id，product_name，product_desc，category_id，category_name，eat_period

   - 每一个type里面，都会包含一堆document

     ```json
     {
       "product_id": "1",
       "product_name": "长虹电视机",
       "product_desc": "4k高清",
       "category_id": "3",
       "category_name": "电器",
       "service_period": "1年"
     }
     {
       "product_id": "2",
       "product_name": "基围虾",
       "product_desc": "纯天然，冰岛产",
       "category_id": "4",
       "category_name": "生鲜",
       "eat_period": "7天"
     }
     ```

6. Document（文档-行）
   文档是es中的最小数据单元，一个document可以是一条客户数据，一条商品分类数据，一条订单数据，通常用JSON数据结构表示，每个index下的type中，都可以去存储多个document。

7. Field（字段-列）
   Field是Elasticsearch的最小单位。一个document里面有多个field，每个field就是一个数据字段。

   ```json
   product document
   {
     "product_id": "1",
     "product_name": "高露洁牙膏",
     "product_desc": "高效美白",
     "category_id": "2",
     "category_name": "日化用品"
   }
   ```

8. mapping（映射-约束）

   数据如何存放到索引对象上，需要有一个映射配置，包括：数据类型、是否存储、是否分词等。
   这样就创建了一个名为blog的Index。Type不用单独创建，在创建Mapping 时指定就可以。Mapping用来定义Document中每个字段的类型，即所使用的 analyzer、是否索引等属性，非常关键等。创建Mapping 的代码示例如下：

   ```json
   client.indices.putMapping({
       index : 'blog',
       type : 'article',
       body : {
           article: {
               properties: {
                   id: {
                       type: 'string',
                       analyzer: 'ik',
                       store: 'yes',
                   },
                   title: {
                       type: 'string',
                       analyzer: 'ik',
                       store: 'no',
                   },
                   content: {
                       type: 'string',
                       analyzer: 'ik',
                       store: 'yes',
                   }
               }
           }
       }
   });
   ```

9. elasticsearch与数据库的类比

   | 关系型数据库（比如Mysql） | 非关系型数据库（Elasticsearch） |
   | ------------------------- | ------------------------------- |
   | 数据库Database            | 索引Index                       |
   | 表Table                   | 类型Type                        |
   | 数据行Row                 | 文档Document                    |
   | 数据列Column              | 字段Field                       |
   | 约束 Schema               | 映射Mapping                     |

10. ES存入数据和搜索数据机制

    ![](img\Elasticsearch\ES存入数据和搜索数据机制.png)

    1. 索引对象（index）：存储数据的表结构 ，任何搜索数据，存放在索引对象上 。
    2. 映射（mapping）：数据如何存放到索引对象上，需要有一个映射配置， 包括：数据类型、是否存储、是否分词等。
    3. 文档（document）：一条数据记录，存在索引对象上 。
    4. 文档类型（type）：一个索引对象，存放多种类型数据，数据用文档类型进行标识。

# 2.Elasticsearch安装

## 2.1 单节点安装

1. 解压Elasticsearch至module目录下

   ```
    tar -zxvf elasticsearch-5.6.1.tar.gz -C /opt/module/
   ```

2. 在/opt/module/elasticsearch-5.6.1路径下创建data和logs文件夹

   ```
   mkdir data
   mkdir logs
   ```

3. 修改配置文件config/elasticsearch.yml

   ```
   # ------------- Cluster -------------------------------------
   cluster.name: my-application
   # ------------------------------------ Node ----------------
   node.name: bigdata111
   # ----------------------------------- Paths -------------------
   path.data: /opt/module/elasticsearch-5.6.1/data
   path.logs: /opt/module/elasticsearch-5.6.1/logs
   # ----------------------------------- Memory -------------------
   bootstrap.memory_lock: false
   bootstrap.system_call_filter: false  // 此行新增
   # ---------------------------------- Network -------------------
   network.host: 192.168.64.129
   # --------------------------------- Discovery ----------------
   discovery.zen.ping.unicast.hosts: ["bigdata111"]
   ```

   说明：

   - cluster.name

     如果要配置集群需要两个节点上的elasticsearch配置的cluster.name相同，都启动可以自动组成集群，这里如果不改cluster.name则默认是cluster.name=my-application，

   - nodename随意取但是集群内的各节点不能相同

   - 修改后的每行前面不能有空格，修改后的“：”后面必须有一个空格

4. 配置linux系统环境

   - 编辑`/etc/security/limits.conf`

     添加如下内容

     ```
     * soft nofile 65536
     * hard nofile 131072
     * soft nproc 4096
     * hard nproc 4096
     ```

   - 编辑`/etc/security/limits.d/90-nproc.conf`

     修改如下

     ```
     * soft nproc 1024
     #修改为
     * soft nproc 4096
     ```

   - 编辑`/etc/sysctl.conf `

     添加如下配置

     ```
     vm.max_map_count=655360
     ```

     执行命令：` sudo sysctl -p`

5. 启动elasticsearch

   ```
   bin/elasticsearch
   ```

   - 注：报错1，无法在root用户下运行elasticsearch

     解决方法：添加用户，依次执行以下代码添加用户

     ```
     adduser elasticsearch
     passwd 000000
     chown -R  用户名 elasticsearch目录名
     切换用户 su elasticsearch
     ```

   - 注：报错2，在单机模式启动一遍以后，配置完多节点模式后再启动会报错

     原因：data目录下每个节点的data目录下依旧留存master节点的数据

     解决方法：清除每个节点下的elasticsearch目录内data下的所有数据即可

6. 测试

   ```
   [root@bigdata111 ~]# curl http://bigdata111:9200
   {
     "name" : "bigdata111",
     "cluster_name" : "my-application",
     "cluster_uuid" : "x2NEekVLSpy_yppIg3osVQ",
     "version" : {
       "number" : "6.1.1",
       "build_hash" : "bd92e7f",
       "build_date" : "2017-12-17T20:23:25.338Z",
       "build_snapshot" : false,
       "lucene_version" : "7.1.0",
       "minimum_wire_compatibility_version" : "5.6.0",
       "minimum_index_compatibility_version" : "5.0.0"
     },
     "tagline" : "You Know, for Search"
   }
   ```

7. 安装`elasticsearch-head.crx`插件

   直接去谷歌应用商店搜索安装即可

## 2.2 多节点安装elasticsearch

1. 分发elasticsearch至其他节点

   ```
   scp -r elasticsearch-6.1.1/ root@bigdata222:/opt/module
   scp -r elasticsearch-6.1.1/ root@bigdata333:/opt/module
   ```

2. 修改elasticsearch.yml配置信息

   - 每台节点分别添加如下内容

     ```
     http.cors.enabled: true
     http.cors.allow-origin: "*"
     node.master: true
     node.data: true
     ```

     注：`node.master: true`在非主节点下需要修改为false

   - 修改`node.name: bigdata111`，为该节点主机名

   - 修改`network.host: 192.168.64.129`为节点所在IP

   - 修改`discovery.zen.ping.unicast.hosts: ["bigdata111"]`为节点主机名，要和`node.name`相对应

3. 分别修改各新添加节点的linux系统环境

   注：和单节点的系统环境配置相同。

4. 各节点分别启动elasticsearch

   ```
   bin/elasticsearch
   ```

   使用插件查看集群状态

# 3. JAVA API操作

 Elasticsearch的Java客户端非常强大；它可以建立一个嵌入式实例并在必要时运行管理任务。

运行一个Java应用程序和Elasticsearch时，有两种操作模式可供使用。该应用程序可在Elasticsearch集群中扮演更加主动或更加被动的角色。在更加主动的情况下（称为Node Client），应用程序实例将从集群接收请求，确定哪个节点应处理该请求，就像正常节点所做的一样。（应用程序甚至可以托管索引和处理请求。）另一种模式称为Transport Client，它将所有请求都转发到另一个Elasticsearch节点，由后者来确定最终目标。

## 3.1 API基本操作

 Elasticsearch的Java客户端非常强大；它可以建立一个嵌入式实例并在必要时运行管理任务。

  运行一个Java应用程序和Elasticsearch时，有两种操作模式可供使用。该应用程序可在Elasticsearch集群中扮演更加主动或更加被动的角色。在更加主动的情况下（称为Node Client），应用程序实例将从集群接收请求，确定哪个节点应处理该请求，就像正常节点所做的一样。（应用程序甚至可以托管索引和处理请求。）另一种模式称为Transport Client，它将所有请求都转发到另一个Elasticsearch节点，由后者来确定最终目标。

### 3.1.1 maven准备

1. 创建maven，并添加如下pom依赖

   ```xml
   <dependencies>
           <dependency>
               <groupId>junit</groupId>
               <artifactId>junit</artifactId>
               <version>4.12</version>
               <scope>compile</scope>
           </dependency>
   
           <dependency>
               <groupId>org.elasticsearch</groupId>
               <artifactId>elasticsearch</artifactId>
               <version>5.6.1</version>
           </dependency>
   
           <dependency>
               <groupId>org.elasticsearch.client</groupId>
               <artifactId>transport</artifactId>
               <version>5.6.1</version>
           </dependency>
   
           <dependency>
               <groupId>org.apache.logging.log4j</groupId>
               <artifactId>log4j-core</artifactId>
               <version>2.9.0</version>
           </dependency>
   
       </dependencies>
   ```

2. 创建java文件，写入下面的方法

   作用：获取`Transport Client`

   因为每个操作都会需要连接集群，所以将冗余的操作统一起来。

   ```java
   // 对es所有操作都是通过client
   	private TransportClient client;
   
   	@SuppressWarnings("unchecked")
   	@Before
   	public void getClient() throws Exception {
   
   		// 1 设置连接的集群名称
   		Settings settings = Settings.builder().put("cluster.name", "my-application").build();
   
   		// 2 连接集群
   		client = new PreBuiltTransportClient(settings);
   
   
   client.addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("192.168.64.129"), 9300));
   
           //		client.addTransportAddress(new TransportAddress(InetAddress.getByName("192.168.64.129"), 9300));
           
           // 注意：在ES高版本连接集群时需要使用如上注释掉的方法
   		/**
   		 * 原因：这个类在新的版本中去掉了，之前的老版本就有
   		 *
   		 * 解决方案，点击addTransportAddress，进入看源码需要的参数类型为TransportAddress
   		 */
   	}
   ```

3. 在IDEA的resources目录中创建log4j2.xml文件

   原因：在执行代码的时候，会报一个不会阻止代码执行的错误，如果不想看到这个错误，可以将如下代码写成xml文件添加到resources文件中

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <Configuration>
       <Appenders>
           <Console name="STDOUT" target="SYSTEM_OUT">
               <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
           </Console>
           <RollingFile name="RollingFile" fileName="logs/strutslog1.log"
                        filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
               <PatternLayout>
                   <Pattern>%d{MM-dd-yyyy} %p %c{1.} [%t] -%M-%L- %m%n</Pattern>
               </PatternLayout>
               <Policies>
                   <TimeBasedTriggeringPolicy />
                   <SizeBasedTriggeringPolicy size="1 KB"/>
               </Policies>
               <DefaultRolloverStrategy fileIndex="max" max="2"/>
           </RollingFile>
       </Appenders>
       <Loggers>
           <Logger name="com.opensymphony.xwork2" level="WAN"/>
           <Logger name="org.apache.struts2" level="WAN"/>
           <Root level="warn">
               <AppenderRef ref="STDOUT"/>
           </Root>
       </Loggers>
   
   </Configuration>
   ```

   

### 3.1.2 创建索引

1. ElasticSearch服务默认端口9300。
2. Web管理平台端口9200。

```java
//	创建索引
	@Test
	public void createIndex_blog() {
		// 1 创建索引
		client.admin().indices().prepareCreate("blog4").get();
		// 2 关闭连接
		client.close();
	}
```

### 3.1.3 删除索引

```java
//	删除索引
	@Test
	public void deleteIndex() {
		client.admin().indices().prepareDelete("blog").get();

		client.close();
	}
```

### 3.1.4 新建document

1. 源数据json串

   ```java
   //	创建document（json方式）
   	@Test
   	public void createIndexDataByJson() {
   
   		// 使用 json 创建 document
   
   		// 1 文档数据准备
   		String json = "{" + "\"id\":\"1\"," + "\"title\":\"基于Lucene的搜索服务器\","
   				+ "\"content\":\"它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口\"" + "}";
   
   		// 2 创建文档
   		IndexResponse indexResponse = client.prepareIndex("blog4", "article", "1").setSource(json).execute()
   				.actionGet();
   
   		// 3 打印返回结果
   		System.out.println("index: " + indexResponse.getIndex());
   		System.out.println("type: " + indexResponse.getType());
   		System.out.println("id: " + indexResponse.getId());
   		System.out.println("version: " + indexResponse.getVersion());
   		System.out.println("result: " + indexResponse.getResult());
   
   		// 4 关闭连接
   		client.close();
   	}
   
   ```

2. 源数据map方式添加json

   ```java
   //	建立document （map方式）
   	@Test
   	public void createByMap() {
   
   		Map<String, Object> json = new HashMap<String, Object>();
   		json.put("id", "222");
   		json.put("title", "基于Lucene的搜索服务器");
   		json.put("content", "它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口");
   
   		IndexResponse indexResponse = client.prepareIndex("blog4", "article", "2").setSource(json).execute()
   				.actionGet();
   
   		// 3 打印返回结果
   		System.out.println("index: " + indexResponse.getIndex());
   		System.out.println("type: " + indexResponse.getType());
   		System.out.println("id: " + indexResponse.getId());
   		System.out.println("version: " + indexResponse.getVersion());
   		System.out.println("result: " + indexResponse.getResult());
   
   		// 4 关闭连接
   		client.close();
   
   	}
   
   ```

3. 源数据ES构建添加json

   ```java
   //	建立document   源数据es构建器添加json
   	@Test
   	public void createIndexDataByXContent() throws Exception {
   
   		XContentBuilder builder = XContentFactory.jsonBuilder()
   				.startObject()
   				.field("id", 3)
   				.field("title", "基于Lucene的搜索服务器")
   				.field("content", "它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。")
   				.endObject();
   
   		IndexResponse indexResponse = client.prepareIndex("blog4", "article", "3").setSource(builder).get();
   
   		// 3 打印返回结果
   		System.out.println("index: " + indexResponse.getIndex());
   		System.out.println("type: " + indexResponse.getType());
   		System.out.println("id: " + indexResponse.getId());
   		System.out.println("version: " + indexResponse.getVersion());
   		System.out.println("result: " + indexResponse.getResult());
   
   		// 4 关闭连接
   		client.close();
   
   	}
   
   ```

### 3.1.5 搜索文档数据

1. 单个索引

   ```java
   // 搜索文档数据（单个索引）
   	@Test
   	public void getData() {
   		// 查询文档操作
   
   		GetResponse response = client.prepareGet("blog4", "article", "2").get();
   
   		System.out.println(response.getSourceAsString());
   
   		client.close();
   	}
   ```

2. 多个索引

   ```java
   //	 搜索文档数据（多个索引）
   	@Test
   	public void getMultiData() {
   
   		// 查询多个文档
   
   		MultiGetResponse response = client.prepareMultiGet().add("blog4", "article", "1").add("blog4", "article", "2")
   				.add("blog4", "article", "1", "2").get();
   
   		// 遍历返回的结果
   		for (MultiGetItemResponse multiGetItemResponse : response) {
   			GetResponse getResponse = multiGetItemResponse.getResponse();
   
   			if (getResponse.isExists()) {
   				System.out.println(getResponse.getSourceAsString());
   			}
   		}
   
   		client.close();
   
   	}
   ```

### 3.1.6  更新文档数据

1. update

   ```java
   //	更新文档数据 update
   	@Test
   	public void updateData() throws Exception {
   		// 创建更新数据的请求对象
   		UpdateRequest updateRequest = new UpdateRequest();
   		updateRequest.index("blog4");
   		updateRequest.type("article");
   		updateRequest.id("3");
   
   		updateRequest.doc(XContentFactory.jsonBuilder().startObject()
   				// 对没有的字段添加, 对已有的字段替换
   				.field("title", "基于Lucene的搜索服务器").field("content", "它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。大数据前景无限")
   				.field("createDate", "2017-8-22").endObject());
   
   		// 获取更新后的值
   		UpdateResponse indexResponse = client.update(updateRequest).get();
   
   		// 打印返回结果
   		System.out.println("index: " + indexResponse.getIndex());
   		System.out.println("type: " + indexResponse.getType());
   		System.out.println("id: " + indexResponse.getId());
   		System.out.println("version: " + indexResponse.getVersion());
   		System.out.println("result: " + indexResponse.getResult());
   	}
   
   ```

2. upsert

   ```java
   //	更新文档数据 upsert，根据条件进行更新
   	@Test
   	public void testUpsert() throws Exception {
   		IndexRequest indexRequest = new IndexRequest("blog4", "article", "5")
   				.source(XContentFactory.jsonBuilder().startObject().field("title", "搜索服务器")
   						.field("content",
   								"它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。")
   						.endObject());
   
   //		UpdateRequest upsert = new UpdateRequest("blog4", "article", "5")
   //				.doc(XContentFactory.jsonBuilder().startObject().field("user", "李四").endObject()).upsert(indexRequest);
   		UpdateRequest upsert = new UpdateRequest("blog4", "article", "5")
   				.upsert(indexRequest);
   
   		client.update(upsert).get();
   		client.close();
   	}
   ```

### 3.1.7 删除文档数据

```java
//	删除文档数据
	@Test
	public void deleteData() {

		DeleteResponse indexResponse = client.prepareDelete("blog4", "article", "5").get();

		// 打印返回结果
		System.out.println("index: " + indexResponse.getIndex());
		System.out.println("type: " + indexResponse.getType());
		System.out.println("id: " + indexResponse.getId());
		System.out.println("version: " + indexResponse.getVersion());
		System.out.println("result: " + indexResponse.getResult());

		client.close();

	}

```

## 3.2 条件查询

### 3.2.1 查询所有

```java
//	查询所有
	@Test
	public void matchAllQuery() {
		// 相当于 查询所有 select * from emp;

		SearchResponse searchResponse = client.prepareSearch("blog4").setTypes("article")
				.setQuery(QueryBuilders.matchAllQuery()).get();

		SearchHits hits = searchResponse.getHits();

		System.out.println("查询结果有：" + hits.getTotalHits());

		for (SearchHit searchHit : hits) {
			System.out.println(searchHit.getSourceAsString());
		}

		client.close();
	}
```

### 3.2.2 分词查询

```java
//	分词查询
	@Test
	public void query() {
		// 条件查询 类似于 like 对所有字段都like
		SearchResponse searchResponse = client.prepareSearch("blog4").setTypes("article")
				.setQuery(QueryBuilders.queryStringQuery("web")).get();

		SearchHits hits = searchResponse.getHits();

		System.out.println("查询结果有：" + hits.getTotalHits());

		for (SearchHit searchHit : hits) {
			System.out.println(searchHit.getSourceAsString());
		}

		client.close();
	}
```

### 3.2.3 通配符查询

```java
//	通配符查询
	@Test
	public void wildcardQuery() {
		// 通配符查询
		SearchResponse searchResponse = client.prepareSearch("blog4").setTypes("article")
				.setQuery(QueryBuilders.wildcardQuery("content", "*全*")).get();

		SearchHits hits = searchResponse.getHits();

		System.out.println("查询结果有：" + hits.getTotalHits());

		for (SearchHit searchHit : hits) {
			System.out.println(searchHit.getSourceAsString());
		}

		client.close();
	}
```

### 3.2.4 词条查询

```java
//	词条查询
	@Test
	public void termQuery() {
		// 类似于 mysql 中的 =
		/**
		 * = 不是与字段等于 是与 字段的分词结果 等于
		 */
		SearchResponse searchResponse = client.prepareSearch("blog4").setTypes("article")
				.setQuery(QueryBuilders.termQuery("content", "基于RESTful")).get();

		SearchHits hits = searchResponse.getHits();

		System.out.println("查询结果有：" + hits.getTotalHits());

		for (SearchHit searchHit : hits) {
			System.out.println(searchHit.getSourceAsString());
		}

		client.close();
	}

```

### 3.2.5 模糊查询

```java
//	模糊查询
	@Test
	public void fuzzy() {
		// 模糊查询
		SearchResponse searchResponse = client.prepareSearch("blog4").setTypes("article")
				.setQuery(QueryBuilders.fuzzyQuery("content", "基于RESTful")).get();

		SearchHits hits = searchResponse.getHits();

		System.out.println("查询结果有：" + hits.getTotalHits());

		for (SearchHit searchHit : hits) {
			System.out.println(searchHit.getSourceAsString());
		}

		client.close();
	}
```

## 3.3 映射相关操作

```java
//	映射
	@Test
	public void createMapping() throws Exception {
		// 1设置mapping
		XContentBuilder builder = XContentFactory.jsonBuilder().startObject().startObject("article")
				.startObject("properties").startObject("id").field("type", "text").field("store", "false").endObject()
				.startObject("title").field("type", "text").field("store", "false").endObject().startObject("content")
				.field("type", "text").field("store", "false").endObject().endObject().endObject().endObject();

		// 添加 mapping
		PutMappingRequest mappingRequest = Requests.putMappingRequest("blog4").type("article").source(builder);

		client.admin().indices().putMapping(mappingRequest).get();

		client.close();

	}

```

# 4. IK分词器

## 4.1 IK分词器安装

1. 下载和elasticsearch对应的ik压缩文件

   ```
   ./elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.1.1/elasticsearch-analysis-ik-6.1.1.zip
   ```

2. 拷贝ik压缩文件至elasticsearch目录下的plugins下，并解压

   ```
   cp elasticsearch-analysis-ik-6.1.1.zip /opt/module/elasticsearch-6.1.1/plugins/
   unzip elasticsearch-analysis-ik-6.1.1.zip -d ik-analyzer
   ```

   解压后需要删除压缩包`rm -rf elasticsearch-analysis-ik-6.1.1.zip` 

3. 提取文件

   将解压后的目录内的elasticsearch目录单独提取出来，放入plugins根目录下

   ```
   mv elasticsearch  /opt/module/elasticsearch-6.1.1/plugins/
   ```

4. 删除原有压缩包`ik-analyzer`

注：网络上的教程直接解压出来的就是elasticsearch目录，但是测试时候的使用老师准备的安装包多了一个目录，在启动ik的时候报错了，经解决可以使用上面的方法，如果解压出来就只有一个elasticsearch目录应该是可以直接使用的，但是具体情况需要具体分析

## 4.2 基本使用

- ik_smart模式

  ```
  [root@bigdata111 ~]# curl -H "Content-Type:application/json" -XGET 'http://192.168.64.129:9200/_analyze?pretty' -d '{"analyzer":"ik_smart","text":"中华人民共和国"}'
  {
    "tokens" : [
      {
        "token" : "中华人民共和国",
        "start_offset" : 0,
        "end_offset" : 7,
        "type" : "CN_WORD",
        "position" : 0
      }
    ]
  }
  ```

- ik_max_word模式

  ```
  [root@bigdata111 ~]# curl -H "Content-Type:application/json" -XGET 'http://192.168.64.129:9200/_analyze?pretty' -d '{"analyzer":"ik_max_word","text":"中华人民共和国"}'
  {
    "tokens" : [
      {
        "token" : "中华人民共和国",
        "start_offset" : 0,
        "end_offset" : 7,
        "type" : "CN_WORD",
        "position" : 0
      },
      {
        "token" : "中华人民",
        "start_offset" : 0,
        "end_offset" : 4,
        "type" : "CN_WORD",
        "position" : 1
      },
      {
        "token" : "中华",
        "start_offset" : 0,
        "end_offset" : 2,
        "type" : "CN_WORD",
        "position" : 2
      },
      {
        "token" : "华人",
        "start_offset" : 1,
        "end_offset" : 3,
        "type" : "CN_WORD",
        "position" : 3
      },
      {
        "token" : "人民共和国",
        "start_offset" : 2,
        "end_offset" : 7,
        "type" : "CN_WORD",
        "position" : 4
      },
      {
        "token" : "人民",
        "start_offset" : 2,
        "end_offset" : 4,
        "type" : "CN_WORD",
        "position" : 5
      },
      {
        "token" : "共和国",
        "start_offset" : 4,
        "end_offset" : 7,
        "type" : "CN_WORD",
        "position" : 6
      },
      {
        "token" : "共和",
        "start_offset" : 4,
        "end_offset" : 6,
        "type" : "CN_WORD",
        "position" : 7
      },
      {
        "token" : "国",
        "start_offset" : 6,
        "end_offset" : 7,
        "type" : "CN_CHAR",
        "position" : 8
      }
    ]
  }
  ```

**Store 的解释：**

使用 elasticsearch 时碰上了很迷惑的地方，我看官方文档说 store 默认是 no ，我想当然的理解为也就是说这个 field 是不会 store 的，但是查询的时候也能查询出来，经过查找资料了解到原来 store 的意思是，是否在 _source 之外在独立存储一份，这里要说一下 _source 这是源文档，当你索引数据的时候， elasticsearch 会保存一份源文档到 _source ，如果文档的某一字段设置了 store 为 yes (默认为 no)，这时候会在 _source 存储之外再为这个字段独立进行存储，这么做的目的主要是针对内容比较多的字段，放到 _source 返回的话，因为_source 是把所有字段保存为一份文档，命中后读取只需要一次 IO，包含内容特别多的字段会很占带宽影响性能，通常我们也不需要完整的内容返回(可能只关心摘要)，这时候就没必要放到 _source 里一起返回了(当然也可以在查询时指定返回字段)。

# 5.Logstash

Logstash is a tool for managing events and logs. You can use it to collect logs, parse them, and store them for later use (like, for searching).

logstash是一个数据分析软件，主要目的是分析log日志。整一套软件可以当作一个MVC模型，logstash是controller层，Elasticsearch是一个model层，kibana是view层。

首先将数据传给logstash，它将数据进行过滤和格式化（转成JSON格式），然后传给Elasticsearch进行存储、建搜索的索引，kibana提供前端的页面再进行搜索和图表可视化，它是调用Elasticsearch的接口返回的数据进行可视化。logstash和Elasticsearch是用Java写的，kibana使用node.js框架。

这个软件官网有很详细的使用说明，https://www.elastic.co/，除了docs之外，还有视频教程。这篇博客集合了docs和视频里面一些比较重要的设置和使用。

![](img\Elasticsearch\Logstash.png)

## 5.1 安装

1. 下载安装包，也可以使用准备好的安装包

   https://www.elastic.co/downloads/logstash 

2. 解压至目标目录即可正常使用

3. 测试

   进入logstash根目录，执行如下代码

   ```
   bin/logstash -e 'input{stdin{}}output{stdout{codec=>rubydebug}}'
   ```

   输入hello word会出现如下内容

   ```
   hello word
   {
         "@version" => "1",
       "@timestamp" => 2020-03-04T12:13:12.260Z,
             "host" => "bigdata111",
          "message" => "hello word"
   }
   ```

   注：如果出现如下报错，请调高虚拟机内存容量。

   ```
   Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x00000000c5330000, 986513408, 0) failed; error='Cannot allocate memory' (errno=12)
   #
   # There is insufficient memory for the Java Runtime Environment to continue.
   # Native memory allocation (mmap) failed to map 986513408 bytes for committing reserved memory.
   # An error report file with more information is saved as:
   # /usr/local/logstash-6.6.2/confs_test/hs_err_pid3910.log
   ```

## 5.2 配置

准备工作：在logstash根目录下创建confs_test文件夹，用于存放相关配置文件

### 5.2.1 input配置

**读取文件**

1. 创建 `input_file.conf`文件

   vim input_file.conf

   ```
   input {
       file {
           path => ["/opt/file/test"]
           type => "system"
           start_position => "beginning"
   }
   
   }
   output{stdout{codec=>rubydebug}}                             
   ```

   注：test文件随意创建一个即可。

2. 保存退出

3. 说明

   有一些比较有用的配置项，可以用来指定 FileWatch 库的行为：

   - discover_interval

     logstash 每隔多久去检查一次被监听的 path 下是否有新文件。默认值是 15 秒。

   - exclude

     不想被监听的文件可以排除出去，这里跟 path 一样支持 glob 展开。

   - close_older

     一个已经监听中的文件，如果超过这个值的时间内没有更新内容，就关闭监听它的文件句柄。默认是 3600 秒，即一小时。

   - ignore_older

     在每次检查文件列表的时候，如果一个文件的最后修改时间超过这个值，就忽略这个文件。默认是 86400 秒，即一天。

   - sincedb_path

     如果你不想用默认的 $HOME/.sincedb(Windows 平台上在 `C:\Windows\System32\config\systemprofile\.sincedb)`，可以通过这个配置定义 sincedb 文件到其他位置。

   - sincedb_write_interval

     logstash 每隔多久写一次 sincedb 文件，默认是 15 秒。

   - stat_interval

     logstash 每隔多久检查一次被监听文件状态（是否有更新），默认是 1 秒。

   - start_position

     logstash 从什么位置开始读取文件数据，默认是结束位置，也就是说 logstash 进程会以类似 tail -F 的形式运行。如果你是要导入原有数据，把这个设定改成 "beginning"，logstash 进程就从头开始读取，类似 less +F 的形式运行。

4. 运行测试

   - 当前目录下执行如下命令

     ```
     ../bin/logstash -f ./input_file.conf
     ```

   - 然后另开一个窗口执行如下命令

     ```
     echo 'hehe' >> /opt/file/test
     ```

   - 此时会输出如下内容

     ```
     {
            "message" => "hehe",
               "type" => "system",
           "@version" => "1",
         "@timestamp" => 2020-03-04T12:23:34.684Z,
               "host" => "bigdata111",
               "path" => "/opt/file/test"
     }
     ```

**标准输入**

我们已经见过好几个示例使用 stdin 了。这也应该是 logstash 里最简单和基础的插件了。

1. 在目录`confs_test`中创建文件`input_stdin.conf`

   写入以下内容

   ```
   input {
       stdin {
           add_field => {"key" => "value"}
           codec => "plain"
           tags => ["add"]
           type => "std"
       }
   }
   output{stdout{codec=>rubydebug}}
   ```

2. 保存退出

3. 运行测试

   - 执行以下命令

     ```
     ../bin/logstash -f ./input_stdin.conf
     ```

   - 输入hello world，并查看结果

     ```
     hello world
     {
            "message" => "hello world",
                "key" => "value",
         "@timestamp" => 2020-03-04T12:28:33.197Z,
               "type" => "std",
               "tags" => [
             [0] "add"
         ],
           "@version" => "1",
               "host" => "bigdata111"
     }
     ```

   - 说明：

     type 和 tags 是 logstash 事件中两个特殊的字段。通常来说我们会在输入区段中通过 type 来标记事件类型。而 tags 则是在数据处理过程中，由具体的插件来添加或者删除的。

     最常见的用法是像下面这样：

     ```
     input {
         stdin {
             type => "web"
         }
     }
     filter {
         if [type] == "web" {
             grok {
                 match => ["message", %{COMBINEDAPACHELOG}]
             }
         }
     }
     output {
         if "_grokparsefailure" in [tags] {
             nagios_nsca {
                 nagios_status => "1"
             }
         } else {
             elasticsearch {
             }
         }
     }
     
     ```

### 5.2.2 codec配置

Codec 是 logstash 从 1.3.0 版开始新引入的概念(Codec 来自 Coder/decoder 两个单词的首字母缩写)。

在此之前，logstash 只支持纯文本形式输入，然后以过滤器处理它。但现在，我们可以在输入期处理不同类型的数据，这全是因为有了 codec 设置。

所以，这里需要纠正之前的一个概念。Logstash 不只是一个input | filter | output 的数据流，而是一个 input | decode | filter | encode | output 的数据流！codec 就是用来 decode、encode 事件的。

codec 的引入，使得 logstash 可以更好更方便的与其他有自定义数据格式的运维产品共存，比如 graphite、fluent、netflow、collectd，以及使用 msgpack、json、edn 等通用数据格式的其他产品等。

事实上，我们在第一个 "hello world" 用例中就已经用过 codec 了 —— rubydebug 就是一种 codec！虽然它一般只会用在 stdout 插件中，作为配置测试或者调试的工具。

 **采用 JSON 编码**

在早期的版本中，有一种降低 logstash 过滤器的 CPU 负载消耗的做法盛行于社区(在当时的 cookbook 上有专门的一节介绍)：直接输入预定义好的 JSON 数据，这样就可以省略掉 filter/grok 配置！

这个建议依然有效，不过在当前版本中需要稍微做一点配置变动 —— 因为现在有专门的 codec 设置。 

1. 创建文件`codec_test.conf`

   ```
   input {
       stdin {
           add_field => {"key" => "value"}
           codec => "json"
           type => "std"
       }
   }
   output{stdout{codec=>rubydebug}}
   ```

2. 保存退出

3. 执行如下命令运行

   ```
    ../bin/logstash -f ./codec_test.conf 
   ```

4. 输入如下内容

   ```
   {"simCar":18074045598,"validityPeriod":"1996-12-06","unitPrice":9,"quantity":19,"amount":35,"imei":887540376467915,"user":"test"}
   ```

5. 运行结果

   ```
   {"simCar":18074045598,"validityPeriod":"1996-12-06","unitPrice":9,"quantity":19,"amount":35,"imei":887540376467915,"user":"test"}
   {
                  "key" => "value",
               "simCar" => 18074045598,
                 "user" => "test",
               "amount" => 35,
                 "imei" => 887540376467915,
                 "type" => "std",
             "@version" => "1",
             "quantity" => 19,
            "unitPrice" => 9,
                 "host" => "bigdata111",
       "validityPeriod" => "1996-12-06",
           "@timestamp" => 2020-03-04T12:32:07.571Z
   }	
   ```

### 5.2.3 filter配置

**Grok插件**

logstash拥有丰富的filter插件,它们扩展了进入过滤器的原始数据，进行复杂的逻辑处理，甚至可以无中生有的添加新的 logstash 事件到后续的流程中去！Grok 是 Logstash 最重要的插件之一。也是迄今为止使蹩脚的、无结构的日志结构化和可查询的最好方式。Grok在解析 syslog logs、apache and other webserver logs、mysql logs等任意格式的文件上表现完美。 

这个工具非常适用于系统日志，Apache和其他网络服务器日志，MySQL日志等。

1. 创建文件`filter_test.conf`

   ```
   input {
       stdin {
           type => "std"
       }
   }
   filter {
     grok {
       match=>{"message"=> "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}" }
     }
   }
   output{stdout{codec=>rubydebug}}
   ```

2. 保存退出

3. 运行

   ```
   ../bin/logstash -f ./filter_test.conf 
   ```

4. 输入如下内容

   ```
   55.3.244.1 GET /index.html 15824 0.043
   ```

5. 输出结果

   ```
   55.3.244.1 GET /index.html 15824 0.043
   {
             "type" => "std",
       "@timestamp" => 2020-03-04T12:37:58.202Z,
         "duration" => "0.043",
             "host" => "bigdata111",
           "method" => "GET",
          "request" => "/index.html",
           "client" => "55.3.244.1",
          "message" => "55.3.244.1 GET /index.html 15824 0.043",
         "@version" => "1",
            "bytes" => "15824"
   }
   ```

**grok模式的语法如下：**`%{SYNTAX:SEMANTIC}`

- SYNTAX：代表匹配值的类型,例如3.44可以用NUMBER类型所匹配,127.0.0.1可以使用IP类型匹配。
- SEMANTIC：代表存储该值的一个变量名称,例如 3.44 可能是一个事件的持续时间,127.0.0.1可能是请求的client地址。所以这两个值可以用 %{NUMBER:duration} %{IP:client} 来匹配。

你也可以选择将数据类型转换添加到Grok模式。默认情况下，所有语义都保存为字符串。如果您希望转换语义的数据类型，例如将字符串更改为整数，则将其后缀为目标数据类型。例如%{NUMBER:num:int}将num语义从一个字符串转换为一个整数。目前唯一支持的转换是int和float。

Logstash附带约120个模式。你可以在这里找到它们https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns

 **自定义类型**

更多时候logstash grok没办法提供你所需要的匹配类型，这个时候我们可以使用自定义。

创建自定义 patterns 文件。

1. 创建一个名为patterns其中创建一个文件postfix （文件名无关紧要,随便起）,在该文件中，将需要的模式写为模式名称，空格，然后是该模式的正则表达式。例如：

   ```
   POSTFIX_QUEUEID [0-9A-F]{10,11}
   ```

2. 然后使用这个插件中的patterns_dir设置告诉logstash目录是你的自定义模式。

   - 创建配置文件

     ```
     input {
         stdin {
             type => "std"
         }
     }
     filter {
       grok {
         patterns_dir => ["./patterns"]
         match => { "message" => "%{SYSLOGBASE} %{POSTFIX_QUEUEID:queue_id}: %{GREEDYDATA:syslog_message}" }
       }
     }
     output{stdout{codec=>rubydebug}}
     ```

   - 输入如下内容

     ```
     Jan  1 06:25:43 mailserver14 postfix/cleanup[21403]: BEF25A72965: message-id=<20130101142543.5828399CCAF@mailserver1
     ```

   - 输出结果

     ```
     {
               "queue_id" => "BEF25A72965",
                "message" => "Jan  1 06:25:43 mailserver14 postfix/cleanup[21403]: BEF25A72965: message-id=<20130101142543.5828399CCAF@mailserver1",
                    "pid" => "21403",
                "program" => "postfix/cleanup",
               "@version" => "1",
                   "type" => "std",
              "logsource" => "mailserver14",
                   "host" => "zzc-203",
              "timestamp" => "Jan  1 06:25:43",
         "syslog_message" => "message-id=<20130101142543.5828399CCAF@mailserver1",
             "@timestamp" => 2019-03-19T05:31:37.405Z
     }
     ```

**GeoIP 地址查询归类**

GeoIP 是最常见的免费 IP 地址归类查询库，同时也有收费版可以采购。GeoIP 库可以根据 IP 地址提供对应的地域信息，包括国别，省市，经纬度等，对于可视化地图和区域统计非常有用。

1.  创建配置文件

   ```
   input {
       stdin {
           type => "std"
       }
   }
   filter {
       geoip {
           source => "message"
       }
   }
   output{stdout{codec=>rubydebug}}
   ```

2. 输入如下内容

   ```
   183.60.92.253
   ```

3. 输出结果

   ```
   {
             "type" => "std",
         "@version" => "1",
       "@timestamp" => 2019-03-19T05:39:26.714Z,
             "host" => "zzc-203",
          "message" => "183.60.92.253",
            "geoip" => {
            "country_code3" => "CN",
                 "latitude" => 23.1167,
              "region_code" => "44",
              "region_name" => "Guangdong",
                 "location" => {
               "lon" => 113.25,
               "lat" => 23.1167
           },
                "city_name" => "Guangzhou",
             "country_name" => "China",
           "continent_code" => "AS",
            "country_code2" => "CN",
                 "timezone" => "Asia/Shanghai",
                       "ip" => "183.60.92.253",
                "longitude" => 113.25
       }
   }
   ```

### 5.2.4 output配置

**标准输出**

通过日志收集系统将分散在数百台服务器上的数据集中存储在某中心服务器上，这是运维最原始的需求。Logstash 当然也能做到这点。

和 LogStash::Inputs::File 不同, LogStash::Outputs::File 里可以使用 sprintf format 格式来自动定义输出到带日期命名的路径。

1. 创建配置文件

   ```
   input {
       stdin {
           type => "std"
       }
   }
   output {
       file {
           path => "/opt/file/%{+yyyy}/%{+MM}/%{+dd}/%{host}.log"
           codec => line { format => "custom format: %{message}"}
       }
   }
   ```

2. 启动后输入任意内容，即可看到该文件

   ```
   [root@bigdata111 04]# cat bigdata111.log 
   custom format: helwqweqeq
   ```

**写入ES**

1. 创建配置文件

   ```
   input {
       stdin {
           type => "log2es"
       }
   }
   output {
       elasticsearch {
           hosts => ["192.168.64.129:9200"]
           index => "logstash-%{type}-%{+YYYY.MM.dd}"
           document_type => "%{type}"
           sniffing => true
           template_overwrite => true
       }
   }
   ```

2. 运行

3. 在head插件中可以看到数据。

**实战举例，将错误日志写入ES**

1. 创建配置文件

   ```
   input {
       file {
           path => ["/usr/local/logstash-6.6.2/data_test/run_error.log"]
           type => "error"
           start_position => "beginning"
   }
   
   }
   output {
       elasticsearch {
           hosts => ["192.168.109.133:9200"]
           index => "logstash-%{type}-%{+YYYY.MM.dd}"
           document_type => "%{type}"
           sniffing => true
           template_overwrite => true
       }
   }
   ```

2. 运行后，可以在访问ES的webUI页面查看结果

# 6.Kibana

Kibana是一个开源的分析和可视化平台，设计用于和Elasticsearch一起工作。

你用Kibana来搜索，查看，并和存储在Elasticsearch索引中的数据进行交互。

你可以轻松地执行高级数据分析，并且以各种图标、表格和地图的形式可视化数据。

Kibana使得理解大量数据变得很容易。它简单的、基于浏览器的界面使你能够快速创建和共享动态仪表板，实时显示Elasticsearch查询的变化。

**安装步骤**

1. 解压压缩包

   ```
   tar -zxvf kibana-6.6.1-linux-x86_64.tar.gz /opt/module
   ```

2. 修改目录名（可省略）

   ```
   mv kibana-6.6.1-linux-x86_64/ kibana-6.6.1
   ```

3. 修改配置文件`/opt/module/kibana-6.1.1/config`

   单节点即可

   ```
   server.port: 5601
   server.host: "192.168.64.129"  部署节点的IP
   elasticsearch.url: "http://192.168.64.129:9200"
   kibana.index: ".kibana"
   ```

4. 启动

   ```
   bin/kibana
   ```

   webUI

   ```
   bigdata111:5601
   ```


**使用javaAPI创建一些数据，然后使用kibana查看**

```java
	@Test
	public void kibana_data_generate() {
		// 创建索引
		client.admin().indices().prepareCreate("kibana_data").get();

		// 创建示例数据

		// 使用map创建document
		// 文档数据准备
		Map<String, Object> json = null;
		IndexResponse indexResponse = null;
		for(int i = 1; i <= 1000 ; i++){
			json = new HashMap<String, Object>();
			json.put("id", i + "");
			json.put("status_code", generateStatusCode() + "");
			json.put("response_time", Math.random());
			indexResponse = client.prepareIndex("kibana_data", "access_log", i + "").setSource(json).execute().actionGet();
		}

		// 关闭连接
		client.close();
	}

	public int generateStatusCode () {
		int status_code = 0;
		double randomNum = Math.random();
		if (randomNum <= 0.7) {
			status_code = 200;
		} else if (randomNum <=0.8) {
			status_code = 404;
		} else if (randomNum <=0.95) {
			status_code = 500;
		} else {
			status_code = 401;
		}

		return status_code;
	}

	@Test
	public void kibana_data_generate_again() {

		// 使用map创建document
		// 文档数据准备
		Map<String, Object> json = null;
		IndexResponse indexResponse = null;
		for(int i = 3000; i <= 5000 ; i++){
			json = new HashMap<String, Object>();
			json.put("id", i + "");
			json.put("status_code", generateStatusCode2() + "");
			json.put("response_time", Math.random());
			indexResponse = client.prepareIndex("kibana_data", "access_log", i + "").setSource(json).execute().actionGet();
		}

		// 关闭连接
		client.close();
	}

	public int generateStatusCode2 () {
		int status_code = 0;
		double randomNum = Math.random();
		if (randomNum <= 0.7) {
			status_code = 500;
		} else if (randomNum <=0.8) {
			status_code = 404;
		} else if (randomNum <=0.95) {
			status_code = 200;
		} else {
			status_code = 401;
		}

		return status_code;
	}
```

