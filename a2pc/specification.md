# A2PC 规范

虽然 PiDAL 实现了 A2PC 协议，但 A2PC 只是一个思想，落地和实现形式可以是多种多样的，为了满足有自己场景需求的落地 A2PC，同时也尽量减少重复性的工作，对 A2PC 的落地定出此规范，尽量可以复用满足自己需求的实现。减少不必要的开发、维护。

## 角色
A2PC 共有 TM，RM，APP 三种角色。三种角色缺一不可。在实现的时候时候可以一个程序同时身兼多个角色。

### TM
事务管理器(Transaction Manager ，简称 TM)，该角色负责管理全局事务，分配事务唯一 ID，监控事务的执行进度，并负责事务的提交、回滚、失败的恢复等。任何一个事务，在任何时间只能被一个 TM 处理，在高可用模式下，TM 可以做热切换，甚至分片，但是需要保证任何一个时间只能被一个 TM 实例处理。如果同时有两个 TM 处理同一个事务，将会演变成分布式事务。

TM 负责处理的行为有：

1. 接收和处理来自 APP 的 begin 行为。开启一个分布式事务，负责生成 XID 。[可选]
2. 接收和处理来自 APP 的 acquire_lock 行为，并负责注册一个分支事务，并负责维护全局锁状态。
3. 接收和处理来自 APP 的 commit 行为，执行提交事务，修改该事务状态。
4. 接收和处理来自 APP 的 rollback 行为，执行事务的回滚，并向每个分支事务中的 RM 发起 rollback_branch 分支事务回滚操作。
5. 自动发起 compensation 事务补偿行为，在部分执行失败、超时的行为后能够进行补偿。
6. 自动发起 transaction_timeout 事务超时行为，对事务执行时间超过指定时间的事务进行判断和回滚。[可选]

### RM
资源管理器(Resource Manager，简称 RM)，该角色负责管理和共享的资源，许多 APP 都可以访问和使用这些资源，RM 需要能够正常的处理本地事务，锁竞争等逻辑。并且也需要能够支持实现 reundo log 的存储和查询。也需要支持 A2PC 行为的幂等操作，以便于补偿行为的运行。

RM 负责处理的行为有：

1. 接收和处理来自 APP 的 begin_local 开启分支事务行为。
2. 接收和处理来自 APP 的 acquire_local_lock 行为。并负责维护本地锁的状态。
3. 接收和处理来自 APP 的 dml 行为。负责数据 dml 的正确执行。
4. 接收和处理来自 APP 的 write_reundo_log 行为，并提供机制可以确保和 APP 的 dml 行为在同一个本地事务中。
5. 接收和处理来自 APP 的 commit_local 提交分支事务行为。确保 dml 和 reundo log 被一同提交。
6. 接收和处理来自 APP 的 rollback_local 回滚分支事务行为。确保 dml 和 reundo log 被一同回滚。
7. 接收和处理来自 TM 的 rollback_branch 行为，并完成分支事务回滚操作。

### APP
其中应用程序(Application Program，简称 APP)，APP 负责定义全局事务和分支事务的边界，还负责操作 RM 的资源，同时还需要负责生成和插入 reundo log。

负责的行为有：

1. 向 TM 发起 begin 行为。开启一个分布式事务。[可选]
2. 向 TM 发起 acquire_lock 行为，申请全局锁。
3. 向 RM 发起 begin_local 开启分支事务行为。
4. 向 RM 发起 dml 行为。操作 RM 上的资源。
5. 生成 reundo log ，并向 RM 发起 write_reundo_log 行为。配合 RM 一起确保 write_reundo_log 和 dml 行为在同一个本地事务中。
6. 向 TM 发起 acquire_lock 行为，申请全局锁，并能正确的处理超时回滚。
7. 向 RM 发起 commit_local 行为，提交本地事务。
8. 向 RM 发起 rollback_local 行为，回滚本地事务。
9. 向 TM 发起 commit 行为，提交全局事务。
10. 向 TM 发起 rollback 行为，回滚全局事务。

