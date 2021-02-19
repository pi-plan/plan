# TM 角色的单点问题
TM 作为 A2PC 中的重要角色，如果存在点点问题，就会造成整体不可用，即便是 [XA 协议](https://zh.wikipedia.org/wiki/X/Open_XA)中，TM 角色面临一样的问题。

## 关于单点问题
单点问题是指系统中单个节点故障，就会造成整体故障的问题。单点问题的解决方案并不是 A2PC 规范中的一部分，只是在这里给出一个解决方法来解决单点问题。

这里把出现单点故障的原因分为两类：
> 1. 性能达到单点上限。
> 2. 因为其他不可控因素造成单点不可用。

## 性能单点上限
无论单台机器的性能多么强悍，单台机器总会有性能上限。对事务的处理分发到不同的机器上即可分摊压力，从而解决单点性能问题。但是如何做拆分？又如何保障 TM 数据的安全呢？

### 如何拆分？
A2PC 中 RM 负责维护的数据有两块。
1. 分支事务数据，包括分支事务数据节点地址，数据唯一标识，全局锁状态等。
2. 全局事务的 XID，状态维护。

如果单独以全局事务 XID 来拆分，每个分支事务会涉及那些数据是无法预测的，有可能会出现两个分片同时在处理同一个分支事务关联的数据。而且从 A2PC 的涉及来看，最重要的是分支事务和全局锁的数据需要保持唯一，所以这里拆分只能以分支事务关联的数据唯一标识来拆分。比如：数据节点 A，有一张 `product` 数据表，其中 `id=2` 是这条数据的唯一标识。那么 RM 做拆分的唯一标识就是 `数据节点 A-数据表 product-id=2` md5 后取余数这样精细的拆分，也可以根据表来做拆分，但是原则上是：一个分支事务中的数据有且只有一个分片处理。这样就能保证和这条数据相关的所有分支事务都在同一个分片中。分支事务存储的表结构为：
```sql
CREATE TABLE `lock_table_x` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `lock_key` varchar(255) NOT NULL,
  `xid` bigint(20) DEFAULT NULL,
  `node` varchar(255) DEFAULT NULL,
  `table` varchar(128) NOT NULL,
  `context` varchar(2000) DEFAULT '',
  `client_id` varchar(64) DEFAULT '',
  `create_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  `update_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (`id`),
  UNIQUE KEY `un_lock` (`node`,`table`,`lock_key`),
  KEY `in_xid` (`xid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
根据分支事务拆分之后，又有一个问题，就是一个全局事务中，可以有多个分支事务，而这些分支事务又分散在不同的分片上。在全局事务提交或者回滚的时候，修改分支事务的数据，那么这又是一个分布式事务。

这里不能在继续使用 A2PC 来解决分布式事务问题，否这样递归下去这个问题就无解了。所以 TM 需要采用其他的分布式事务处理方案来解决这个问题。TM 把分支事务和全局事务数据与状态分开维护，每个分支事务仅仅维护自身数据和关联的 XID。全局事务的状态有全局事务维护，每次读取状态的时候分别获取分支事务和全局事务两份数据，整合在一起。
```sql
CREATE TABLE `transaction_info_x` (
  `xid` bigint(20) NOT NULL AUTO_INCREMENT,
  `status` tinyint(4) DEFAULT NULL,
  `client_id` varchar(64) DEFAULT '',
  `create_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  `update_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (`xid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
分支事务和全局事务之间 通过 XID 字段进行关联。

### 安全保障
通过上面的拆分方式，来逐个说明如何保证安全的。

##### BEGIN
开启一个分布式事务，因为这里只涉及全局事务的数据，和分支事务暂无关联，是一个独立的本地事务。

##### ACQUIRE_LOCK
获取分支事务里数据的全局锁，虽然是以分支事务做拆分，但是拆分的逻辑在申请全局锁的时候依然是可以正确的获取到所在的分片，如果没有查到有分支事务，可以直接分配全局锁，如果有分支事务，可以通过关联的 XID 查询到全局事务的状态，如果是已经提交或者已经回滚，就分配全局锁。这里仅仅会在有锁竞争的时候会多一次 RPC 查询。而且分支事务的创建不会管理到全局事务的同步调整，XID 是通过 APP 传递过来的，这里也是一个独立的本地事务。

##### COMMIT
提交一个全局事务。因为分支事务是不会保存全局事务状态的，所以在全局事务提交的时候，直接修改全局事务的状态即可。等待日志压缩行为清理掉已经提交的事务相关的数据。

##### ROLLBACK
回滚一个全局事务。和 COMMIT 行为一样，也是全局事务的状态，在回滚的时候虽然回去回滚每个分支事务，但每个分支事务的回滚是可以独立进行的，也是可以补偿、幂等的（[回滚行为的安全](/a2pc/introduction?id=事务回滚)），分支事务的提交成功后 TM 只需要修改全局事务一个状态即可，也是一个独立的本地事务。

##### COMPENSATION
事务补偿行为，是 TM 自己发起的主要逻辑是触发补偿，和 ROLLBACK 行为一样。

##### TRANSACTION_TIMEOUT
全局事务执行时间超时，也是和 ROLLBACK 行为一样。

其他的 BEGIN_LOCAL，ACQUIRE_LOCAL_LOCK，DML，WRITE_REUNDO_LOG，COMMIT_LOCAL，ROLLBACK_LOCAL，ROLLBACK_BRANCH 这些行为都不涉及到 RM 。
至此，根据分支事务对 RM 进行分片之后，可以解决单机性能瓶颈问题，而且只会在出现锁竞争的时候才会增加性能成本，成本也仅仅是一次 RPC 通信。

## 单点故障转移
无论 RM 是否进行了分片，其中的任何一个实例都有可能遇到故障，造成不可用，在遇到故障的时候又该如何做故障转移避免整体不可用呢？首先，把故障的单点分为两类：
> 1. 存储节点故障，比如 RM 中用 MySQL 负责数据的持久化，存储节点故障就是在这种场景下 MySQL 出现了故障。
> 2. 计算节点故障，在持久化和计算分离的 RM 实现中，负责 RM 逻辑的实例，就是极端节点。
在 A2PC 的实现中，这两类节点也可以在一起，故障转移单点方式也是一样的逻辑。

### 计算节点单点故障
在采用计算和存储分离的 RM 实现中，计算是无态的，因此是可以同时存在多个计算节点的，当一个计算节点故障了，可以自动切换到其他计算节点。计算节点只负责业务逻辑，持久化的并不保存这在计算几点，所以计算节点的切换、上下线都是安全的。

### 存储节点单点故障
为了避免数据错乱，存储节点统一时间只能有一个节点担任主节点的角色负责写入，其他节点同步数据，随时做好在主节点故障后替换掉主节点。但是这里主从节点直接的数据需要实时保持一致。才能在切换的时候，不会出现数据缺失的情况，这个逻辑的实现，可以参考 [Raft](https://raft.github.io/) ，[Paxos](https://zh.wikipedia.org/wiki/Paxos算法) 这一系列[共识算法](https://en.wikipedia.org/wiki/Consensus_(computer_science))实现这里不在赘述其原理。

如果使用 MySQL 做存储节点，可以使用 Group Replication 直接启动此功能。如果需要自己实现存储节点的故障自动迁移，可以使用 [PRaft](/praft/introduction) Raft 算法库来快速实现功能。