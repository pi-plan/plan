# Pi 计划
Pi 计划的愿景是：让开发这件事变的简单、高效、可靠、稳定。她是一个持续性计划，会产出、更新维护的一系列相关组件、库、服务、文章、思想等。因为开发并不仅仅是代码的编写、执行、项目管理、进度管理、团队协作、跨部门沟通等等。

## Pi 计划第一阶段
提供基于 RDBMS 的异地多活系统和方案。其中需要的组件都是经过抽象，可以被其他业务/场景所复用的。 这些组件并不是为了多活而定制的，只是可以为多活提供支持，在各自的场景中可以独立被使用。

### PiDTS
PiDTS(Pi Data transmission server) 数据传输服务。支持不同数据库之间的数据传输服务。例如 MySQL -> PostgreSQL、MySQL -> Hive 等。支持全量同步、时时同步、数据同步幂等、数据防止回环。
- [了解更多](/pidts/introduction)
- [Github](https://github.com/pi-plan/pidts)

### PiDAL
PiDAL(Pi Data Access Layer) 是一个安全、易用、支持在线 Reshading、数据安全保障、支持跨 Sharding 事务的数据库中间件。不仅支持数据库在线扩容也支持数据库 Sharding 规则的重新调整。对跨 Sharding 的事务不仅提供了基于 XA 规范的分布式事务，还提供了异步、无入侵、高性能的分布式事务模式。
- [了解更多](/pidal/introduction)
- [Github](https://github.com/pi-plan/pidal)

### PiDBSidecar
在容器化实施的比较完善的场景下，PiDAL 以 Sidecar 的模式提供支持，可以降低不必要的性能消耗。具体的逻辑实现都是由 PiDAL 实现。DB Proxy 和 Sidecar 模式之间的区别和优缺点对比详情可以 [点击这里查看](/pidal/introduction?id=driver、sidecar、dbproxy)。
- [了解更多](/pidal/introduction?id=driver、sidecar、dbproxy)
- [Github](https://github.com/pi-plan/pidts)

### PiDBProxy
在容器化不够完善或者有中心化之类的需求，PiDAL 以 DB Proxy 的模式提供支持，能带来中心化、静态入口等优势。和 PiSidecar 一样底层都是 PiDAL，只是部署方式不同。DB Proxy 和 Sidecar 模式之间的区别和优缺点对比详情可以 [点击这里查看](/pidal/introduction?id=driver、sidecar、dbproxy)。
- [了解更多](/pidal/introduction?id=driver、sidecar、dbproxy)
- [Github](https://github.com/pi-plan/pidts)

### PiLCS
PiLCS(Pi Logical clock Server) 是一个高可用、高性能的「逻辑时间」服务，可以生成唯一 ID、消息递增版本号、分布式事务 ID 的单调递增序列服务。单调递增序列不仅在单机事务中，在分布式事务和分布式系统中都有着重要的作用，甚至是必要条件。PiLCS 以 Raft 为基础，为分布式架构提供了高性能、高可用的单调序列服务。
- [了解更多](/pilcs/introduction)
- [Github](https://github.com/pi-plan/pilcs)

### PRaft
PRaft 是一个 Python 实现的异步、高性能 [Raft](https://raft.github.io/) [共识算法](https://en.wikipedia.org/wiki/Consensus_(computer_science))库。不仅实现了 Leader 选举、日志复制这些基本逻辑，也实现了是线上环境中实用的 PreVote 逻辑、状态机快照与日志清理、Raft 集群成员动态变更、Leader 节点转移、无投票权成员 Leaner 的特性。
- [了解更多](/praft/introduction)
- [Github](https://github.com/pi-plan/praft)


## 各个组件之间协作关系
![各组件之间的系统全景图](/static/pi-plan-system-landscapediagram.png)

## 参与

欢迎每个想参与 Pi 计划的同学，有 Bug 、建议、反馈等可以在 [GitHub](https://github.com/pi-plan) 上对应的代码库里创建 Issue。我们会及时跟进反馈。如果您想参与代码的开发，可以直接提交 Pull requests。