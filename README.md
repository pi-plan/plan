# Pi 计划
Pi 计划的愿景是：让开发这件事变的简单、高效，让应用程序变得可靠、稳定。她是一个持续性计划，会产出、更新维护的一系列相关组件、库、服务、文章、思想等。因为开发并不仅仅是代码的编写、执行、项目管理、进度管理、团队协作、跨部门沟通等等。
- [Github](https://github.com/pi-plan/plan)

## 第一阶段
Pi计划第一阶段计划实现：  

> 0. A2PC 一个安全高效的分布式事务协议。
> 1. PiDAL 一个支持分布式事务的数据库中间。并实现 A2PC 协议。
> 2. PiLCS 一个基于 Raft 算法的高可用的高性能的逻辑时间服务。
> 3. PRaft 一个异步、高性能的 Raft 算法库。
> 4. PiDTS 一个支持全量同步、实时同步、数据同步幂、多主模式下数据防止回环的数据同步服务。
> 5. PiMMS 一个高可用的元数据管理中心，负责同步和管理各种元数据和协调各个组件之间的规则。
> 6. 以上各个组件可以独立在各自的场景中使用，结合在一起可以实现业务抽象的[异地多活架构](/multi-site-high-availability/introduction)。


### PiDAL
PiDAL(Pi Data Access Layer) 是一个纯异步、高性能、兼用 MySQL 通讯协议的数据库中间件。不仅提供了常规的分库、分表、读写分离等功能，还基于 [A2PC 协议](/a2pc/introduction) 实现了分布式事务处理能力，无入侵、对客户端完全透明，无需额外兼容即可保证分库分表后数据的正确性。

另外，根据对分布式数据库中常见的问题，PiDAL 提供了 [双 Sharding 表](/pidal/introduction?id=双-sharding-表) 解决方案，在开发过程中减少因为分库对开发过程的影响。在 PiDTS 组件的协同下，不仅支持数据库在线扩容也支持数据库 Sharding 规则的重新调整。  

PiDAL 不仅支持以 DB Proxy 模式部署，在容器化实施的比较完善的场景下，PiDAL 也能以 Sidecar 的模式提供支持，可以降低不必要的性能消耗。DB Proxy 和 Sidecar 模式之间的区别和优缺点对比详情可以 [点击这里查看](/pidal/introduction?id=driver、sidecar、dbproxy)。
- [了解更多](/pidal/introduction)
- [Github](https://github.com/pi-plan/pidal)

### A2PC
**A2PC** （Asynchronous Two-Phase Commit）是一个安全、高性能、开源的分布式事务解决方案。原理上是基于 2PC 算法与 MVCC 结合的演变。相比较 [XA 协议](https://zh.wikipedia.org/wiki/X/Open_XA)（MySQL 等数据库原生支持的 2PC 分布式事务规范）中锁资源周期长，单点，多次网络请求等问题，A2PC 的异步提交和回滚能大大减少锁定周期、网络请求次数减少 30%，在 PiDAL 中优化网络请求，网络请求等待周期减少 50%。

PiDAL 实现了 A2PC ，但是为了让更多不能使用 PiDAL 的开发者企业是低成本的使用到 A2PC 方案，这里把 A2PC 的关键操作、步骤、RPC 协议、redo log、undo log、TM 等各种信息角色做出规范，便于在各种语言、数据库、资源、通讯方式的场景下复用。
- [原理介绍](/pidal/introduction)
- [协议规范](https://github.com/pi-plan/pidal)
- [A2PC 实现列表](https://github.com/pi-plan/pidal)


### PiDTS
PiDTS(Pi Data transmission server) 数据传输服务。支持不同数据库之间的数据传输服务。例如 MySQL -> PostgreSQL、MySQL -> Hive 等。支持全量同步、时时同步、数据同步幂、多主模式下数据防止回环等。
- [了解更多](/pidts/introduction)
- [Github](https://github.com/pi-plan/pidts)


### PiLCS
PiLCS(Pi Logical clock Server) 是一个高可用、高性能的「逻辑时间」服务，可以生成唯一 ID、消息递增版本号、分布式事务 ID 的单调递增序列服务。单调递增序列不仅在单机事务中，在分布式事务和分布式系统中都有着重要的作用，甚至是必要条件。PiLCS 以 Raft 为基础，为分布式架构提供了高性能、高可用的单调序列服务。
- [了解更多](/pilcs/introduction)
- [Github](https://github.com/pi-plan/pilcs)

### PiMMS
PiMMS(Pi Meta Data Manager Server) 是一个高可用的元数据管理中心，负责同步和管理各种元数据和协调各个组件之间的规则。
- [Github](https://github.com/pi-plan/pimms)


### PRaft
PRaft 是一个 Python 实现的异步、高性能 [Raft](https://raft.github.io/) [共识算法](https://en.wikipedia.org/wiki/Consensus_(computer_science))库。不仅实现了 Leader 选举、日志复制这些基本逻辑，也实现了是线上环境中实用的 PreVote 逻辑、状态机快照与日志清理、Raft 集群成员动态变更、Leader 节点转移、无投票权成员 Leaner 的特性。
- [了解更多](/praft/introduction)
- [Github](https://github.com/pi-plan/praft)


## 各个组件之间协作关系
![各组件之间的系统全景图](/static/pi-plan-system-landscapediagram.png)

## 参与

欢迎每个想参与 Pi 计划的同学，有 Bug 、建议、反馈等可以在 [GitHub](https://github.com/pi-plan) 上对应的代码库里创建 Issue。我们会及时跟进反馈。如果您愿意参与代码的开发，可以直接 Pull requests。