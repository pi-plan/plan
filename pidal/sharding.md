# 分库分表
配置分库分表，先了解下 PiDAL 中的概念。

## 概念 & 术语
### DB Sharding/数据库分片
DB Sharding/数据库分片是一种横向分区的数据库架构模式，所谓的横向分区技术就是将一个表中的数据按行拆分到多个不同的库表中，这些不同的库表被称作分区/Sharding。每个分区都拥有相同的模式和相同的列，但是数据行却完全不同。同样的，每一个分区中的数据都是唯一的，并且独立于其他的分区。

### DBSKey (DB Sharding Key)
DB Sharding Key（简写 DBSKey）是数据表的概念，比如对 user 表进行分表，共分为 16 张表，分别对应user_0 到 user_15，规则是表中 `user_id` 字段除以 `16` 取余数，得到的余数就是对应的真是表。那么 `user_id` 这个字段就是 DBSKey。

### Sharding 算法
是指对数据、Zone 进行分配的时候采用的算法，比如根据 user id 求余数、根据注册地址等，这类算法被称为 Sharding 算法。

### 逻辑表、真实表
数据库分库分表的实践中，比如 `user` 表被分为 16 表，分别对应的是 `user_0` 到 `user_15`，对于客户端来说是感知不到 `user_0` 的存在，只需要使用 `user` 表即可，PiDAL 会透明的处理掉分表的逻辑。这里的 `user` 被称为「逻辑表」，`user_0` 到 `user_15` 被称为「真实表」。

### 数据节点
在 PiDAL 的分库分表的中，真实表 `user_0` 可能落在机器 A 上，而，`user_1` 可能落在机器 B 上，关联的 `order` 表落在机器 C 上。这里的机器 A，B，C 就是指数据节点。

### 原始表
原始表是 PiDAL 中的一种表类型，这种表不做分表，关于这个表的操作直接会不经过处理直接发送到对应的数据节点上。

### Sharding 表
Sharding 表是 PiDAL 中的一种表类型，PiDAL 会根据配置的分表规则，解析出 SQL 把操作分发到对应的数据节点上。客户端完全透明。

### 双 Sharding 表
双 Sharding 表是 Sharding 表的加强模式在 Sharding 表增加了一层判断使用那个 sharding 数据的逻辑。对与客户端来说这些逻辑都是透明的。

## 配置

```json
{
  "db": {
    "nodes": [
      {
        "type": "source",
        "name": "db0",
        "dsn": "mysql://root:@127.0.0.1:3306/test",
        "minimum_pool_size": 1,
        "maximum_pool_size": 100,
        "acquire_timeout": 5,
        "wait_time": 100
      },
      {
        "type": "replica",
        "name": "db01",
        "dsn": "mysql://root:@127.0.0.1:3306/test",
        "maximum_pool_size": 100,
        "wait_time": 100,
        "follow": "db0"
      },
      {
        "type": "source",
        "name": "db1",
        "dsn": "mysql://root:@127.0.0.1:3306/test",
        "wait_time": 100,
        "maximum_pool_size": 100
      }
    ],
    "tables": [
      {
        "type": "raw",
        "name": "test_raw",
        "lock_key": "PRIMARY",
        "strategies": [
          {
            "backends": [
              "db0.test_raw"
            ]
          }
        ]
      },
      {
        "type": "sharding",
        "name": "test_sharding",
        "lock_key": "user_id",
        "strategies": [
          {
            "backends": [
              "db0.test_s_0",
              "db0.test_s_2",
              "db1.test_s_{1, 4, 2}"
            ],
            "sharding_columns": [
              "user_id"
            ],
            "algorithm": "mod",
            "algorithm_args": [
              4
            ]
          }
        ]
      },
      {
        "type": "double_sharding",
        "name": "test_double_sharding",
        "lock_key": "tu",
        "strategies": [
          {
            "backends": [
              "db0.test_d_0",
              "db0.test_d_2",
              "db1.test_d_{1, 4, 2}"
            ],
            "sharding_columns": [
              "user_id"
            ],
            "algorithm": "mod",
            "algorithm_args": [
              4
            ]
          },
          {
            "backends": [
              "db0.test_d1_0",
              "db0.test_d1_2",
              "db1.test_d1_{1, 4, 2}"
            ],
            "sharding_columns": [
              "topic_id"
            ],
            "algorithm": "mod",
            "algorithm_args": [
              4
            ]
          }
        ]
      }
    ]
  }
}
```
这是 PiDAL 中配置分库分表的配置结构。

| 字段 | 例子 | 用途 |
| --- | --- | --- |
| nodes | [] | 是一个列表，主要是配置数据节点的信息。 |
| nodes[*].type | source | 是主数据库还是从数据库，用于在读写分离的时候使用不同的数据节点 source=主节点，replica=从节点 |
| nodes[*].name | db0 | 数据节点的唯一标识，在配置表的时候用这个名字来区分不同的表在那个数据节点上。 |
| nodes[*].dsn | mysql://root:@127.0.0.1:3306/test | 数据连接信息，包括账号、密码、IP、密码、数据库以及其他连接参数。|
| nodes[*].minimum_pool_size | 5 | 连接池最小值。 |
| nodes[*].maximum_pool_size | 100 | 连接池最大值。 |
| nodes[*].acquire_timeout | 5 | 从连接池申请连接等待的最大时间，超过时间即认为超时。 |
| nodes[*].wait_time | 3600 | 连接最大闲置时间，超过这个时间，连接池会销毁这个连接。 |
| nodes[*].follow | db0 | 只有 type 是 replica 才会用到，用来表示从数据是和那个主数据库保持保持同步的。|
| tables | [] | 是一个表，里面每一项都是一个逻辑表的配置信息 |
| tables[*].type | sharding | 表类型，raw=原始表，sharding=Sharding 表，double_sharding=双 Sharding 表 |
| tables[*].name | test_sharding | 数据的逻辑表名字，客户端在操作的时候使用这个名字，不需要理解真实表名字 |
| tables[*].lock_key | PRIMARY | 唯一性索引的名字，在处理分布式事务的时候，根据这个索引里包含的字段值来确认唯一性。 |
| tables[*].strategies | [] | 分表策略列表，至少需要有一个，双 Sharding 表需要有两个。 |
| tables[*].strategies[*].sharding_columns | ["user_id"] | 分表的时候，根据那个字段来分表 |
| tables[*].strategies[*].algorithm | mod | 分表算法的名字，比如 mod 代表求余数 |
| tables[*].strategies[*].algorithm_args | [4] | 分表算法需要的参数，比如在 mod 算法里面这里需要配置被除数。 |
| tables[*].strategies[*].backends | [] | 字符串列表，里面每一项都是一个真实表的信息。 |
| tables[*].strategies[*].backends.* | db1.test_s_2 | 真实表的信息，「.」之前的是数据节点的名字，必须要在 nodes[*].name 中存在。`test_s_2` 会被解析为 `test_s_` 前缀和序号 `2`，db1.test_s_2 表示的是序号为 `2` 的表在 `db1` 数据节点，表的前缀是 `test_s_`。|
| tables[*].strategies[*].backends.* | db1.test_s_{1, 4, 2} | 真实表的的一种快捷配置语法，这种配置等同于，同时配置 db1.test_s_1， db1.test_s_3 两个配置。|