# TiDBTree
TiDB技术

<pre>
TiDB
      TiDB是一个分布式NewSQL数据库。它支持水平弹性扩容，ACID事务，标准SQL，MYSQL语法和
   MYSQL协议，具有数据强一致的高可用性，是一个不仅适合OLTP场景还适合OLAP的混合数据库。
</pre>

<pre>
      TIDB是PingCAP公司受Google Spanner/F1论文启发而设计的开源分布式HTAP数据库，结合了
   传统的RDBMS和NOSQL的最佳特性。TIDB兼容MYSQL，支持无限的水平扩容，具备强一致性，高可用性
   。TIDB的目标是为OLTP和OLAP场景提供一站式的解决方案，TIDB具备以下核心特点：
      1：高度兼容MYSQL
         大多数情况下，无需修改代码即可从MYSQL轻松迁移到TIDB，分库分表后的MYSQL集群亦可通
         过TIDB工具进行实施迁移。
      2：水平弹性扩容
         通过简单的增加新节点即可实现TIDB的水平扩容，按需扩展吞吐和存储，轻松应对高并发，
         海量数据场景。
      3：分布式事务
         TIDB 100%支持标准的ACID事务
      5：真正金融级高可用
         相比传统主从M-S复制方案，基于RAFT的多数派选举协议可以提供金融级的100%数据强一致性
         保证，且在不丢失大多数副本的前提下，可以实现故障的自动回复，无需人工介入。
      6：一站式HTAP解决方案
         TIDB作为典型的OLTP行存储数据库，同时兼容强大的OLAP性能，配合TiSpark，可提供一
         站式HTAP解决方案，一份存储同时处理OLTP&OLAP无需传统繁琐的ETL过程。
      7：云原生SQL数据库
         TIDB是为云而设计的数据库，通K8s深度耦合，支持公有云，私有云，混合云，使部署，配置
         ，维护变得简单。
         TIDB的设计目标是100%的OLTP场景和80%的OLAP场景，更复杂的OLAP分析可以通过TiSpark
         项目来完成，TiDB对业务没有任何侵入性，能优雅的替换传统的数据库中间件，数据库分库
         分表等Sharding方案。同时它也让开发运维人员不用关注数据库扩容的细节问题，专注于业务
         开发，极大的提升研发的生产力。
</pre>

![](https://i.imgur.com/HuroPK3.png)

<pre>
TiDB的整体架构

    TiDB集群主要分为三个组件：
        1）TiDB Server
           TiDB Server负责接收SQL请求，处理SQL相关的逻辑，并通过PD找到存储计算所需数据的
         TiKV地址，与TiKV交互获取数据，最终返回结果。TiDB Server是无状态的，其本身并不存
         储数据，值负责计算，可以无限水平扩展，可以通过负载均衡组件（LVS, HAProxy, F5）
         对外提供统一的接入地址。

    PD Server
         Placement Driver是整个集群的管理模块，其主要工作有三个
             1：存储集群的元信息（某个Key存储在哪个TiKV节点）
             2：对TiKV集群进行调度和负载均衡（如数据的迁移， Raft group leader的迁移）
             3：分配全局唯一且递增的事务ID
         PD是一个集群，需要部署奇数个节点，一半线上推荐至少3个节点。

    TiKV Server
         TiKV Server负责存储数据，从外部看TiKV是一个分布式的提供事务的Key-Value存储引擎
      。存储数据的基本单位是Region，每个Region负责存储一个Key Range（从StartKey
      到EndKey的左闭右开区间）的数据，每个TiKV节点负责多个Region。TiKV使用Raft协议做复制
      ，保持数据的一致性和容灾。副本以Region为单位进行管理，不同节点上的多个Region构成一个
      RaftGroup，互为副本，数据在多个TiKV之间的负载均衡由PD调度，这里也是以Region为单位
      进行调度。
</pre>

<pre>
TiDB核心特性
   1：水平扩展
       无限水平扩展是TiDB的一大特点，这里说的水平扩展包括两个方面：计算能力和存储能力
       TiDB Server负责处理SQL请求，随着业务的增长，可以简单的添加TiDB Server节点，
       提高整体的处理能力，提供更高的吞吐。TiKV负责存储数据，随着数据量的增长，可以部署
       更多的TiKV Server节点解决数据Scale的问题。PD会在TiKV节点之间以Region为单位做
       调度，将部分数据转移到新的节点上，所以在业务的早期，可以只部署少量的服务实例（
       推荐至少部署3个TiKV, 3个PD, 2个TiDB）,随着业务量的增长，按照需求添加TiKV
       或者TiDB实例。

   2：高可用
       高可用是TiDB的另一大特点，TiDB/TiKV/PD这三个组件都能容忍部分实例失效。不影响整个
       集群的可用性。下面分别说明这三个组件的可用性，单个实例失效后的后果以及如何恢复。

       TiDB
           TiDB是无状态的，推荐至少部署两个实例，前端通过负载均衡组件对外提供服务。当单个
       实例失效时，会影响正在这个实例上进行的Session,从应用的角度看，会出现单次请求失败
       的情况，重新连接后即可继续获得服务。单个实例失效后，可以重启这个实例或者部署一个新
       的实例。

       PD
           PD是一个集群，通过Raft协议保持数据的一致性，单个实例失效时，如果这个实例不是
       Raft的Leader，那么服务完全不受影响；如果这个实例是Raft的Leader，会重新选出新的
       Raft的Leader，自动回复服务。PD在选举的过程中无法对外提供服务，这个时间大约是3s.
       推荐至少部署3个PD实例，单个实例失效后，重启这个实例或者添加新的实例。

       TiKV
           TiKV是一个集群，通过Raft协议保持数据的一致性，并通过PD做负载均衡调度，单个节点
       失效时，会影响这个节点上存储的所有Region.对于Region中的Leader节点，会中断服务，等
       待重新选举；对于Region中的Follower节点，不会影响服务。当某个TiKV节点失效，并且在
       一段时间内（默认30分钟）无法恢复，PD会将其上的数据迁移到其他的TiKV节点上。
</pre>