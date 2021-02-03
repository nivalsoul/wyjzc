# Hadoop
## hdfs读写数据流程
**HDFS读数据流程**
- 1、与NameNode通信查询元数据，找到文件块所在的DataNode服务器
- 2、挑选一台DataNode（网络拓扑上的就近原则，如果都一样，则随机挑选一台DataNode）服务器，请求建立socket流
- 3、DataNode开始发送数据(从磁盘里面读取数据放入流，以packet（一个packet为64kb）为单位来做校验)
- 4、客户端以packet为单位接收，先在本地缓存，然后写入目标文件
![avatar](https://image-static.segmentfault.com/289/499/2894999463-5aaa907f52ca0_articlex)

**HDFS写数据流程**
- 1、跟NameNode通信请求上传文件，NameNode检查目标文件是否已经存在，父目录是否已经存在
- 2、NameNode返回是否可以上传
- 3、Client先对文件进行切分，请求第一个block该传输到哪些DataNode服务器上
- 4、NameNode返回3个DataNode服务器DataNode 1，DataNode 2，DataNode 3
- 5、Client请求3台中的一台DataNode 1(网络拓扑上的就近原则，如果都一样，则随机挑选一台DataNode)上传数据（本质上是一个RPC调用，建立pipeline）,DataNode 1收到请求会继续调用DataNode 2,然后DataNode 2调用DataNode 3，将整个pipeline建立完成，然后逐级返回客户端
- 6、Client开始往DataNode 1上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以pocket为单位。写入的时候DataNode会进行数据校验，它并不是通过一个packet进行一次校验而是以chunk为单位进行校验（512byte）。DataNode 1收到一个packet就会传给DataNode 2，DataNode 2传给DataNode 3，DataNode 1每传一个pocket会放入一个应答队列等待应答
- 7、当一个block传输完成之后，Client再次请求NameNode上传第二个block的服务器.
![avatar](https://img2018.cnblogs.com/blog/699090/201906/699090-20190626155745864-1227676006.png)


## MapReduce原理
原理、和spark的区别

## shuffle机制

## Hadoop任务的Yarn调度过程 


# Hive

## hive优化

## Mapjoin原理
- MapJoin顾名思义，就是在Map阶段进行表之间的连接，map阶段直接拿另外一个表的数据和内存中表数据做匹配。而不需要进入到Reduce阶段才进行连接。这样就节省了在Shuffle阶段时要进行的大量数据传输。从而起到了优化作业的作用。
- MapJoin通常用于一个很小的表和一个大表进行join的场景，具体小表有多小，由参数 hive.mapjoin.smalltable.filesize来决定，该参数表示小表的总大小，默认值为25000000字节，即25M
- Hive0.7之前，需要使用hint提示* /+ mapjoin(table) */才会执行MapJoin,否则执行Common Join，但在0.7版本之后，默认自动会转换Map Join，由数 **hive.auto.convert.join来控制，默认为true
- 执行流程：
  - 首先是Task A，它是一个Local Task（在客户端本地执行的Task），负责扫描小表b的数据，将其转换成一个HashTable的数据结构，并写入本地的文件中，之后将该文件上传到hadoop的DistributeCache中
  -  接下来是Task B，该任务是一个没有Reduce的MR，启动MapTasks扫描大表a,在Map阶段，根据a的每一条记录去和DistributeCache中b表对应的HashTable关联，并直接输出结果
  -  由于MapJoin没有Reduce，所以由Map直接输出结果文件，有多少个Map Task，就有多少个结果文件

## 数据倾斜
### 场景
- join
- count(distinct)
- group by
- 一些窗口函数中用了partition by

### 解决方案
**大表和小表join产生的数据倾斜**
- 使用mapjoin（高版本默认开启）
  - set hive.auto.convert.join=true; 
  - set hive.mapjoin.smalltable.filesize=25000000; 
- 对于小表join大表的笛卡尔乘积，还可以通过规避的方法避免：具体比如给 Join的两个表都增加一列Join key，原理很简单：将小表扩充一列join key，并将小表的总数复制数倍，join key 各不相同，比如第一次为1，复制一次joinkey为2，依次类推；将大表扩充一列join key 为随机数，这个随机数为小表里的joinkey的随机值，如1-5的随机值。

**大表和大表的join产生的数据倾斜**
- 大表与大表关联，但是其中一张表的空值或者0比较多，容易shuffle给一个reduce，造成运行慢，这种情况可以对异常值赋一个随机值来分散key,均匀分配给多个reduce去执行，或者将异常值单独处理然后union all。注：如果这个字段不是join、group by 的条件，就不会因为它而产生倾斜。
- 当key值都是有效值时，解决办法为设置以下几个参数：
  - set hive.exec.reducers.bytes.per.reducer = 1000000000; 也就是每个节点的reduce 默认是处理1G大小的数据，如果你的join 操作也产生了数据倾斜，那么你可以在hive 中设定
  - set hive.optimize.skewjoin = true; set hive.skewjoin.key = skew_key_threshold （default = 100000）
  - hive 在运行的时候没有办法判断哪个key 会产生多大的倾斜，所以使用这个参数控制倾斜的阈值，如果超过这个值，新的值会发送给那些还没有达到的reduce, 一般可以设置成你处理的总记录数/reduce个数的2-4倍都可以接受

**group by 造成的数据倾斜**
设置参数：
hive.map.aggr=true  （默认true） 这个配置项代表是否在map端进行聚合，相当于Combiner
hive.groupby.skewindata=true（默认false）

**count(distinct)以及其他参数不当的数据倾斜**
- 设置的reduce个数太少：可以增加数量，set mapred.reduce.tasks=800; 默认是先设置hive.exec.reducers.bytes.per.reducer这个参数，设置了后hive会自动计算reduce的个数，因此两个参数一般不同时使用
- 当HiveQL中包含count（distinct）时：如果数据量非常大，执行如select a,count(distinct b) from t group by a;类型的SQL时，会出现数据倾斜的问题。解决方法：使用sum...group by代替。如select a,sum(1) from (select a, b from t group by a,b) group by a;

**窗口函数中partition by造成的数据倾斜**
场景可能会和上述情况类似，需要具体分析

## UDF
**UDF**
含义：返回对应值，一对一，操作作用于单个数据行，并且产生一个数据行作为输出。大多数函数都属于这一类（比如数学函数和字符串函数）
实现方式：
- 1.继承 UDF ， 重写evaluate方法
- 2.继承 GenericUDF，重写initialize、getDisplayString、evaluate方法

**UDAF**
含义：返回聚类值，多对一，接受多个输入数据行，并产生一个输出数据行。像COUNT和MAX这样的函数就是聚集函数。
实现方式：自定义一个java类继承UDAF类，内部定义一个静态类，实现UDAFEvaluator接口，实现方法init,iterate,terminatePartial,merge,terminate

**UDTF**
含义：返回拆分值，一对多，操作作用于单个数据行，并且产生多个数据行（一个表作为输出），比如lateral view explore()
实现方式：继承GenericUDTF,实现initialize, process, close三个方法

## 4个By区别
- 1）Sort By：分区内有序，只会在每个reducer中对数据进行排序，可以指定执行的reduce个数（set mapred.reduce.tasks=xx）； 
- 2）Order By：全局排序，只有一个 Reducer； 
- 3）Distrbute By：类似 MR 中 Partition，进行分区，结合 sort by 使用。 
- 4）Cluster By：当Distribute by和Sort by字段相同时，可以使用Cluster by方式。但是排序只能是升序排序，不能指定排序规则为 ASC 或者 DESC。

## 常用函数
**row_number()、rank()和dense_rank()**
- row_number:不管排名是否有相同的，都按照顺序：1，2，3…..n
- rank:排名相同的名次一样，同一排名有几个，后面排名就会跳过几次： 1,2,2,4,5...
- dense_rank:排名相同的名次一样，且后面名次不跳跃：1,2,2,3,4...

**Lag, Lead, first_value,last_value**
- LAG(col,n,DEFAULT) 用于统计窗口内往上第n行值
- LEAD(col,n,DEFAULT) 用于统计窗口内往下第n行值, 与LAG相反
- first_value:  取分组内排序后，截止到当前行，第一个值
- last_value:  取分组内排序后，截止到当前行，最后一个值

**ntile**
ntile(n) over(partition by x order by y)：用于将分组数据按照顺序切分成n片，返回当前切片值,如果不能平均分配，则优先分配较小编号的桶，并且各个桶中能放的行数最多相差1，不支持ROWS BETWEEN。



# hbase
## 存储结构

## rowkey设计原则
- Rowkey的长度原则
- Rowkey的唯一原则
- Rowkey的排序原则
- Rowkey的散列原则
**热点问题**
- Reverse反转：针对固定长度的Rowkey反转后存储，这样可以使Rowkey中经常改变的部分放在最前面，可以有效的随机Rowkey。
- Salt加盐：Salt是将每一个Rowkey加一个前缀，前缀使用一些随机字符，使得数据分散在多个不同的Region，达到Region负载均衡的目标。
- Hash散列或者Mod：用Hash散列来替代随机Salt前缀的好处是能让一个给定的行有相同的前缀，这在分散了Region负载的同时，使读操作也能够推断。


# spark
## 基本架构原理
**spark生态架构图**
![avatar](https://images2015.cnblogs.com/blog/1004194/201608/1004194-20160829161404996-1972748563.png)
**运行流程**
![avatar](https://images2015.cnblogs.com/blog/1004194/201608/1004194-20160830094200918-1846127221.png)
- 构建Spark Application的运行环境，启动SparkContext
- SparkContext向资源管理器（可以是Standalone，Mesos，Yarn）申请运行Executor资源，并启动StandaloneExecutorbackend，
- Executor向SparkContext申请Task
- SparkContext将应用程序分发给Executor
- SparkContext构建成DAG图，将DAG图分解成Stage、将Taskset发送给Task Scheduler，最后由Task Scheduler将Task发送给Executor运行
- Task在Executor上运行，运行完释放所有资源
**运行架构特点**
多线程运行、运行过程与资源管理器无关、Task采用了数据本地性和推测执行来优化。

## RDD
**基本概念**
RDD 是 Spark 提供的最重要的抽象概念，它是一种有容错机制的特殊数据集合，可以分布在集群的结点上，以函数式操作集合的方式进行各种并行操作。通俗点来讲，可以将 RDD 理解为一个分布式对象集合，本质上是一个只读的分区记录集合。每个 RDD 可以分成多个分区，每个分区就是一个数据集片段。一个 RDD 的不同分区可以保存到集群中的不同结点上，从而可以在集群中的不同结点上进行并行计算。
RDD的特点：
- 只读：不能修改，只能通过转换操作生成新的 RDD。
- 分布式：可以分布在多台机器上进行并行处理。
- 弹性：计算过程中内存不够时它会和磁盘进行数据交换。
- 基于内存：可以全部或部分缓存在内存中，在多次计算间重用。

**生成方式**
- 使用SparkContext的parallelize()方法序列化本地数据集合创建RDD：sc.makeRDD或者sc.parallelize
- 使用外界的数据源创建RDD，比如说本地文件系统，分布式文件系统HDFS等等：sc.textFile("local path或者HDFS path")
- 通过将已有RDD使用transform算子操作产生新的RDD：rdd.map/rdd.reduceByKey等

**RDD、DataFrame、DataSet**
- RDD是分布式的Java对象的集合
- DataFrame是一种以RDD为基础的分布式数据集，类似于传统数据库中的二维表格，比RDD多了数据的结构信息，即schema
- Dataset可以认为是DataFrame的一个特例，主要区别是Dataset每一个record存储的是一个强类型值而不是一个Row。在spark2.x，DataFrame和Dataset的api进行了统一，所以都可以采用DSL和SQL方式进行开发，都可以通过sparksession对象进行创建或者是通过transform转化操作得到

**算子** 
对RDD的操作分为transformation和action两类，真正的作业提交运行发生在action之后，调用action之后会将对原始输入数据的所有transformation操作封装成作业并向集群提交运行。
- transformation：RDD 的转换操作是返回新的 RDD 的操作。转换出来的 RDD 是惰性求值的，只有在行动操作中用到这些 RDD 时才会被计算
- action：行动操作用于执行计算并按指定的方式输出结果。行动操作接受 RDD，但是返回非 RDD，即输出一个值或者结果。在 RDD 执行过程中，真正的计算发生在行动操作。
- 会引起Shuffle过程的Spark算子
  - 1、repartition类的操作：比如repartition、repartitionAndSortWithinPartitions、coalesce等，一般会shuffle，因为需要在整个集群中，对之前所有的分区的数据进行随机，均匀的打乱，然后把数据放入下游新的指定数量的分区内
  - 2、byKey类的操作：比如reduceByKey、groupByKey、sortByKey等，因为要对一个key，进行聚合操作，那么肯定要保证集群中，所有节点上的，相同的key，一定是到同一个节点上进行处理
  - 3、join类的操作：比如join、cogroup等，两个rdd进行join，就必须将相同join key的数据，shuffle到同一个节点上，然后进行相同key的两个rdd数据的笛卡尔乘积
  - 4、去重：distinct
  - 5、集合操作：intersection、subtract等


**宽窄依赖**
![avatar](https://img-blog.csdn.net/20170803212113749?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWxidW1fZ3lk/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
- 宽依赖：父RDD的分区被子RDD的多个分区使用，例如 groupByKey、reduceByKey、sortByKey等操作会产生宽依赖，会产生shuffle
- 窄依赖：父RDD的每个分区都只被子RDD的一个分区使用，例如map、filter、union等操作会产生窄依赖
- join操作有两种情况：如果两个RDD在进行join操作时，一个RDD的partition仅仅和另一个RDD中已知个数的Partition进行join，那么这种类型的join操作就是窄依赖；其它情况的join操作就是宽依赖，由于是需要父RDD的所有partition进行join的转换，这就涉及到了shuffle，因此这种类型的join操作也是宽依赖。

**Stage划分**
stage的划分是Spark作业调度的关键一步，它基于DAG确定依赖关系，借此来划分stage，将依赖链断开，每个stage内部可以并行运行，整个作业按照stage顺序依次执行，最终完成整个Job。实际应用提交的Job中RDD依赖关系是十分复杂的，依据这些依赖关系来划分stage自然是十分困难的，Spark此时就利用了前文提到的依赖关系，调度器从DAG图末端出发，逆向遍历整个依赖关系链，遇到ShuffleDependency（宽依赖关系的一种叫法）就断开，遇到NarrowDependency就将其加入到当前stage。stage中task数目由stage末端的RDD分区个数来决定，RDD转换是基于分区的一种粗粒度计算，一个stage执行的结果就是这几个分区构成的RDD。也就是：遇到一个宽依赖就分一个stage

## Shuffle
stage中是高效快速的pipline的计算模式，宽依赖之间会划分stage，而Stage之间就是Shuffle
在Spark的中，负责shuffle过程的执行、计算和处理的组件主要就是ShuffleManager，也即shuffle管理器。ShuffleManager随着Spark的发展有两种实现的方式，分别为HashShuffleManager和SortShuffleManager，因此spark的Shuffle有Hash Shuffle和Sort Shuffle两种


## 缓存机制
RDD 持久化是 Spark 非常重要的特性之一。用户可显式将一个 RDD 持久化到内存或磁盘中，以便重用该RDD。RDD 持久化是一个分布式的过程，其内部的每个 Partition 各自缓存到所在的计算节点上。RDD 持久化存储能大大加快数据计算效率，尤其适合迭代式计算和交互式计算。
Spark 提供了 persist 和 cache 两个持久化函数，其中 cache 将 RDD 持久化到内存中，而 persist 则支持多种存储级别。但是并不是这两个方法被调用时立即缓存，而是触发后面的action时，该RDD将会被缓存在计算节点的内存中，并供后面重用。通过过查看源码发现cache最终也是调用了persist方法，默认的存储级别都是仅在内存存储一份，Spark的存储级别还有好多种，存储级别在object StorageLevel中定义的。

## 共享变量
spark中的共享变量有：累加器、广播变量

## 数据倾斜
数据倾斜 是 某些 task 处理了大量数据，所以数据倾斜很可能引起 OOM，数据倾斜和 OOM 的某些解决办法可以通用；
解决数据倾斜的大致思路为：
1. 避免 shuffle，可在数据输入 spark 之前进行 shuffle，或者 用 map 等替代 shuffle
　　// 聚合原始数据、广播小 RDD
2. 如果避免不了 shuffle，就减少 reduce task 的数据量
　　// 缩小 key 粒度、增加 reduce task 数量
　　// 通过随机数多次聚合，减少每次聚合的数据量，针对 reduceByKey、groupByKey 等
3. 其他情况，单个 key 数据量大，多个 key 数据量大，针对 join

## SparkSQL
### 特点
- 数据兼容：不仅兼容Hive，还可以从RDD、parquet文件、Json文件获取数据、支持从RDBMS获取数据
- 性能优化：采用内存列式存储、自定义序列化器等方式提升性能；
- 组件扩展：SQL的语法解析器、分析器、优化器都可以重新定义和扩展
- 兼容: Hive兼容层面仅依赖HiveQL解析、Hive元数据。
- 从HQL被解析成抽象语法树（AST）起，就全部由Spark SQL接管了，Spark SQL执行计划生成和优化都由Catalyst（函数式关系查询优化框架）负责
- 支持: 数据既可以来自RDD，也可以是Hive、HDFS、Cassandra等外部数据源，还可以是JSON格式的数据；
- Spark SQL目前支持Scala、Java、Python三种语言，支持SQL-92规范；

### 作用
- Spark 中用于处理结构化数据的模块；
- 相对于RDD的API来说，提供更多结构化数据信息和计算方法
- 可以通过SQL或DataSet API方式同Spark SQL进行交互

### Spark SQL架构图
![avatar](https://img-blog.csdnimg.cn/20191015082539972.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NzEwOTAw,size_16,color_FFFFFF,t_70)

## Spark Streaming
SparkStreaming是一套框架。
SparkStreaming是Spark核心API的一个扩展，可以实现高吞吐量的，具备容错机制的实时流数据处理。
支持多种数据源获取数据：
![avatar](https://img2018.cnblogs.com/blog/1612915/201902/1612915-20190227162849081-482876303.png)
Spark Streaming接收Kafka、Flume、HDFS等各种来源的实时输入数据，进行处理后，处理结构保存在HDFS、DataBase等各种地方。


# Flink
## 简介
Flink 是一个针对流数据和批数据的分布式处理引擎，代码主要是由 Java 实现，部分代码是 Scala。它可以处理有界的批量数据集、也可以处理无界的实时数据集。对 Flink 而言，其所要处理的主要场景就是流数据，批数据只是流数据的一个极限特例而已，所以 Flink 也是一款真正的流批统一的计算引擎。
Flink 提供了 State、Checkpoint、Time、Window 等，它们为 Flink 提供了基石，本篇文章下面会稍作讲解，具体深度分析后面会有专门的文章来讲解。

## Flink 整体结构 
![avatar](http://5b0988e595225.cdn.sohucs.com/images/20190909/9468b78a1f94401e85fd624e80580ecc.jpeg)
从下至上：
- 1、部署：Flink 支持本地运行（IDE 中直接运行程序）、能在独立集群（Standalone 模式）或者在被 YARN、Mesos、K8s 管理的集群上运行，也能部署在云上。
- 2、运行：Flink 的核心是分布式流式数据引擎，意味着数据以一次一个事件的形式被处理。
- 3、API：DataStream、DataSet、Table、SQL API。
- 4、扩展库：Flink 还包括用于 CEP（复杂事件处理）、机器学习、图形处理等场景。

# Kafka
## 原理
Kafka中都有哪里会有选举过程，使用什么工具支持选举(ZooKeeper) 
## 数据丢失、重复
Kafka中如何保证数据一致性(从生产者、broker、消费者组三个部分都介绍下) 
## 数据积压
。。。

# 数仓
## 层级架构
**ODS层**
原始数据层，存放原始数据，直接加载原始日志、数据，数据保持原貌不做处理。
- （1）保持数据原貌不做任何修改，起到备份数据的作用。
- （2）数据采用压缩，减少磁盘存储空间（例如：原始数据100G，可以压缩到10G左右）
- （3）创建分区表，防止后续的全表扫描
**DWD层**
明细数据层，结构和粒度与ods层保持一致，对ods层数据进行清洗(去除空值，脏数据，超过极限范围的数据)。DWD层需构建维度模型，一般采用星型模型，呈现的状态一般为星座模型。
**DWS层**
服务数据层，以dwd为基础，进行轻度汇总。一般聚集到以用户当日，设备当日，商家当日，商品当日等等的粒度。
在这层通常会有以某一个维度为线索，组成跨主题的宽表，比如 一个用户的当日的签到数、收藏数、评论数、抽奖数、订阅数、点赞数、浏览商品数、添加购物车数、下单数、支付数、退款数、点击广告数组成的多列表。
**ADS层**
数据应用层， 也有公司或书把这层成为app层、dal层、dm层，叫法繁多。面向实际的数据需求，以DWD或者DWS层的数据为基础，组成的各种统计报表。统计结果最终同步到RDS以供BI或应用系统查询使用。

## 数仓建模理论
### 维度建模
以事实表为核心，多个维度表作为手臂形成的星型模型，是维度建模的典型实现方式。
优点：技术要求不高，快速上手，敏捷迭代，快速交付；更快速完成分析需求，较好的大规模复杂查询的响应性能
缺点：维度表的冗余会较多，视野狭窄

### 关系建模
是数据仓库之父Inmon推崇的、从全企业的高度设计一个3NF模型的方法，用实体加关系描述的数据模型描述企业业务架构，在范式理论上符合3NF，站在企业角度面向主题的抽象，而不是针对某个具体业务流程的实体对象关系抽象。它更多是面向数据的整合和一致性治理。
当有一个或多个维表没有直接连接到事实表上，而是通过其他维表连接到事实表上时，其图解就像多个雪花连接在一起，故称雪花模型。雪花模型是对星型模型的扩展。它对星型模型的维表进一步层次化，原有的各维表可能被扩展为小的事实表，形成一些局部的 "层次 " 区域，这些被分解的表都连接到主维度表而不是事实表。雪花模型更加符合数据库范式，减少数据冗余，但是在分析数据的时候，操作比较复杂，需要join的表比较多所以其性能并不一定比星型模型高。
优点：规范性较好，冗余小，数据集成和数据一致性方面得到重视，比如运营商可以参考国际电信运营业务流程规范（ETOM），有所谓的最佳实践。
缺点：需要全面了解企业业务、数据和关系；实施周期非常长，成本昂贵；对建模人员的能力要求也非常高，容易烂尾。

## 元数据


## 数据治理