## 数据结构

#### XID
XID 是一个单调递增的 int64 位数字，是用来区分一个全局事务的唯一标识，在实现「读已提交」等能力的时候，XID 作为事务发生时间先后循序的判断一句。生成 XID 的角色需要确保生成的 XID 一定是单调递增的，具体实现方式可以参考 [PiLCS](/pilcs/introduction) 服务的实现。

#### Undo Log
Undo Log 是数据变更前的镜像，在事务滚回的时候通过 undo log 确保数据回复到事务启动时候的状态，也可以通过 undo log 实现不同级别的事务隔离。undo log 需要包含 RM 的数据资源中的每一个字段。比如一张数据表。
```sql
CREATE TABLE `product` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `product_name` bigint(20) NOT NULL,
  `stock` bigint(20) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
那么对应生成的 undo log 用 josn 描述就是 ：
``` json
[
    {
        "id": 2,
        "product_name": "测试商品",
        "stock": 10
    }
]
```
这里是个数组，是因为要兼容会有多行数据的场景。如果是数据创建的操作，undo log 可以是 `null`。

#### Redo Log
Redo Log 是数据变更后的数据镜像，通过 redo log 可以在需要的场景下对数据进行校验和补偿。格式和要求与 undo log 完全一致。

#### reundo_log 表结构
如果是关系型数据库作为 RM，那么 reundo log 的表结构是：
```sql
CREATE TABLE `reundo_log` (
  `xid` bigint(20) NOT NULL AUTO_INCREMENT,
  `lock_key` varchar(255) NOT NULL,
  `context` varchar(2000) DEFAULT NULL,
  `reundo_log` longblob NOT NULL,
  `table` varchar(128) NOT NULL,
  `client_id` varchar(64) DEFAULT NULL,
  `create_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  `update_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (`id`),
  KEY `un_xid` (`xid`),
  KEY `un_undo_log` (`table`,`lock_key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
如果 RM 不是关系型数据库，那么可以有自定义的实现形式，不过需要包含和满足下表的数据结构要求。

| 字段 | 类型 | 长度 | 例子 | 说明 |
| ---- | --- | --- | --- | --- |
| xid  | int64 | 64 位 | 10927 | 用来存储事务的唯一标识 |
| lock_key | string | 255 个字符| 'id=2'| 用来区分事务中变更数据的唯一标识 |
| context | string | 自定义 | | 用来存储事务所在的一些上下文 |
| reundo_log | string | 尽量大 | {"undo":[{"id":2,"product_name":"测试商品","stock":10}],"redo":[{"id":2,"product_name":"测试商品","stock":7}]} | 用来存储事务中生成的 reundo log |
| table | string | 128 字符 | order | 同一类数据的集合名字，比如数据库中的表 |
| client_id | string | 64 字符 | 10.24.1.123 | 客户端的标识，便于追踪事务 |
| create_time | 时间 | 2020-10-04 21:08 :44.259888 | 精确到微秒 | 事务创建时间，便于追踪事务 |
| update_time | 时间 | 2020-10-04 21:08 :44.259888 | 精确到微秒 | 事务最后一次修改的时间，便于追踪事务|

#### RPC 请求对象结构
在 APP 和 RM 之间的 RPC 调用中，请求参数的数据结构，用 json 描述为：
```json
{
  "action": "begin",
  "xid": 1024,
  "node": "db0",
  "table": "order",
  "lock_key": "id=2",
  "context": null
}
```
每个字段的含义：

| 字段 | 类型 | 长度 | 例子 | 说明 |
| ---- | --- | --- | --- | --- |
| action | string | 64 字符 | begin | 是行为的名字，只能取自 A2PC 中的行为名字。 |
| xid | int64 | 64 位 | 1024 | XID，当前事务的唯一标识 |
| node | string | 64 字符 | db01 |数据节点的唯一标识，或者 IP + 端口 |
| table | string | 128 字符 | order | 同一类数据的集合名字，比如数据库中的表 |
| lock_key | string | 255 字符 | id=2 | 用来区分事务中变更数据的唯一标识 |
| context | string |  |  | 事务运行的上下文。 |

其中的 `action` `xid` 字段是必须的，其他的可以根据 RPC 行为的不同为取舍。

#### RPC 响应对象结构
RPC 调用时的返回值，其数据结构用 json 描述位：
```json
{
  "status": 0,
  "xid": 1024,
  "msg": null
}
```
每个字段的含义为：

| 字段 | 类型 | 长度 | 例子 | 说明 |
| ---- | --- | --- | --- | --- |
| status | int | 32 位 | 0 | RPC 请求的运行结果，0 标识正常运行了，其他值表示遇到错误 |
| xid | int64 | 64 位 | 1024 | XID，当前事务的唯一标识 |
| msg | string | 128 字符 |  | 可读的执行结果或者错误信息。 |

## 行为

#### BEGIN
开启一个分布式事务，并生成 XID，如果 XID 由 APP 生成，这一步骤行为可以省略，在第一次 ACQUIRE_LOCK 的时候创建 一个事务对象即可。属于 APP 和 TM 之间的 RPC 调用，参数和响应值遵循 RPC 对象数据结构。

#### ACQUIRE_LOCK
获取数据的全局锁，如果出现锁竞争会等待到指定时间，然后返回 APP 无法获取到全局锁。属于 APP 和 TM 之间的 RPC 调用，参数和响应值遵循 RPC 对象数据结构。

#### COMMIT
提交一个全局事务。属于 APP 和 TM 之间的 RPC 调用，参数和响应值遵循 RPC 对象数据结构。

#### ROLLBACK
回滚一个全局事务。属于 APP 和 TM 之间的 RPC 调用，参数和响应值遵循 RPC 对象数据结构。

#### COMPENSATION
事务补偿行为，是 TM 自己发起的，主要是针对部分分支事务 rollback 失败的时候的补偿行为。也可以用来压缩 reundo log。

#### TRANSACTION_TIMEOUT
全局事务执行时间超时，可选行为，由 TM 主动发起，针对部分全局事务执行超过一定的时间限制，将事务置为回滚，回滚所有分支事务，并对后续的 commit 行为报错，对 rollback 行为幂等。

#### BEGIN_LOCAL
开启一个本地事务，是 APP 向 RM 发起的行为，如果 RM 是关系型数据库，遵循 SQL 标准。

#### ACQUIRE_LOCAL_LOCK
获取数据的本地锁，是 APP 向 RM 发起的行为，如果 RM 是关系型数据库，遵循 SQL 标准和数据库实现。

#### DML
对数据的增删该查行为，是 APP 向 RM 发起的行为，如果 RM 是关系型数据库，遵循 SQL 标准。

#### WRITE_REUNDO_LOG
创建数据变更的 reundo log 是 APP 向 RM 发起的行为，如果 RM 是关系型数据库，遵循 SQL 标准。

#### COMMIT_LOCAL
提交一个本地事务，是 APP 向 RM 发起的行为，如果 RM 是关系型数据库，遵循 SQL 标准。

#### ROLLBACK_LOCAL
回滚一个本地事务，是 APP 向 RM 发起的行为，如果 RM 是关系型数据库，遵循 SQL 标准。

#### ROLLBACK_BRANCH
回滚一个分支事务，是 TM 向 RM 发起的行为，本质上就是一次本地数据的操作。

## 基本原则
### 第一阶段
1. 在第一阶段遇到任何无法确认状态的异常，一律当作失败处理。可以重试或回滚。

### 第二阶段
1. 所有行为必须支持幂等操作。
2. 一旦 `ROLLBACK` 之后，必须在确保每个分支任务已经回滚后才能进行下一步操作。
