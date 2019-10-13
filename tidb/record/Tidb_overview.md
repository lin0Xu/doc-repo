## Tidb

###### docker-compose使用

```
##启动
docker-compose up -d
##连接
mysql -h 127.0.0.1 -P 4000 -u tidb -ptidb

## grafana监控：
http://localhost:3000



```



##### TiDB与Mysql区别

```
## https://github.com/pingcap/docs-cn/blob/master/v1.0/sql/ddl.md
	支持 UNIQUE 索引，不支持 FULLTEXT 和 SPATIAL 索引。
	index_col_name 支持长度选项，最大长度限制为3072字节，该长度限制不根据建表时使用的存储引擎、字符集而变。这是因为 TiDB 并非使用 Innodb 、 MyISAM 等存储引擎，因此，仅对建表时的存储引擎选项进行了 MySQL 语法上的兼容。对于字符集，TiDB 使用的是 utf8mb4 字符集，对于建表时的字符集选项同样仅有 MySQL 语法上的兼容。详见与 MySQL 兼容性对比章节。
	index_col_name 支持索引排序选项 ASC 和 DESC。 排序选项行为与 MySQL 一致，仅支持语法解析，内部所有索引都是以正序排列。详见 MySQL 的 CREATE INDEX Syntax 章节。
	index_option 支持 KEY_BLOCK_SIZE 、index_type 和 COMMENT 。 COMMENT 允许最大1024个字符。不支持 WITH PARSER 选项。
	index_type 支持 BTREE 和 HASH ，但仅有 MySQL 语法上的支持，即索引类型与建表语句中的存储引擎选项无关。举例：在 MySQL 中，使用 Innodb 的表，在 CREATE INDEX 时只能使用 BTREE 索引，而在 TiDB 中既可以使用 BTREE 也可以使用 HASH 。
	不支持 MySQL 的 algorithm_option 和 lock_option 选项。
	TiDB 单表最多支持 512 个列。InnoDB 的限制是 1017。MySQL 的硬限制是 4096。
	DROP INDEX 用于删除表上的一个索引，目前暂不支持删除主键索引。
```

##### Tidb FAQ

```
https://github.com/pingcap/docs-cn/blob/master/v1.0/FAQ.md
```





##### tidb官方文档

```
https://github.com/pingcap/docs-cn
```

##### FAQ

```
https://pingcap.com/docs-cn/v3.0/faq/tidb/
```

##### 故障诊断

```
https://pingcap.com/docs-cn/dev/how-to/troubleshoot/cluster-setup/
```

##### 官方博客

```
https://pingcap.github.io/blog/
```

##### TiDB 的最佳适用场景

```
1. 数据量大，单机保存不下
2. 不希望做 Sharding 或者懒得做 Sharding
3. 访问模式上没有明显的热点
4. 需要事务、需要强一致、需要灾备
```







##### Tidb特点：

```
1. 按照字节序的顺序扫描的效率是比较高的；
2. 连续的行大概率会存储在同一台机器的邻近位置，每次批量的读取和写入的效率会高； 
3. 索引是有序的（主键也是一种索引），一行的每一列的索引都会占用一个 KV Pair，比如，某个表除了主键有 3 个索引，那么在这个表中插入一行，对应在底层存储就是 4 个 KV Pairs 的写入：数据行以及 3 个索引行。
4. 一行的数据都是存在一个 KV Pair 中，不会被切分，这点和类 BigTable 的列式存储很不一样。
```

```
 Q:字节序排序是什么？如何排序？
 
```

