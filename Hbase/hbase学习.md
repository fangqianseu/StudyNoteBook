# 总览

+ hbase是一个key-value型的，可以存储结构化和半结构化的无类型数据的数据库

+ hbase中的所有数据都是以 byte 的形式存储，没有数据类型的说法，可以理解为都是string类型

+ hbase可以看做是 `[table, rowKey, family, quailier, timestamp]  -> value` 5维坐标确定的一个 k-v集合

  因此，在我的理解中，family的存在体现了hbase结构化的一面，需要在表内保持一致，而family内部则相当自由，可以根据需要存储任意结构的数据，是非结构化的一面。

+ hbase会自动对插入的数据进行排序，排序规则为：

  + 按照`rowKey family qualifier` 的顺序， 依据 ASCLL码升序
  + 按照 `timestamp ` 进行降序

+ hbase的ACID

  + 操作保证低级原子性。比如Put()操作只会整体成功，或者整体失败回到开始前状态，永远不会不分行写入而另一部分没有
  + 行间操作不是原子性的，不保证所有的操作全部成功或者全部失败。单行操作是原子性的
  + 对于给定行的多个写操作，分割为每个写操作为整体彼此独立
  + 给定行的Get操作，返回系统当时保存的完整行
  + 全表扫描不是对某个时间点的快照扫描，会返回扫描开始后的更新数据。但是对于单行，则是更新后的完整数据保证

+ hbase的存储是面向列族的，一个列族中的所有数据存储在一个底层文件中，一条记录分散在多个底层的列族文件中。这就导致对列族进行检索的效率最高，不同于msql这种面向行的存储。

# Hbase数据模型

+ hbase的数据模型

  + 表(table)：hbase用表来组织数据，概念近似于mysql中的表
  + 行(row)：在table中，数据按照row存储，由行键(rowKey)唯一标识
  + 列族(family)：row中的数据按照family分组存储。family也影响到hbase的数据物理存放，因此需要预先定义且不轻易修改。一个table中的每个row都拥有相同的family
  + 单元(cell)：列族中用于保存数据的基础单元，包含以下3组概念
    + 列限定符(quailier)：在列族中数据通过列限定符来定位，不必事前定义，不用与其他row保持一致
    + 值(value)：存储与一个quailier对应的值，真正保存数据的地方
    + 时间戳(timestamp)：在cell中，每个quailier存储的cell都会有一个timestamp，用作版本管理

+ hbase的逻辑数据模型

  ![image-20201015112954084](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20201015112954.png)

  hbase中，数据是按照 `row -> family -> qualifier -> timestamp` 的层级进行存储的，数据检索时也按照相同的层级

  row中的 `rowKey`唯一索引，查询时只能按照rowKey进行检索。

  在一行row之中，可以存在多个family，同一个表中的每个row的family必须相同。

  family之中，存在 `<qualifier,value>`的键值对，相同family之内的键值对结构不必相同，可以根据喜好自由设定。

  对于每个键值对，都会存在一个timestamp时间戳属性，用于版本控制。同一个qualifier，可以存在不同timestamp的多个value值

# hbase存储结构

## 逻辑存储结构

![image-20201016105145455](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20201016105145.png)

上图为Hbase的逻辑存储结构，

+  Region Server 就是一个机器节点(服务器)，可能存储了多个table；

+ 每个table在行的方向上分割为多个 Region；每个表最开始只有一个Region，随着数据不断插入表，region不断增大，当增大到一个阀值的时候，Region就会等分会两个新的Region。table中的行不断增多，就会有越来越多的Region

+ Region 包含着多个列簇 (Colume Family， CF)：CF中，只包含一张表的一个列族中的数据。一个列族的数据可能在多个CF中存储

## 物理存储结构

![img](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20201016103134.png)

上图为网上看到的一张很不错的物理存储结构图，其中有些对象可以与逻辑存储结构对应：

Region Server 对应 HRegion Server，为物理的节点级别；Region 对应 HRegion，为表级别；Column Family 对应 HStore，为列族级别

- HRegion Server 对应机器节点，包含多个 HRegion，负责响应用户的 IO 请求，向HDFS文件系统中读写数据。一个HRegionServer会有多个HRegion和一个HLog。

- 每个Region Server维护一个Hlog，记录数据的所有变更，类似mysql中的binlog，用来做灾难恢复。一旦数据修改，就可以从log中进行恢复。

  每个Region Server维护一个Hlog,而不是每个Region一个。这样不同region(来自不同table)的日志会混在一起，目的是减少磁盘寻址次数，因此可以提高写性能。但也有副作用：如果一台region server下线，为了恢复其上的region，需要将region server上的log进行拆分，然后分发到其它region server上进行恢复。

  Region server会将数据保存到内存，直到达到阈值再将其刷写到磁盘，这样可避免io，提高性能。但会在宕机时带来数据丢失问题。Hlog较好的解决这个问题：每次操作都会先写入日志，只有日志写入成功后才会告知客户端写入成功，之后服务器才按照需要批量处理内存中的数据。

  如果服务器崩溃，Region Server会回访Hlog，通过数据回写，来恢复服务器的内存数据。

- HRegion 包含多个 HStore，是Hbase中分布式存储和负载均衡的最小单元，可以分布在不同的HRegion server上。

