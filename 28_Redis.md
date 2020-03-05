# 28_Redis

# 1.NoSql数据库的发展历史简介

## 1.1 web系统的变迁历史

![](img\redis\nosql变迁历史.png)

**web1.0时代简介**

基本上就是一些简单的静态页面的渲染，不会涉及到太多的复杂业务逻辑，功能简单单一，基本上服务器性能不会有太大压力

**缺点：**

1. Service 越来越多，调用关系变复杂，前端搭建本地环境不再是一件简单的事。考虑团队协作，往往会考虑搭建集中式的开发服务器来解决。这种解决方案对编译型的后端开发来说也许还好，但对前端开发来说并不友好。天哪，我只是想调整下按钮样式，却要本地开发、代码上传、验证生效等好几个步骤。也许习惯了也还好，但开发服务器总是不那么稳定，出问题时往往需要依赖后端开发搞定。看似仅仅是前端开发难以本地化，但这对研发效率的影响其实蛮大。
2. JSP 等代码的可维护性越来越差。JSP 非常强大，可以内嵌 Java 代码。这种强大使得前后端的职责不清晰，JSP 变成了一个灰色地带。经常为了赶项目，为了各种紧急需求，会在 JSP 里揉杂大量业务代码。积攒到一定阶段时，往往会带来大量维护成本。

**web2.0时代简介**

![](img\redis\web2.0.png)

随着Web2.0的时代的到来，用户访问量大幅度提升，同时产生了大量的用户数据。加上后来的智能移动设备的普及，所有的互联网平台都面临了巨大的性能挑战。包括web服务器CPU及内存压力。数据库服务器IO压力等

为了解决服务器的性能压力问题，出现了各种各样的解决方案，最典型的就是使用MVC的架构，MVC 是个非常好的协作模式，从架构层面让开发者懂得什么代码应该写在什么地方。为了让 View 层更简单干脆，还可以选择 Velocity、Freemaker 等模板，使得模板里写不了 Java 代码。看起来是功能变弱了，但正是这种限制使得前后端分工更清晰。但是同样也会面临以下问题

1. 前端开发重度依赖开发环境。这种架构下，前后端协作有两种模式：一种是前端写 demo，写好后，让后端去套模板。淘宝早期包括现在依旧有大量业务线是这种模式。好处很明显，demo 可以本地开发，很高效。不足是还需要后端套模板，有可能套错，套完后还需要前端确定，来回沟通调整的成本比较大。另一种协作模式是前端负责浏览器端的所有开发和服务器端的 View 层模板开发，支付宝是这种模式。好处是 UI 相关的代码都是前端去写就好，后端不用太关注，不足就是前端开发重度绑定后端环境，环境成为影响前端开发效率的重要因素。

2. 前后端职责依旧纠缠不清。Velocity 模板还是蛮强大的，变量、逻辑、宏等特性，依旧可以通过拿到的上下文变量来实现各种业务逻辑。这样，只要前端弱势一点，往往就会被后端要求在模板层写出不少业务代码。还有一个很大的灰色地带是 Controller，页面路由等功能本应该是前端最关注的，但却是由后端来实现。Controller 本身与 Model 往往也会纠缠不清，看了让人咬牙的代码经常会出现在 Controller 层。这些问题不能全归结于程序员的素养，否则 JSP 就够了。

**关于如何解决Web服务器的负载压力**，其中最常用的一种方式就是使用nginx实现web集群的服务转发以及服务拆分等等

![](img\redis\nginx.png) 但是这样也会存在问题，后端服务器的多个tomcat之间如何解决session共享的问题，以及session存放的问题等等，为了解决session存放的问题，也有多种解决方案

方案一：存放在cookie里面。不安全，否定

方案二：存放在文件或者数据库当中。速度慢

方案三：session复制。大量session冗余，节点浪费大

方案四：使用NoSQL缓存数据库。例如redis或者memcache等，完美解决

## 1.2 NoSql适用场景

- 对数据高并发的读写
- 海量数据的读写
- 对数据高可扩展性
- 速度够快，能够快速读取数据

## 1.3 NoSql不适用场景

- 需要事务支持
- 基于sql的结构化查询存储，处理复杂的关系，需要即席查询（用户自定义查询条件的查询）

**总结：**用不着sql和用了sql也不行的情况，可以考虑使用NoSql

# 2. NoSql数据库兄弟会

## 2.1 memcache介绍

- 很早出现的NoSql数据库
- 数据都在内存中，一般不持久化
- 支持简单的key-value模式
- 一般作为缓存数据库辅助持久化的数据库

## 2.2 redis介绍

- 几乎覆盖了Memcached的绝大部分功能
- 数据都在内存汇总，支持持久化，主要用做备份恢复
- 除了支持简单的key-value模式，还支持多种数据结构的存储，比如：list，set，hash，zset等
- 一般是作为缓存数库辅助持久化的数据库
- 现在市面上使用非常多的一款内存数据库

## 2.3 mongoDB介绍

- 高性能、开源、模式自由（schema free）的文档型数据库
- 数据都在内存汇总，如果内存不足，把不常用的数据保存到硬盘
- 虽然是key-value模式，但是对value（尤其是json）提供了丰富的查询功能
- 支持二进制数据及大型对象
- 可以根据数据的特点代替RDBMS，成为独立的数据库。或者配合RDBMS，存储特定的数据

## 2.4 列式存储HBase介绍

HBase是**Hadoop**项目中的数据库。它用于需要对大量的数据进行随机、实时的读写操作的场景中。HBase的目标就是处理数据量非常庞大的表，可以用普通的计算机处理超过10亿行数据，还可处理有数百万列元素的数据表。

# 3. Redis的基本介绍以及使用场景

redis官网地址：

https://redis.io/

中文网站

http://www.redis.cn/

## 3.1 redis的基本介绍