```
 Q:数据行、索引都以 K->V pairs存储在TiKV中，如果行数据太大， K->V 中V怎么存？会不会有存不下的情况，存不下怎么办？
 	建议：
	尽可能批量写入，但是一次写入总大小不要超过 Region 的分裂阈值（64M），另外 TiDB 也对单个事务有大小的限制。
	存储超宽表是比较不合适的，特别是一行的列非常多，同时不是太稀疏，一个经验是最好单行的总数据大小不要超过 64K，越小越好。大的数据最好拆到多张表中。
	对于高并发且访问频繁的数据，尽可能一次访问只命中一个 Region，这个也很好理解，比如一个模糊查询或者一个没有索引的表扫描操作，可能会发生在多个物理节点上，一来会有更大的网络开销，二来访问的 Region 越多，遇到 stale region 然后重试的概率也越大（可以理解为 TiDB 会经常做 Region 的移动，客户端的路由信息可能更新不那么及时），这些可能会影响 .99 延迟；另一方面，小事务（在一个 Region 的范围内）的写入的延迟会更低，TiDB 针对同一个 Region 内的跨行事务是有优化的。另外 TiDB 对通过主键精准的点查询（结果集只有一条）效率更高。
```

```
 Q:Region 分裂、合并、迁移；

```

```
Q:某一行数据的索引kv和数据不在同一个region怎么办？这两个Region的主在不同的TiKV节点上;
 	建议： 对于大海捞针式的查询来说 (海量数据中精准定位某条或者某几条)，务必通过索引。 当然也不要盲目的创建索引，创建太多索引会影响写入的性能。
 	TiDB 实现了全局索引，所以索引和 Table 中的数据并不一定在一个数据分片上，通过索引查询的时候，需要先扫描索引，得到对应的行 ID，然后通过行 ID 去取数据，所以可能会涉及到两次网络请求，会有一定的性能开销。
 	如果查询涉及到大量的行，那么扫描索引是并发进行，只要第一批结果已经返回，就可以开始去取 Table 的数据，所以这里是一个并行 + Pipeline 的模式，虽然有两次访问的开销，但是延迟并不会很大。
```

```
Q:对于分布式数据库来说是不推荐的，随着插入的压力增大，会在这张表的尾部 Region形成热点，而且这个热点并没有办法分散到多台机器。TiDB 在 GA 的版本中会对非自增 ID 主键进行优化，让 insert workload 尽可能分散。
```

```
Q:TiDB 的典型的应用场景是：
 	大数据量下，MySQL 复杂查询很慢；
	大数据量下，数据增长很快，接近单机处理的极限，不想分库分表或者使用数据库中间件等对业务侵入性较大，架构反过来约束业务的 Sharding 方案；
	大数据量下，有高并发实时写入、实时查询、实时统计分析的需求；
	有分布式事务、多数据中心的数据 100% 强一致性、auto-failover 的高可用的需求。
	如果整篇文章你只想记住一句话，那就是数据条数少于 5000w 的场景下通常用不到 TiDB，TiDB 是为大规模的数据场景设计的。如果还想记住一句话，那就是单机 MySQL 能满足的场景也用不到 TiDB。
```

```
Q:如果在一台机器上启动多个 TiKV 实例，会有什么问题？
	通过label标识，限制同一个Region的不同replica不能在同一个label下；
```

```
Q:SQL on KV
	TiDB 自动将 SQL 结构映射为 KV 结构。具体的可以参考《三篇文章了解 TiDB 技术内幕 - 说计算》这篇文档。简单来说，TiDB 做了两件事：
	一行数据映射为一个 KV，Key 以 TableID 构造前缀，以行 ID 为后缀
	一条索引映射为一个 KV，Key 以 TableID+IndexID 构造前缀，以索引值构造后缀
	可以看到，对于一个表中的数据或者索引，会具有相同的前缀，这样在 TiKV 的 Key 空间内，这些 Key-Value 会在相邻的位置。那么当写入量很大，并且集中在一个表上面时，就会造成写入的热点，特别是连续写入的数据中某些索引值也是连续的(比如 update time 这种按时间递增的字段)，会在很少的几个 Region 上形成写入热点，成为整个系统的瓶颈。同样，如果所有的数据读取操作也都集中在很小的一个范围内 (比如在连续的几万或者十几万行数据上)，那么可能造成数据的访问热点。
```

