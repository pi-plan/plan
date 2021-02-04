# 术语表

## Zone
Zone 是一份能完整支撑业务运行的逻辑单元，可以是一个独立的物理机房，也可以是一个逻辑机房。在 Pi 系统里是逻辑上的一个业务完整运行单元。

## Zone ID
每个 Zone 有个唯一的编号，在设计中用来标记和区分不同的 Zone。

## Zone Sharding
Zone Sharding 是指把业务逻辑分配到不同的 Zone 执行的过程。和 DB Sharding 类似，只是更在以 Zone 层面上。

## ZSID (Zone Sharding ID)
Zone Sharding ID（简写ZSID）是用来分配 Zone 连续序列序列，范围的大小不受限制可以是 2^10 或者更大的数字。比如 Zone Sharding ID 有 0 ～ 63 个，0 ～ 31 由 Zone 1 处理，32 ～ 63 由 Zone 2 进行处理。每个业务根据自己的需要比如根据用户 ID、IP、设备等信息计算出 Zone Sharding ID，然后根据规则落地到不同的 Zone 执行。

## ZSKey (Zone Sharding Key)
Zone Sharding Key（简写 ZSKey）是业务、数据库的概念，比如通过用户 ID 计算出来 Zone Sharding ID 那么用户 ID 就叫做 Zone Sharding Key。在数据层面，通过 `user_id` 字段计算出来 Zone Sharding ID 那么，这个字段就叫做 Zone Sharding Key。每个 Zone Sharding Key 得到的 ZSID 都是固定的，Zone Sharding 规则变更是值调整 ZSID 的规则，由此来实现 Zone Sharding 的改变。

## DB Sharding/数据库分片
DB Sharding/数据库分片是一种横向分区的数据库架构模式，所谓的横向分区技术就是将一个表中的数据按行拆分到多个不同的库表中，这些不同的库表被称作分区/Sharding。每个分区都拥有相同的模式和相同的列，但是数据行却完全不同。同样的，每一个分区中的数据都是唯一的，并且独立于其他的分区。

## DBSKey (DB Sharding Key)
DB Sharding Key（简写 DBSKey）是数据表的概念，比如对 user 表进行分表，共分为 16 张表，分别对应user_0 到 user_15，规则是表中 `user_id` 字段除以 `16` 取余数，得到的余数就是对应的真是表。那么 `user_id` 这个字段就是 DBSKey。

## Sharding 算法
是指对数据、Zone 进行分配的时候采用的算法，比如根据 user id 求余数、根据注册地址等，这类算法被称为 Sharding 算法。

## 逻辑表、真实表
数据库分库分表的实践中，比如 `user` 表被分为 16 表，分别对应的是 `user_0` 到 `user_15`，对于客户端来说是感知不到 `user_0` 的存在，只需要使用 `user` 表即可，PiDAL 会透明的处理掉分表的逻辑。这里的 `user` 被称为「逻辑表」，`user_0` 到 `user_15` 被称为「真实表」。

## 数据节点
在 PiDAL 的分库分表的中，真实表 `user_0` 可能落在机器 A 上，而，`user_1` 可能落在机器 B 上，关联的 `order` 表落在机器 C 上。这里的机器 A，B，C 就是指数据节点。

## 全局事务
在 PiDAL 的分布式事务，一次修改可能会涉及到多个数据节点的数据写入。这多个数据节点同在一个事务，这个事务就是全局事务，在 PiDAL 中全局事务的使用和 本地事务一样，`begin` 会开启一个事务，`commit` 会提交一个全局事务 `rollback` 会回滚一个全局事务。

## 本地事务/分支事务
本地事务是指，在一个单机的数据库（数据节点）中的一个完成事务操作。一个全局事务可能会包含多个本地事务，在讨论全局事务的语境中也被称为分支事务。

## TM
A2PC 中的事务管理器，负责管理全局事务。负责全局事务的提交和回滚，以及事务隔离的处理。

## RM
A2PC 中的资源管理器，在 PiDAL 中等同于每个数据节点。

## APP
A2PC 中的客户端，比如在 PiDAL 的场景中，PiDAL 的就是 APP。如果是以其他方式实现 A2PC ，那个向 TM 发起 begin，commit，rollback 的可以理解为 APP。