redis是当前比较热门的NoSql系统之一，它是一个开源使用的ANSL，C语言编写的key-value存储系统（区别于MySql的二维表格的形式存储）。和Memcache类似，但是很大程度补偿了Memcache的不足。和Memcache一样，redis数据都是缓存在计算机内存中，不同的是**，Memche只能将数据缓存到内存当中，无法自动定期写入硬盘**，这就表示，一旦断电或者重启，内存清空，数据丢失。所以**Memcache的应用场景适用于缓存无需持久化的数据**。而**redis不同的是他会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，实现数据的持久化**

## 3.2 redis的适用场景

### 3.2.1取最新N个数据的操作

比如，典型的取你网站的最新文章，通过下面的方式，我们可以将最新的5000条评论的ID放在Redis的list集合中，并将超出集合部分从数据库获取。

- 使用`LPUSH latest.comments<ID>`命令，向list集合中插入数据

- 插入完成后再用`LTRIM latest.comments 0 5000`命令使其永远只保存最近5000个ID

- 然后我们在客户端获取某一页评论时可以使用下面的逻辑（伪代码）

  ```s
  FUNCTION get_latest_comments(start,num_items):
      id_list = redis.lrange("latest.comments",start,start+num_items-1)
      IF id_list.length < num_items
          id_list = SQL_DB("SELECT ... ORDER BY time LIMIT ...")
      END
      RETURN id_list
  END
  ```

如果你还有不同的筛选维度，比如某个分类的最新N条，那么你可以在建一个按此分类的list，只存ID的话，Redis是非常高效的

### 3.2.2 排行榜应用，取TOP N操作

这个需求与上面需求的不同之处在于，**前面操作以时间为权重，这个是以某个条件为权重**，比如按顶的次数排序，这个时候就需要我们的sorted set出马了，将你要排序的值设置成sorted set 的 score，将具体的数据设置成相应的value，每次只需要执行一次ZADD命令即可

### 3.2.3 需要精准设定过期时间的应用

比如你可以把上面说到的sorted set的score值设置成过期时间的时间戳，那么就可以简单地通过过期时间排序，定时清除过期数据了，不仅是清除Redis中的过期数据，你完全可以把Redis里这个过期时间当成是对数据库中数据的索引，用Redis来找出哪些数据需要过期删除，然后再精准地从数据库中删除相应的记录。

### 3.2.4 计数器应用

Redis的命令都是原子性的，你可以轻松地利用INCR、DECR命令来构建计数器系统

### 3.2.5 uniq操作，获取某段时间所有数据排重值

这个使用Redis的set数据结构最合适了，只需要不断地将数据往set中扔就行了，set意为集合，所以会自动排重。

### 3.2.6 实时系统，反垃圾系统

通过上面说到的set功能，你可以知道一个终端用户是否进行了某个操作，可以找到其操作的集合并进行分析统计对比等。没有做不到，只有想不到。

### 3.2.7 Pub/Sub构建实时消息系统

Redis的Pub/Sub系统可以构建实时的消息系统，比如很多用Pub/Sub构建的实时聊天系统的例子。

### 3.2.8 构建队列系统

使用list可以构建队列系统，使用sorted set甚至可以构建有优先级的队列系统。

### 3.2.9缓存

将数据直接存放到内存中，性能优于Memcached，数据结构更多样化。

## 3.3 redis的特点

- 高效性：

  redis的读取速度是110000次/s，写的速度是81000次/s

- 原子性：

  redis的所有操作都是原子性的，同时redis还支持对几个操作全并后的原子性执行

- 支持多种数据结构

  string（字符串）；list（列表）；hash（哈希），set（集合）；zset(有序集合)

- 稳定性

  持久化，主从复制（集群）

- 其他特性

  支持过期时间，支持事务，消息订阅

# 4. Redis环境安装（单节点）

## 4.1 下载安装包

```
wget http://download.redis.io/releases/redis-3.2.8.tar.gz
```

也可以上传准备好的redis压缩包

## 4.2 解压压缩包到指定目录

```
tar -zxvf redis-3.2.8.tar.gz -C /opt/module
```

## 4.3 安装c程序运行环境

```
yum -y install gcc-c++
```

## 4.4 安装较新版本的tcl

使用在线方式进行安装

```
yum  -y  install  tcl
```

## 4.5 编译redis

进入redis安装的根目录，执行以下命令

```
make MALLOC=libc 
```

## 4.6 修改redis配置文件

首先，在redis的根目录下创建两个文件夹

```
mkdir logs
mkdir redisdata
```

在redis的根目录下有redis.conf的配置文件

vim redis.conf，修改以下内容

```
bind bigdata111
daemonize yes
pidfile /var/run/redis_6379.pid
logfile "/opt/module/redis-3.2.8/logs/redis.log"
dir /opt/module/redis-3.2.8/redisdata
```

## 4.7 启动redis

```
src/redis-server redis.conf
```

## 4.8 连接redis客户端

```
src/redis-cli -h bigdata111
```

# 5.Redis的数据类型

![](img\redis\redis的数据类型.png)

redis当中一共支持五种数据类型，分别是string字符串类型，list列表类型，集合set类型，hash表类型以及有序集合zset类型，通过这五种不同的数据类型，我们可以实现各种不同的功能也可以应用于各种不同的场景，接下来我们来看五种数据类型的操作语法

![](img\redis\redis的key-value.png)

redis当中各种数据类型结构如上图：

redis当中各种数据类型的操作

https://www.runoob.com/redis/redis-keys.html

## 5.1 rediss当中对字符串string的操作