```
Q:需要特别注意的参数：
[raftstore]
# 默认为 true，表示强制将数据刷到磁盘上。如果是非金融安全级别的业务场景，建议设置成 false，
# 以便获得更高的性能。
sync-log = true
```

```
Q:分布式事务


```







##### TiDB 核心组件

###### TiDB Server

```
	TiDB Server 负责接收 SQL 请求，处理 SQL 相关的逻辑，并通过 PD 找到存储计算所需数据的 TiKV 地址，与 TiKV 交互获取数据，最终返回结果。 TiDB Server 是无状态的，其本身并不存储数据，只负责计算，可以无限水平扩展，可以通过负载均衡组件（如LVS、HAProxy 或 F5）对外提供统一的接入地址。
```

###### PD Server

```
	Placement Driver (简称 PD) 是整个集群的管理模块，其主要工作有三个： 一是存储集群的元信息（某个 Key 存储在哪个 TiKV 节点）；二是对 TiKV 集群进行调度和负载均衡（如数据的迁移、Raft group leader 的迁移等）；三是分配全局唯一且递增的事务 ID。
	PD 是一个集群，需要部署奇数个节点，一般线上推荐至少部署 3 个节点。
```

###### TiKV Server

```
	TiKV Server 负责存储数据，从外部看 TiKV 是一个分布式的提供事务的 Key-Value 存储引擎。存储数据的基本单位是 Region，每个 Region 负责存储一个 Key Range （从 StartKey 到 EndKey 的左闭右开区间）的数据，每个 TiKV 节点会负责多个 Region 。TiKV 使用 Raft 协议做复制，保持数据的一致性和容灾。副本以 Region 为单位进行管理，不同节点上的多个 Region 构成一个 Raft Group，互为副本。数据在多个 TiKV 之间的负载均衡由 PD 调度，这里也是以 Region 为单位进行调度。
```

##### TiDB高可用

```
	高可用是 TiDB 的另一大特点，TiDB/TiKV/PD 这三个组件都能容忍部分实例失效，不影响整个集群的可用性。下面分别说明这三个组件的可用性、单个实例失效后的后果以及如何恢复。
1. TiDB
	TiDB 是无状态的，推荐至少部署两个实例，前端通过负载均衡组件对外提供服务。当单个实例失效时，会影响正在这个实例上进行的 Session，从应用的角度看，会出现单次请求失败的情况，重新连接后即可继续获得服务。单个实例失效后，可以重启这个实例或者部署一个新的实例。
2.PD
	PD 是一个集群，通过 Raft 协议保持数据的一致性，单个实例失效时，如果这个实例不是 Raft 的 leader，那么服务完全不受影响；如果这个实例是 Raft 的 leader，会重新选出新的 Raft leader，自动恢复服务。PD 在选举的过程中无法对外提供服务，这个时间大约是3秒钟。推荐至少部署三个 PD 实例，单个实例失效后，重启这个实例或者添加新的实例。
3.TiKV
	TiKV 是一个集群，通过 Raft 协议保持数据的一致性（副本数量可配置，默认保存三副本），并通过 PD 做负载均衡调度。单个节点失效时，会影响这个节点上存储的所有 Region。对于 Region 中的 Leader 结点，会中断服务，等待重新选举；对于 Region 中的 Follower 节点，不会影响服务。当某个 TiKV 节点失效，并且在一段时间内（默认 10 分钟）无法恢复，PD 会将其上的数据迁移到其他的 TiKV 节点上。
```





 #####  Jepsen和分布式一致性验证

```
https://aphyr.com/tags/jepsen
https://pingcap.com/blog-cn/tidb-jepsen/
```

##### 两阶段提交事务

```
https://pingcap.com/blog-cn/percolator-and-txn/
```

##### Mysql协议文档

```
https://dev.mysql.com/doc/internals/en/client-server-protocol.html
```