- HStore 是 HBase 的核心存储单元，每个store保存一个columns family，由一个memStore和0至多个StoreFile组成。

- MemStore 是一块内存，默认大小是 128M，如果超过了这个大小，那么就会进行刷盘，把内存里的数据刷进到 StoreFile 中。

- StoreFile以HFile格式保存在HDFS上，每当一个memstore容量达到阈值，就会flush到hdfs上，产生一个Hfile


# hbase的架构

hbase是一个分布式的数据库，可以布置为一个集群，具备横向扩展能力，实现海量的数据检索

<img src="https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20201018101233.png" alt="image-20201018101233474" style="zoom:50%;" />

+ Zookeeper
  +  保证任何时候只有一个master节点
  + 存贮所有Region的寻址入口。
  + 实时监控Region Server的状态，将Region server的上线和下线信息实时通知给Master
  + 存储Hbase的schema：如有哪些table，每个table有哪些column family

+ Master
  + 为Region server分配region
  + 负责region server的负载均衡
  + 发现失效的region server并重新分配其上的region
  + 处理schema更新请求

+ Region Server
  + 维护Master分配的region，处理对IO请求
  + 负责切分超过设定限定的region，分割为多个小部分

　　可以看到，client访问Hbase上数据时，master仅仅维护者table和region的元数据信息，主要的访问工作是由各个Region Server负责；可以通过对Region Server的横向扩展，满足不断上升的数据量需求

## 读过程

 + Client 请求读取数据时，先转发到 ZK 集群，在 ZK 集群中寻找到相对应的 Region Server，再找到对应的 Region
 + 先是查 MemStore1（内存缓存），如果在 MemStore 中获取到数据，那么就会直接返回，否则就是再由 Region 找到对应的 Store File，从而查到具体的数据。
 + HMaster 和 HRegion Server 可以是同一个节点上，可以有多个 HMaster 存在，但是只有一个 HMaster 在活跃。
 + 在 Client 端会进行 rowkey-> HRegion 映射关系的缓存，降低下次寻址的压力。

## 写过程

+  Client进行发起数据的插入请求，如果 Client本身存储了关于 Rowkey和 Region 的映射关系的话，那么就会先查找到具体的对应关系，如果没有的话，就会在ZK中进行查找到对应 Region server，然后再转发到具体的 Region 上。
+ 所有的数据在写入的时候先是记录在 Hlog 中，只有写入成功后才会告知客户端写入成功。之后先写进 Memstore 中，然后再刷到 Hfile中。
+ 同时，Regain server会检查 MemStore 是否满了：如果满了，那么就会进行刷盘，输出到一个 新Hfile 中。

# 常用java API

![image-20201017200245383](https://raw.githubusercontent.com/fangqianseu/imgStore/master/tx/20201017200245.png)

## Admin

+ 用于对hbase进行管理操作，如增删改查
+ 常用的是添加表、设定最高版本数、删除表等
+ 在创建表时，需要同时指定列族属性：应为列族属性是确定的，后续很难更改

## Get

+ Get用于知道 rowKey时，获取对应的数据
+ 由于不同的family存在不同的Hfile文件，一次Get会读取多个文件，带来多次磁盘IO；可以通过`get.addFamily()`指定列族，减少io
+ Get的条件限制作用于服务端，可以减少网络传输

## Put

+ 用于增加数据
+ 可以增加行，也能增加列族
+ 支持批量添加操作，减少与服务端的交互

## Scan

+ 用于对表格数据的检索（不确定行键的情况下）
+ 支持分页查询(Pagefilter)
+ 可以对 rowKey、family、quilifier、value 四种粒度的检索

## Delete

+ 用于删除数据
+ 同put、get，可以批量添加数据

# 常用优化点

+ rowKey作为Hbase唯一的检索方式，需要精心的设计，提高检索性能。

  但是有些时候，对于性能的追求可能带来稳定性的隐患：比如，如果使用时间序列作为rowKey，这样每次新增数据都会在文件的尾部，而且检索效率也会很高，但是在进行大量写入时，会全部集中在一个region中，造成热点问题

+ Mysql这类的数据库中，范式会显得特别重要。但是在HBase中，有时需要打破范式，比如数据冗余。

+ 在进行scan操作时，需要合理考虑scan一次读取的行数：过小交互的次数会变多，过多则传输的数据量过多，带来延时较高

+  批量get请求，，可以减少RPC的次数，显著提高吞吐量。但是批量get请求要么成功返回所有请求数据，要么抛出异常。压力变大时，可能带来负优化

+ 获取数据时，指定列族名。从上面可以看出，一行数据的不同列族是存在不同的文件中的。不指定列族进行检索的话，不同列族的数据需要独立进行检索，性能必然会比指定列族的查询差的多。

  此外，指定请求的列的话，不需要将整个列族的所有列的数据返回，减少了网路IO

+ 读缓存优化。BlockCache是读缓存，读多写少业务可以将BlockCache占比调大。但是，盲目调大读缓存会带来gc的压力，需要根据实际情况进行设置

除了上述的常用的优化点之外，hbase的可配置项都在`org.apache.hadoop.hbase.HConstants`这个类中进行配置，可以根据实际情况进行调节。