| 序号 | 命令                             | 描述                                                         | 示例                                             |
| ---- | -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| 1    | SET key value                    | 设定指定key的值                                              | SET hello world                                  |
| 2    | GET key                          | 获取指定 key 的值                                            | GET hello                                        |
| 3    | GETRANGE key start end           | 返回 key 中字符串值的子字符                                  | GETRANGE hello 0 3                               |
| 4    | GETSET key value                 | 将给定 key 的值设为 value ，并返回 key 的旧值(old value)。   | GETSET hello world2                              |
| 5    | MGET key1 [key2..]               | 获取所有(一个或多个)给定 key 的值。                          | MGET hello world                                 |
| 6    | SETEX key seconds value          | 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位) | SETEX hello 10 world3                            |
| 7    | SETNX key value                  | 只有在 key 不存在时设置 key 的值。                           | SETNX itcast redisvalue                          |
| 8    | SETRANGE key offset value        | 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始 | SETRANGE itcast 0 helloredis                     |
| 9    | STRLEN key                       | 返回 key 所储存的字符串值的长度                              | STRLEN itcast                                    |
| 10   | MSET key value [key value ...]   | 同时设置一个或多个 key-value 对。                            | itcast2 itcastvalue2 itcast3 itcastvalue3        |
| 12   | MSETNX key value [key value ...] | 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。 | MSETNX itcast4 itcastvalue4 itcast5 itcastvalue5 |
| 13   | PSETEX key milliseconds value    | 这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。 | PSETEX itcast6 6000 itcast6value                 |
| 14   | INCR key                         | 将 key 中储存的数字值增一                                    | set itcast7 1<br> INCR itcast7<br> GET itcast7   |
| 15   | INCRBY key increment             | 将 key 所储存的值加上给定的增量值（increment）               | INCRBY itcast7 2get itcast7                      |
| 16   | INCRBYFLOAT key increment        | 将 key 所储存的值加上给定的浮点增量值（increment）           | INCRBYFLOAT itcast7 0.8                          |
| 17   | DECR key                         | 将 key 中储存的数字值减一                                    | set itcast8 1DECR itcast8GET itcast8             |
| 18   | DECRBY key decrement             | key 所储存的值减去给定的减量值（decrement）                  | DECRBY itcast8 3                                 |
| 19   | APPEND key value                 | 如果 key 已经存在并且是一个字符串， APPEND 命令将指定的 value 追加到该 key 原来值（value）的末尾。 | APPEND itcast8 hello                             |

## 5.2 redis当中对hash列表的操作

Redis hash 是一个string类型的field和value的映射表，hash特别适合用于存储对象。

Redis 中每个 hash 可以存储 232 - 1 键值对（40多亿）

| 序号 | 命令                                     | 描述                                                     | 示例                                                         |
| ---- | ---------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| 1    | HSET key field value                     | 将哈希表 key 中的字段 field 的值设为 value               | HSET key1 field1 value1                                      |
| 2    | HSETNX key field value                   | 只有在字段 field 不存在时，设置哈希表字段的值。          | HSETNX key1 field2 value2                                    |
| 3    | HMSET key field1 value1 [field2 value2 ] | 同时将多个 field-value (域-值)对设置到哈希表 key 中。    | HMSET key1 field3 value3 field4 value4                       |
| 4    | HEXISTS key field                        | 看哈希表 key 中，指定的字段是否存在。                    | HEXISTS key1 field4HEXISTS key1 field6                       |
| 5    | HGET key field                           | 获取存储在哈希表中指定字段的值                           | HGET key1 field4                                             |
| 6    | HGETALL key                              | 获取在哈希表中指定 key 的所有字段和值                    | HGETALL key1                                                 |
| 7    | HKEYS key                                | 获取所有哈希表中的字段                                   | HKEYS key1                                                   |
| 8    | HLEN key                                 | 获取哈希表中字段的数量                                   | HLEN key1                                                    |
| 9    | HMGET key field1 [field2]                | 获取所有给定字段的值                                     | HMGET key1 field1 field2                                     |
| 10   | HINCRBY key field increment              | 为哈希表 key 中的指定字段的整数值加上增量 increment 。   | HSET key2 field1 <br>1HINCRBY key2 field1 1            <br>HGET key2 field1 |
| 11   | HINCRBYFLOAT key field increment         | 为哈希表 key 中的指定字段的浮点数值加上增量 increment 。 | HINCRBYFLOAT key2 field1 0.8                                 |
| 12   | HVALS key                                | 获取哈希表中所有值                                       | HVALS key1                                                   |
| 13   | HDEL key field1 [field2]                 | 删除一个或多个哈希表字段                                 | HDEL key1 field1	HVALS key1                               |

## 5.3 redis当中对list列表的操作

Redis列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）

一个列表最多可以包含 232 - 1 个元素 (4294967295, 每个列表超过40亿个元素)。

 

| 序号 | 命令                                  | 描述                                                         | 示例                                           |
| ---- | ------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| 1    | LPUSH key value1 [value2]             | 将一个或多个值插入到列表头部                                 | LPUSH list1 value1 value2                      |
| 2    | LRANGE key start stop                 | 查看list当中所有的数据                                       | LRANGE list1 0 -1                              |
| 3    | [LPUSHX key value                     | 将一个值插入到已存在的列表头部                               | LPUSHX list1 value3 <br>LINDEX list1 0         |
| 4    | RPUSH key value1 [value2]             | 在列表中添加一个或多个值                                     | RPUSH list1 value4 <br>value5LRANGE list1 0 -1 |
| 5    | RPUSHX key value                      | 为已存在的列表添加值                                         | RPUSHX list1 value6                            |
| 6    | LINSERT key BEFORE\|AFTER pivot value | 在列表的元素前或者后插入元素                                 | LINSERT list1 BEFORE value3 beforevalue3       |
| 7    | LINDEX key index                      | 通过索引获取列表中的元素                                     | LINDEX list1 0                                 |
| 8    | LSET key index value                  | 通过索引设置列表元素的值                                     | LSET list1 0 hello                             |
| 9    | LLEN key                              | 获取列表长度                                                 | LLEN list1                                     |
| 10   | LPOP key                              | 移出并获取列表的第一个元素                                   | LPOP list1                                     |
| 11   | RPOP key                              | 移除列表的最后一个元素，返回值为移除的元素。                 | RPOP list1                                     |
| 12   | BLPOP key1 [key2 \] timeout]          | 移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止 | BLPOP list1 2000                               |
| 13   | BRPOP key1 [key2 \] timeout           | 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止 | BRPOP list1 2000                               |
| 14   | RPOPLPUSH source destination          | 移除列表的最后一个元素，并将该元素添加到另一个列表并返回     | RPOPLPUSH list1 list2                          |
| 15   | BRPOPLPUSH source destination timeout | 从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止 | BRPOPLPUSH list1 list2 2000                    |
| 16   | LTRIM key start stop                  | 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除 | LTRIM list1 0 2                                |
| 17   | DEL key1 key2                         | 删除指定key的列表                                            | DEL list2                                      |

