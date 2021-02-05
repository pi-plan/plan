# PiDTS
PiDTS(Pi Data transmission server) 数据传输服务。支持不同数据库之间的数据传输服务。例如 MySQL -> PostgreSQL、MySQL -> Hive 等。支持全量同步、时时同步、数据同步幂、多主模式下数据防止回环等。

- [Github](https://github.com/pi-plan/pidts)

## PiDTS 组件和核心概念
### ChangeEvent 和 LSN
ChangeEvent 和 LSN 是 PiDTS 的主要对象，其中原始数据库的每次变更会被转化为 PiDTS 里的 ChangeEvent，这个概念屏蔽里不同数据库之间的差异，在处理的时候可以忽略掉原始数据库的差异。而 LSN 是每个事件的序号，用来标示每条日志。

ChangeEvent 和 LSN的定义：
```python
class ChangeEventLSN(object):
    def __init__(self):
        self.source_zone_change_no: int = 0
        self.server_id: int = 0  # 产生变更的机器 ID
        self.log_index: int = 0  # 日志文件的 编号
        self.log_position: int = 0  # 日志位置
        self.xid: int = 0
        ...... 


class ChangeEvent(object):

    def __init__(self):
        self.lsn: ChangeEventLSN  # 日志序列号
        self.prev_lsn: ChangeEventLSN  # 上一条数据的 LSN
        self.source_zone_id: int  # 产生变更事件的 Zone ID
        self.old_data: dict  # 变更前数据
        self.new_data: dict  # 变更后数据
        self.is_retry: bool  # 是否是在回滚/重试
        ......
```
`ChangeEventLSN` 是由根据固定信息生成的。同一条原始信息可以生成一样的 LSN，而且通过 LSN 也可以反推到原始格式的位置便于使用。LSN 是单调递增的，通过 LSN 对比也能知道消息的先后顺序。  
`ChangeEvent` 不仅包含自己的 LSN，还包含了上一条的 LSN，这样可以在收到消息的时候来验证是否论序或者丢失消息。

### PiDTS Replicator 和 Apply
Replicator 是负责将原始变更事件转换为 `ChangeEvent` 并推送给队列的组件，例如在对应 MySQL 的 Replicator 里，负责把 binlog 日志转化为 `ChangeEvent` 并推送给队列。`Apply` 负责从队列里接收 `ChangeEvent` 并进行校验，通过之后写入到目标数据库。

## 数据同步顺序和是否丢失保障
Replicator 是从已经持久化数据库实例读取日志，所以读取的时候顺序有保障的，也不会丢失消息，除非 binlog 已经被删除了。这个时候通过[全量复制]()依然可以得到完整和准确的数据。

在 Apply 接收到消息的时候会校验 `prev_lsn` 是否和收到的上一个 `lsn` 判断是否一致，如果不一致有三种情况：
1. 消息乱序
2. 消息丢失
3. 消息重发

针对乱序和丢失，会停止应用到目标数据库上，然后 DTS 会自动尝试重发一段时间之前的日志，如果重试后还不行，就停止复制，因为无法保证数据的正确性，给出报警，及时排查和处理。

而面对消息重发的情况又会有两种情况：
1. 当前消息的`ChangeEvent` 因为某些原因会重复发来两次，因为 lsn 是唯一的，只需要判断 lsn 和上一个是否相等就可以做幂等处理。
2. 从已经接收到的消息之前的一段距离的日志重新发送。针对这种情况 Replicator 在发送的时候会加上 `is_retry: True` 这样 Apply 会忽略此次的 prev_lsn 校验。重复的消息会有幂等机制保障不会被老的数据覆盖掉。

## 数据同步幂等
消息的重发不论是重试还是等其他场景下，都是会作为一种保障的方式存在的。针对消息重发这种场景不能发生老的数据覆盖已经同步过来的新数据，这样就会造成数据异常。比如：

| 消息序号 | 值 |
| :-- | :-- |
| 1 | 100 |
| 2 | 101 |
| 3 | 102 |
| 4 | 103 |

最终的结果是 `103` 然后这个时候收到重发的消息

| 消息序号 | 值 |
| :-- | :-- |
| 2 | 101 |
| 3 | 102 |
| 4 | 103 |

在处理序号为 2、3、4 的时候需要一直保证值为 `103` ，如果发生变换比如 `101` -> `102` -> `103` 这样最终还是 `103` 但是，在 `101` -> `102` 的期间，值并不是 `103` ，这个时候如果有读取的行为就会读到错误的数据。所以幂等也需要保障数据的安全。

PiDTS 通过增加一个字段来确保幂等操作，如果单独 PiDTS 的时候可以给数据库表增加一个自动更新的字段
```
`update_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
```
这样 update_time 字段在每次变更的时候都会随着时间增长。如果业务场景要求很敏感不能依赖时间戳，
可以搭配 PiDAL 会自动插入一个连续递增的序号，在遇到时间漂移或者无法区分时间精度的场景下，依然能保持递增有序。
Apply 在应用 ChangeEvent 的时候会判断目标数据库是否大于 ChangeEvent 里的 update_time，只有大于的情况下才会被应用到目标数据库。这样在面对消息重发的时候幂等机制能保障数据的安全。

## 容错能力
虽然在正常的情况下可以保障不会出现问题，但是在错综复杂的网络环境中依然会产生各种意外，所以也要具备一定的容错能力。比如在遇到业务 Bug、网络突然不可用等场景下，需要能最大程度的保障数据安全和系统的正常运行。 PiDTS 在面对复杂的错误情况下提供给了数据锁定和在线全量同步两种机制来保障遇到问题的是尽量减少异常带来的影响。
### 数据锁定
在数据同步的时候，PiDTS 会在保证来顺序，幂等来保证重复等，即便全部通过前面的策略之后，在最后还会根据 old_data 进行校验，正常情况下，old_data 和数据库当前记录应该是一致的，如果无法通过校验就会锁定当前数据并报警，PiDTS 会进行校验，被锁定的数据无法被更改，一直到数据被人工修复，根据时间的经验来说除了手动触发的，自然触发数据锁定的概率大约是百万分之一。
### 消息延迟
生产环境中需要尽量保证延迟尽量低，那么造成消息延迟可能有两种原因：
1. 负载压力大
2. 网络拥堵

#### 负载压力大
对于负载压力大的情况，只需要通过减小一个节点的压力即可，比如一个 Replicator 负责同步一个表，这种情况下，单表既然都能负责业务的查询和写入，因为 Replicator 和Apply 的逻辑相比业务来说简单那些校验逻辑也只是简单的数字对比而已。常规情况下一套 Replicator -> Apply 可以负载多个表甚至一个 Zone 的同步。

#### 网络拥堵
如果是因为网络拥堵造成的那就只能通过优化网络环境的方式来进行来， PiDTS 并不会给网络带来很大的压力。优化网络配置之后就可以改善网络环境造成的消息延迟问题。

### 消息过期被丢弃
在一些场景下，一个 Zone 停止服务来一段时间，因为积累的消息太多，最早的消息被删除来，无法通过消息队列来进行同步，这种场景下 PiDTS 支持幂等的在线全量同步。不仅在这种场景下，在新增一个 Zone 的时候等场景下、在一个 Zone 数据出现严重问题的时候需要都可以不影响其他 Zone 正常运行的情况下进行数据同步。PiDTS 的在线全量同步也是可以保障数据安全的。

## 在线全量同步安全保障
PiDTS 的在线全量同步会同时进行两个操作：
1. Row Copy
2. Incremental Copy

### Row Copy
Row Copy 是复制已经存在的老数据，通过 `select * from xxx order by id` 不断的获取 src Zone 的数据，直到原始数据全部被复制完 Row Copy 操作结束。src Zone 的数据在同步到 dest Zone 的时候会被转换为 `insert ignore into` 操作。在 Row Copy 期间产生的数据变更会通过 Incremental Copy 操作被同步到 dest Zone 不会丢失。

### Incremental Copy
Incremental Copy 是实时复制通过日志变更记录来进行同步，也就是 PiDTS 正常情况下运行的数据复制模式。Incremental Copy 和 Row Copy 是可以同时进行的。 Incremental Copy 会把数据变更记录也做一次转换，会把原始的 `insert` 转换为 `insert or update`、 `update` 会根据情况转变为 `insert or update`、`delete` 操作保持不变。**在全量复制的时候以 Incremental Copy 复制的数据为准**。

### 安全保障论证
Row Copy 和 Incremental Copy 同时进行是可以保障数据不会丢失，最终的数据状态也会和 src Zone 一致。针对数据的 `insert`、`update`、`delete` 的三种操作逐一进行说明。

#### Insert
Row Copy 和 Incremental Copy 无论顺序是先后，都是会被插入的，所以 Insert 数据都是安全的。

#### Update
在全量同步的时候进行了 Update 操作，说明这条数据已经存在了。无论哪种方式总归是会被同步过去的。只是顺序的问题而已，如果 Update 操作发生在 Row Copy 之前，那么这个最新的数据会通过 Row Copy 的方式同步给 dest Zone。

如果是 Row Copy 先到达直接进行插入即可，此时数据就是最新的，如果这之后 Incremental Copy 在同步变更事件过来，就执行正常的 update 即可，因为数据是一样，不会有数据变更。 如果是 Incremental Copy 先到达会转换为 insert 操作，这个时候插入的就是最新的数据，而后续 Row Copy 到达会因为幂等机制而忽略。

如果被 Update 的数据行是先被 Row Copy 操作来之后发生的，因为 Row Copy 和 Incremental Copy 是同时进行的，那么对应的修改记录会通过 Incremental Copy 操作被同步过来。此时因为数据已经存在了，进行正常的 update 操作即可。

#### Delete
Delete 操作相对 Update 来说简单一些，如果在 Row Copy 之前进行了 Delete 操作，那么 Row Copy 就不会同步这一条数据。如果是在 Row Copy 之后进行的 Delete 操作，那么最终会被 Incremental Copy 同步过来 Delete 操作。
