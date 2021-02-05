# 分布式事务

PiDAL 支持 A2PC 和 simple 两种事务模式，即便是在运行时，也可以根据需要自行选择使用那种事务模式。

## Simple 事务模式

Simple 事务模式就是依赖简单数据节点自己的本地事务，PiDAL 只做了简单的事务的启动、提交或者回滚、如果这其中有异常，Simple 也不会提供补偿或者保证机制。Simple 事务模式事务中所有的读写都在一个数据节点的场景中。

## A2PC 事务模式
A2PC 事务模式基于 [A2PC 协议](/a2pc/introduction)实现的分布式事务模式，具有安全，高性能等优势的分布式事务模式，PiDAL 推荐使用此模式。

## 配置
```json
{
  "db": {
    "name": "test_database",
    "transaction_mod": "a2pc",
    "idle_in_transaction_session_timeout": 5000,
    "a2pc": {
      "servers": [
        {
          "host": "127.0.0.1",
          "port": 9025
        }
      ],
      "backends": [
        {
          "type": "source",
          "name": "tdb0",
          "dsn": "mysql://root:@127.0.0.1:3306/test",
          "minimum_pool_size": 1,
          "maximum_pool_size": 100,
          "acquire_timeout": 5,
          "wait_time": 100
        },
        {
          "type": "source",
          "name": "tdb1",
          "dsn": "mysql://root:@127.0.0.1:3306/test",
          "minimum_pool_size": 1,
          "maximum_pool_size": 100,
          "acquire_timeout": 5,
          "wait_time": 100
        }
      ]
    }
  }
}
```
这是 PiDAL 关于事务的配置例子。

| 字段 | 例子 | 用途 |
| --- | --- | --- |
| name | test_database | 数据库名字 |
| transaction_mod | a2pc | 默认的事务模型，当客户端开启一个事务的时候没有指定事务模型的时候，使用此处配置的默认模型 |
| idle_in_transaction_session_timeout | 50 |  事务默认闲置超时时间，事务最后一个活动后，空闲超过此时间，默认将事务回滚。 |
| a2pc | {} | a2pc 事务模型需要用到的配置 |
| a2pc.servers | [] | A2PC 中 RM 的服务地址，可以同时有多个。 |
| a2pc.backends | [] | A2PC 中 RM 使用的数据库的节点配置|
其他同名配置向含义和 [分库分表配置](/pidal/sharding?id=配置) 相同。