## 5.4 redis操作set集合

redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。

Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。

集合中最大的成员数为 232 - 1 (4294967295, 每个集合可存储40多亿个成员)。

 

| 序号 | 命令                                | 描述                                                | 示例                                             |
| ---- | ----------------------------------- | --------------------------------------------------- | ------------------------------------------------ |
| 1    | SADD key member1 [member2]          | 向集合添加一个或多个成员                            | SADD set1 setvalue1 setvalue2                    |
| 2    | SMEMBERS key                        | 返回集合中的所有成员                                | SMEMBERS set1                                    |
| 3    | SCARD key                           | 获取集合的成员数                                    | SCARD set1                                       |
| 4    | SDIFF key1 [key2]                   | 返回给定所有集合的差集                              | SADD set2 setvalue2 setvalue3<br>SDIFF set1 set2 |
| 5    | SDIFFSTORE destination key1 [key2]  | 返回给定所有集合的差集并存储在 destination 中       | SDIFFSTORE set3 set1 set2                        |
| 6    | SINTER key1 [key2]                  | 返回给定所有集合的交集                              | SINTER set1 set2                                 |
| 7    | SINTERSTORE destination key1 [key2] | 返回给定所有集合的交集并存储在 destination 中       | SINTERSTORE set4 set1 set2                       |
| 8    | SISMEMBER key member                | 判断 member 元素是否是集合 key 的成员               | SISMEMBER set1 setvalue1                         |
| 9    | SMOVE source destination member     | 将 member 元素从 source 集合移动到 destination 集合 | SMOVE set1 set2 setvalue1                        |
| 10   | SPOP key                            | 移除并返回集合中的一个随机元素                      | SPOP set2                                        |
| 11   | SRANDMEMBER key [count]             | 返回集合中一个或多个随机数                          | SRANDMEMBER set2 2                               |
| 12   | SREM key member1 [member2]          | 移除集合中一个或多个成员                            | SREM set2 setvalue1                              |
| 13   | SUNION key1 [key2]                  | 返回所有给定集合的并集                              | SUNION set1 set2                                 |
| 14   | SUNIONSTORE destination key1 [key2] | 所有给定集合的并集存储在 destination 集合中         | SUNIONSTORE set5 set1 set2                       |

## 5.5 redis中对key的操作

| 序号 | 命令                     | 描述                                                         | 示例                 |
| ---- | ------------------------ | ------------------------------------------------------------ | -------------------- |
| 1    | DEL key                  | 该命令用于在 key 存在时删除 key                              | del itcast5          |
| 2    | DUMP key                 | 序列化给定 key ，并返回被序列化的值。                        | DUMP key1            |
| 3    | EXISTS key               | 检查给定 key 是否存在。                                      | exists itcast        |
| 4    | EXPIRE key               | seconds 为给定 key 设置过期时间，以秒计                      | expire itcast 5      |
| 5    | PEXPIRE key milliseconds | 设置 key 的过期时间以毫秒计                                  | PEXPIRE set2 3000000 |
| 6    | KEYS pattern             | 查找所有符合给定模式( pattern)的 key                         | keys *               |
| 7    | PERSIST key              | 移除 key 的过期时间，key 将持久保持                          | persist set2         |
| 11   | PTTL key                 | 以毫秒为单位返回 key 的剩余的过期时间                        | pttl  set2           |
| 12   | TTL key                  | 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。 | ttl set2             |
| 13   | RANDOMKEY                | 从当前数据库中随机返回一个 key                               | randomkey            |
| 14   | RENAME key newkey        | 修改 key 的名称                                              | rename  set5 set8    |
| 15   | RENAMENX key newkey      | 仅当 newkey 不存在时，将 key 改名为 newkey 。                | renamenx  set8 set10 |
| 16   | TYPE key                 | 返回 key 所储存的值的类型。                                  | type  set10          |



# 6.redis的javaAPI操作

redis不仅可以通过命令行进行操作，同时redis也可以通过javaAPI进行操作，我们可以通过使用javaAPI来对redis数据库当中的各种数据类型进行操作

## 6.1 创建maven工程并导入jar包

```xml

    <dependencies>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>6.14.3</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>6.14.3</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                    <!--    <verbal>true</verbal>-->
                </configuration>
            </plugin>
        </plugins>
    </build>
```

## 6.2 redi的API操作

```java
package jedis;

import org.testng.annotations.AfterTest;
import org.testng.annotations.BeforeTest;
import redis.clients.jedis.*;
import org.testng.annotations.Test;

import java.io.IOException;
import java.util.*;

/**
 * @Class:redis.jedis.jedisOperate
 * @Descript:
 * @Author:宋天
 * @Date:2020/1/14
 */
public class jedisOperate {
    private JedisPool jedisPool;
    /**
    * 连接redis
    * */
   @BeforeTest
    public void connetJedis(){
//        创建数据库连接池对象
        JedisPoolConfig jedis = new JedisPoolConfig();
//        设置连接redis的最大空闲数
        jedis.setMaxIdle(10);
//        设置连接redis超时时间
        jedis.setMaxWaitMillis(5000);
//        设置redis连接最大客户端数
        jedis.setMaxTotal(50);
//
        jedis.setMinIdle(5);
//        获取jedis连接池
        jedisPool = new JedisPool(jedis,"bigdata111",6379);
    }

    /**
     * String类型的数据操作
     */
    @Test
    public  void stringOperate(){
//        获取redis客户端
        Jedis jedis = jedisPool.getResource();
//        设置string类型的值
        jedis.set("jediskey","jedisvalue");
//        获取值
        String jediskey = jedis.get("jediskey");
        System.out.println(jediskey);
//        计数器
        jedis.incr("jincr");
        String jincr = jedis.get("jincr");
        System.out.println(jincr);

    }

    /**
     * hash表的操作
     */
    @Test
    public void hashOperate(){
        Jedis resource = jedisPool.getResource();
        resource.hset("jhsetkey","jmapkey","jmapValue");
        resource.hset("jhsetkey","jmapkey2","jmapvalue2");

        Map<String, String> jhsetkey = resource.hgetAll("jhsetkey");
        for (String s : jhsetkey.keySet()) {
            System.out.println("key值为" +  s);
            System.out.println(jhsetkey.get(s));
        }
        //修改数据
        resource.hset("jhsetkey", "jmapkey", "linevalue");
        //获取map当中所有key
        Set<String> jhsetkey1 = resource.hkeys("jhsetkey");
        for (String s : jhsetkey1) {
            System.out.println("所有的key为" +  s);
        }

        List<String> jhsetkey2 = resource.hvals("jhsetkey");
        for (String s : jhsetkey2) {
            System.out.println("所有的value值为" +  s);
        }


        //将jhsetkey这个数据给删掉
        resource.del("jhsetkey");
        //关闭客户端连接对象
        resource.close();


    }


    /**
     * list集合操作
     *
     */
    @Test
    public void listOperate(){
        Jedis resource = jedisPool.getResource();

        resource.lpush("listkey","listvalue1","listvalue2","listvalue3");

        List<String> listkey = resource.lrange("listkey", 0, 1);
        for (String s : listkey) {
            System.out.println(s);
        }
        System.out.println("===============xxxxxxxxxxxxxxxxxxxx===============");
        //从 右边弹出
        // String listkey1 = resource.rpop("listkey");
        // System.out.println(listkey1);


        String listkey2 = resource.lpop("listkey");
        System.out.println(listkey2);

        resource.close();



    }


    /**
     * 对set集合操作
     */
    @Test
    public void setOperate(){
        Jedis resource = jedisPool.getResource();

        //添加数据
        resource.sadd("setkey","setvalue1","setvalue2","setvalue3");

        //查看数据
        Set<String> setkey = resource.smembers("setkey");
        for (String line : setkey) {

            System.out.println(line);
        }
        resource.srem("setkey", "setvalue2");
        System.out.println("=============================");
        Set<String> setkey2 = resource.smembers("setkey");
        for (String line : setkey2) {
            System.out.println(line);
        }
    }


    /**
     * redis哨兵模式下的代码开发
     *
     */
    @Test
    public void testSentinel(){

        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMinIdle(5);
        jedisPoolConfig.setMaxTotal(30);
        jedisPoolConfig.setMaxWaitMillis(3000);
        jedisPoolConfig.setMaxIdle(10);

        HashSet<String> sentinels = new HashSet<>(Arrays.asList("bigdata111:26379", "bigdata222:26379", "bigdata333:26379"));
        /**
         * String masterName, Set<String> sentinels,
         final GenericObjectPoolConfig poolConfig
         */
        JedisSentinelPool sentinelPool = new JedisSentinelPool("mymaster", sentinels, jedisPoolConfig);
        //获取jedis客户端
        Jedis jedis = sentinelPool.getResource();
        jedis.set("sentinelKey","sentinelValue");
        jedis.close();
        sentinelPool.close();
    }


    /**
     * redis的集群
     */
    @Test
    public void redisCluster() throws IOException {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMinIdle(5);
        jedisPoolConfig.setMaxTotal(30);
        jedisPoolConfig.setMaxWaitMillis(3000);
        jedisPoolConfig.setMaxIdle(10);

        HashSet<HostAndPort> hostAndPorts = new HashSet<>();
        hostAndPorts.add(new HostAndPort("bigdata111",7001));
        hostAndPorts.add(new HostAndPort("bigdata111",7002));
        hostAndPorts.add(new HostAndPort("bigdata111",7003));
        hostAndPorts.add(new HostAndPort("bigdata111",7004));
        hostAndPorts.add(new HostAndPort("bigdata111",7005));
        hostAndPorts.add(new HostAndPort("bigdata111",7006));
        //获取jedis  cluster
        JedisCluster jedisCluster = new JedisCluster(hostAndPorts, jedisPoolConfig);

        jedisCluster.set("test","abc");

        String test = jedisCluster.get("test");
        System.out.println(test);

        jedisCluster.close();

    }

    @AfterTest
    public void jedisPoolClose(){
        jedisPool.close();
    }

}

```

# 7. redis的持久化

由于redis是一个内存数据库，所有的数据都是保存在内存当中的，内存当中的数据极易丢失，所以redis的数据持久化就显得尤为重要，在redis当中，提供了两种数据持久化的方式，分别为RDB以及AOF，且redis默认开启的数据持久化方式为RDB方式，接下来我们就分别来看下两种方式的配置

## 7.1 RDB持久化方案

### 7.1.1 RDM方案介绍

redis会定期保存数据快照至一个rbd文件中，并在启动时自动加载rdb文件，恢复之前保存的数据。可以在配置文件中配置redis进行快照保存的时机

**save [seconds] [changes]**

意为在[seconds]秒内如果发生了[changes]次数据修改，则进行一次RDB快照保存，例如

```
save 60 100
```

会让Redis每60秒检查一次数据变更情况，如果发生了100次或以上的数据变更，则进行RDB快照保存。可以配置多条save指令，让Redis执行多级的快照保存策略。**Redis默认开启RDB快照。**也可以通过SAVE或者BGSAVE命令手动触发RDB快照保存。

**SAVE 和 BGSAVE 两个命令都会调用 rdbSave 函数，**但它们调用的方式各有不同：

- SAVE 直接调用 rdbSave ，阻塞 Redis 主进程，直到保存完成为止。在主进程阻塞期间，服务器不能处理客户端的任何请求。
- BGSAVE 则 fork 出一个子进程，子进程负责调用 rdbSave ，并在保存完成之后向主进程发送信号，通知保存已完成。 Redis 服务器在BGSAVE 执行期间仍然可以继续处理客户端的请求。

### 7.1.2 RDB方案优点

1. 对性能影响最小。如前文所述，Redis在保存RDB快照时会fork出子进程进行，几乎不影响Redis处理客户端请求的效率。
2.  每次快照会生成一个完整的数据快照文件，所以可以辅以其他手段保存多个时间点的快照（例如把每天0点的快照备份至其他存储媒介中），作为非常可靠的灾难恢复手段。
3.  使用RDB文件进行数据恢复比使用AOF要快很多

### 7.1.3 RDB方案缺点

- 快照是定期生成的，所以在Redis crash时或多或少会丢失一部分数据。
- 如果数据集非常大且CPU不够强（比如单核CPU），Redis在fork子进程时可能会消耗相对较长的时间，影响Redis对外提供服务的能力。

### 7.1.4 方案配置

修改redis.conf配置文件

```
save 900 1
save 300 10
save 60 10000
save 5 1
```

重新启动redis服务

每次生成新的dump.rdb都会覆盖掉之前的老的快照

```
ps -ef | grep redis
kill -9 69632 74217
src/redis-server redis.conf
```

## 7.2 AOF持久化方案

### 7.2.1 AOF方案介绍

采用AOF持久方式时，Redis会把每一个写请求都记录在一个日志文件里。在Redis重启时，会把AOF文件中记录的所有写操作顺序执行一遍，确保数据恢复到最新。AOF默认是关闭的，如要开启，进行如下配置：

```
appendonly yes
```

AOF提供了三种fsync配置，always/everysec/no，通过配置项[appendfsync]指定：

- appendfsync no：不进行fsync，将flush文件的时机交给OS决定，速度最快appendfsync always：每写入一条日志就进行一次fsync操作，数据安全性最高，但速度最慢

- appendfsync everysec：折中的做法，交由后台线程每秒fsync一次

  随着AOF不断地记录写操作日志，因为所有的操作都会记录，所以必定会出现一些无用的日志。大量无用的日志会让AOF文件过大，也会让数据恢复的时间过长。不过Redis提供了AOF rewrite功能，可以重写AOF文件，只保留能够把数据恢复到最新状态的最小写操作集。

- AOF rewrite可以通过BGREWRITEAOF命令触发，也可以配置Redis定期自动进行：

  auto-aof-rewrite-percentage 100auto-aof-rewrite-min-size 64mb

  上面两行配置的含义是，Redis在每次AOF rewrite时，会记录完成rewrite后的AOF日志大小，当AOF日志大小在该基础上增长了100%后，自动进行AOF rewrite。同时如果增长的大小没有达到64mb，则不会进行rewrite。

###  7.2.2 AOF优点

1.  最安全，在启用appendfsync always时，任何已写入的数据都不会丢失，使用在启用appendfsync everysec也至多只会丢失1秒的数据
2.  AOF文件在发生断电等问题时也不会损坏，即使出现了某条日志只写入了一半的情况，也可以使用redis-check-aof工具轻松修复。
3. AOF文件易读，可修改，在进行了某些错误的数据清除操作后，只要AOF文件没有rewrite，就可以把AOF文件备份出来，把错误的命令删除，然后恢复数据。

### 7.2.3 AOF缺点

1.  AOF文件通常比RDB文件更大
2.  性能消耗比RDB高
3. 数据恢复速度比RDB慢

- Redis的数据持久化工作本身就会带来延迟，需要根据数据的安全级别和性能要求制定合理的持久化策略：

- AOF + fsync always的设置虽然能够绝对确保数据安全，但每个操作都会触发一次fsync，会对Redis的性能有比较明显的影响

- AOF + fsync every second是比较好的折中方案，每秒fsync一次

- AOF + fsync never会提供AOF持久化方案下的最优性能使用RDB持久化通常会提供比使用AOF更高的性能，但需要注意RDB的策略配置
- 每一次RDB快照和AOF Rewrite都需要Redis主进程进行fork操作。fork操作本身可能会产生较高的耗时，与CPU和Redis占用的内存大小有关。根据具体的情况合理配置RDB快照和AOF Rewrite时机，避免过于频繁的fork带来的延迟
- Redis在fork子进程时需要将内存分页表拷贝至子进程，以占用了24GB内存的Redis实例为例，共需要拷贝24GB / 4kB * 8 = 48MB的数据。在使用单Xeon 2.27Ghz的物理机上，这一fork操作耗时216ms。

### 7.2.4 AOF方案配置

在redis中，aof的持久化机制默认是关闭的

**AOF持久化，默认是关闭的，默认是打开RDB持久化**

**`appendonly yes`，可以打开AOF持久化机制，**在生产环境里面，一般来说AOF都是要打开的，除非你说随便丢个几分钟的数据也无所谓

打开AOF持久化机制之后，redis每次接收到一条写命令，就会写入日志文件中，当然是先写入os cache的，然后每隔一定时间再fsync一下

而且即使AOF和RDB都开启了，redis重启的时候，也是优先通过AOF进行数据恢复的，因为aof数据比较完整

可以配置AOF的fsync策略，有三种策略可以选择，

- 一种是每次写入一条数据就执行一次fsync; 
- 一种是每隔一秒执行一次fsync; 
- 一种是不主动执行fsync

always: 每次写入一条数据，立即将这个数据对应的写日志fsync到磁盘上去，性能非常非常差，吞吐量很低; 确保说redis里的数据一条都不丢，那就只能这样了

在redis当中默认的AOF持久化机制都是关闭的

 **配置redis的AOF持久化机制方式**

vim redis.conf

```
appendonly yes
```



# 8.redis的主从复制架构

在Redis中，用户可以通过执行SLAVEOF命令或者设置slaveof选项，让一个服务器去复制（replicate）另一个服务器，我们称呼被复制的服务器为主服务器（master），而对主服务器进行复制的服务器则被称为从服务器（slave），如下所示。

![](img\redis\redis的主从复制架构.png)

使用主从复制这种模式，实现bigdata111作为主节点，bigdata222与bigdata333作为从节点，并且将bigdata111所有的数据全部都同步到bigdata222与bigdata333服务器





**以下皆为bigdata222与bigdata333同时操作**

1. 安装依赖环境

   ```
   yum -y install gcc-c++
   ```

2. 上传redis压缩包并解压

   ```
   tar -zxvf redis-3.2.8.tar.gz -C /opt/module
   ```

3. 安装tcl

   ```
   yum  -y  install  tcl
   ```

4. 编译redis

   ```
   make MALLOC=libc  
   ```

   注：建议手敲，不建议复制粘贴

5. 在安装目录的根目录下创建文件夹

   ```
   mkdir ogs
   mkdir redisdata
   ```

   修改redis.conf配置文件

   bigdata222

   ```
   bind bigdata222
   daemonize yes
   pidfile /var/run/redis_6379.pid
   logfile "/opt/module/redis-3.2.8/logs/redis.log"
   dir /opt/module/redis-3.2.8/redisdata
   slaveof bigdata222 6379
   ```

   bigdata333

   ```
   bind bigdata333
   daemonize yes
   pidfile /var/run/redis_6379.pid
   logfile "/opt/module/redis-3.2.8/logs/redis.log"
   dir /opt/module/redis-3.2.8/redisdata
   slaveof bigdata333 6379
   ```

6. 启动bigdata222与bigdata333机器的redis服务

   ```
   src/redis-server  ../redis.conf
   ```

启动成功便可以实现redis的主从复制，**bigdata111可以读写操作，bigdata222与bigdata333只支持读取操作。**

# 9.redis当中的Sentinel架构

Sentinel（哨兵）是Redis 的高可用性解决方案：由一个或多个Sentinel 实例 组成的Sentinel 系统可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器。

例如：

![](img\redis\redis当中的Sentinel架构.png)

在Server1 掉线后：

![](img\redis\在Server1 掉线后.png)

升级Server2 为新的主服务器：

![](img\redis\升级Server2 为新的主服务器.png)

## 9.1 redis的Sentinel配置

**以下皆为三台服务器同时操作**

1. 在redis的根目录下有如下配置文件

   vim sentinel.conf

   ```
   # 配置监听的主服务器，这里sentinel monitor代表监控，mymaster代表服务器的名称，可以自定义，192.168.11.128代表监控的主服务器，6379代表端口，2代表只有两个或两个以上的哨兵认为主服务器不可用的时候，才会进行failover操作。
   #修改bind配置，每台机器修改为自己对应的主机名
   bind bigdata111  
   #配置sentinel服务后台运行
   daemonize yes
   #修改三台机器监控的主节点，现在主节点是bigdata111服务器
   sentinel monitor mymaster bigdata111 6379 2
   # sentinel author-pass定义服务的密码，mymaster是服务名称，123456是Redis服务器密码
   # sentinel auth-pass <master-name> <password>
   ```

2. 三台机器启动哨兵服务

   ```
   src/redis-sentinel sentinel.conf 
   ```

3. bigdata111服务器杀死redis服务进程

   使用kill  -9命令杀死redis服务进程，模拟redis故障宕机情况

   过一段时间之后，就会在bigdata222与bigdata333服务器选择一台服务器来切换为主节点

## 9.2 redis的sentinel模式代码开发连接

```java
/**
     * redis哨兵模式下的代码开发
     *
     */
    @Test
    public void testSentinel(){

        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMinIdle(5);
        jedisPoolConfig.setMaxTotal(30);
        jedisPoolConfig.setMaxWaitMillis(3000);
        jedisPoolConfig.setMaxIdle(10);

        HashSet<String> sentinels = new HashSet<>(Arrays.asList("bigdata111:26379", "bigdata222:26379", "bigdata333:26379"));
        /**
         * String masterName, Set<String> sentinels,
         final GenericObjectPoolConfig poolConfig
         */
        JedisSentinelPool sentinelPool = new JedisSentinelPool("mymaster", sentinels, jedisPoolConfig);
        //获取jedis客户端
        Jedis jedis = sentinelPool.getResource();
        jedis.set("sentinelKey","sentinelValue");
        jedis.close();
        sentinelPool.close();
    }

```

# 10. redis集群

## 10.1 redis集群介绍

- Redis 集群是一个提供在多个Redis节点之间共享数据的程序集。

- Redis 集群并不支持同时处理多个键的 Redis 命令，因为这需要在多个节点间移动数据，这样会降低redis集群的性能，在高负载的情况下可能会导致不可预料的错误。

- Redis 集群通过分区来提供一定程度的可用性，即使集群中有一部分节点失效或者无法进行通讯， 集群也可以继续处理命令请求。

**Redis 集群的优势:**

1. 缓存永不宕机：启动集群，永远让集群的一部分起作用。主节点失效了子节点能迅速改变角色成为主节点，整个集群的部分节点失败或者不可达的情况下能够继续处理命令；
2. 迅速恢复数据：持久化数据，能在宕机后迅速解决数据丢失的问题；
3. Redis可以使用所有机器的内存，变相扩展性能；
4. 使Redis的计算能力通过简单地增加服务器得到成倍提升,Redis的网络带宽也会随着计算机和网卡的增加而成倍增长；
5. Redis集群没有中心节点，不会因为某个节点成为整个集群的性能瓶颈;
6. 异步处理数据，实现快速读写。

![](img\redis\redis集群.png)

##  10.2 redis集群环境搭建

由于redis集群当中最少需要三个主节点，每个主节点，最少需要一个对应的从节点，所以搭建redis集群最少需要三主三从的配置，所以redis集群最少需要6台redis的实例，我们这里使用三台机器，每台服务器上面运行两个redis的实例。我们这里使用bigdata111服务器，通过配置不同的端口，实现redis集群的环境搭建

1. bigdata111服务器解压redis压缩包

   ```
   tar -zxf redis-3.2.8.tar.gz -C /opt/module
   ```

2. 安装redis必须依赖环境并进行编译

   ```
   yum -y install gcc-c++ tcl
   make MALLOC=libc 
   ```

3. 创建redis不同实例的配置文件夹

   创建文件夹，并将redis的配置文件拷贝到以下这些目录

   ```
   cd /opt/module/redis-3.2.8
   mkdir -p  /opt/module/redis-3.2.8/clusters/7001
   mkdir -p  /opt/module/redis-3.2.8/clusters/7002
   mkdir -p  /opt/module/redis-3.2.8/clusters/7003
   mkdir -p  /opt/module/redis-3.2.8/clusters/7004
   mkdir -p  /opt/module/redis-3.2.8/clusters/7005
   mkdir -p  /opt/module/redis-3.2.8/clusters/7006
   ```

4. 修改redis的六个配置文件

   ```
   mkdir -p /opt/module/redis-3.2.8/logs
   mkdir -p /opt/module/redis-3.2.8/redisdata/7001
   mkdir -p /opt/module/redis-3.2.8/redisdata/7002
   mkdir -p /opt/module/redis-3.2.8/redisdata/7003
   mkdir -p /opt/module/redis-3.2.8/redisdata/7004
   mkdir -p /opt/module/redis-3.2.8/redisdata/7005
   mkdir -p /opt/module/redis-3.2.8/redisdata/7006
   ```

5. 第一个配置文件修改

   vim  /export/redis-3.2.8/redis.conf

   ```
   bind bigdata111
   port 7001
   cluster-enabled yes
   cluster-config-file nodes-7001.conf
   cluster-node-timeout 5000
   appendonly yes
   daemonize yes
   pidfile /var/run/redis_7001.pid
   logfile "/opt/module/redis-3.2.8/logs/7001.log"
   dir /opt/module/redis-3.2.8/redisdata/7001
   ```

6. 将修改后的文件拷贝到对应的文件夹下面去

   ```
   cp /opt/module/redis-3.2.8/redis.conf /opt/module/redis-3.2.8/clusters/7001
   ```

7. 第二个配置文件修改

   ```
   vim  /export/redis-3.2.8/clusters/7002/redis.conf
   ```

   ```
   bind bigdata111
   port 7002
   cluster-enabled yes
   cluster-config-file nodes-7002.conf
   cluster-node-timeout 5000
   appendonly yes
   daemonize yes
   pidfile /var/run/redis_7002.pid
   logfile "/opt/module/redis-3.2.8/logs/7002.log"
   dir /opt/module/redis-3.2.8/redisdata/7002
   ```

8. 其他配置文件同上

   注意更改每个文件夹下的以下配置即可

   ```
   port 7005
   cluster-config-file nodes-7005.conf
   pidfile /var/run/redis_7005.pid
   logfile "/opt/module/redis-3.2.8/logs/7005.log"
   dir /opt/module/servers/redis-3.2.8/redisdata/7005
   ```

9. 启动redis进程

   ```
   cd /opt/module/redis-3.2.8
   src/redis-server clusters/7001/redis.conf
   src/redis-server clusters/7002/redis.conf
   src/redis-server clusters/7003/redis.conf
   src/redis-server clusters/7004/redis.conf
   src/redis-server clusters/7005/redis.conf
   src/redis-server clusters/7006/redis.conf
   ```

10. 安装ruby运行环境

    redis集群的启动需要借助ruby的环境

    ```
    yum install ruby
    yum install rubygems
    gem install redis
    ```

    这个时候可能会报错，这时，需要升级Ruby版本

    bigdata111执行以下命令升级ruby版本

    ```
    cd /opt/module/redis-3.2.8
    gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
    
    curl -sSL https://get.rvm.io | bash -s stable
    source /etc/profile.d/rvm.sh
    rvm list known
    rvm install 2.4.1
    ```

11. 创建redis集群

    ```
    cd /opt/module/redis-3.2.8
    gem install redis
    src/redis-trib.rb create --replicas 1 192.168.100.100:7001 192.168.100.100:7002 192.168.100.100:7003 192.168.100.100:7004 192.168.100.100:7005 192.168.100.100:7006
    ```

12. 连接redis客户端

    ```
    cd /opt/module/redis-3.2.8
    src/redis-cli  -h node01 -c -p 7001
    ```

注：集群搭建并未测试，没啥必要。

## 10.3 redis集群管理

添加一个新节点作为主节点

启动新节点的redis服务，然后添加到集群当中去

启动服务

```
src/redis-server clusters/7007/redis.conf
src/redis-trib.rb add-node 192.168.100.100:7007 192.168.100.100:7001

```

添加一个新节点作为副本

```
src/redis-server clusters/7008/redis.conf
src/redis-trib.rb add-node --slave 192.168.100.100:7008 192.168.100.100:7001
```

删除一个节点

命令格式

```
src/redis-trib del-node 127.0.0.1:7000 `<node-id>`
```

```
src/redis-trib.rb del-node 192.168.100.100:7008 7c7b7f68bc56bf24cbb36b599d2e2d97b26c5540
```

重新分片

```
./redis-trib.rb reshard bigdata111:7001
./redis-trib.rb reshard --from <node-id> --to <node-id> --slots <number of slots> --yes <host>:<port>
```

## 10.4 JavaAPI操作redis集群

```java

    /**
     * redis的集群
     */
    @Test
    public void redisCluster() throws IOException {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMinIdle(5);
        jedisPoolConfig.setMaxTotal(30);
        jedisPoolConfig.setMaxWaitMillis(3000);
        jedisPoolConfig.setMaxIdle(10);

        HashSet<HostAndPort> hostAndPorts = new HashSet<>();
        hostAndPorts.add(new HostAndPort("bigdata111",7001));
        hostAndPorts.add(new HostAndPort("bigdata111",7002));
        hostAndPorts.add(new HostAndPort("bigdata111",7003));
        hostAndPorts.add(new HostAndPort("bigdata111",7004));
        hostAndPorts.add(new HostAndPort("bigdata111",7005));
        hostAndPorts.add(new HostAndPort("bigdata111",7006));
        //获取jedis  cluster
        JedisCluster jedisCluster = new JedisCluster(hostAndPorts, jedisPoolConfig);

        jedisCluster.set("test","abc");

        String test = jedisCluster.get("test");
        System.out.println(test);

        jedisCluster.close();

    }

    @AfterTest
    public void jedisPoolClose(){
        jedisPool.close();
    }

```

