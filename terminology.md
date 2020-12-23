# 术语表

## Zone
Zone 是一份能完整支撑业务运行的逻辑单元，可以是一个独立的物理机房，也可以是一个逻辑机房。在 Pi 系统里是逻辑上的一个业务完整运行单元。

## Zone Sharding
Zone Sharding 是指把业务逻辑分配到不同的 Zone 执行的过程。和 DB Sharding 类似，只是更在以 Zone 层面上。

## ZSID (Zone Sharding ID)
Zone Sharding ID（简写ZSID）是用来分配 Zone 连续序列序列，范围的大小不受限制可以是 2^10 或者更大的数字。比如 Zone Sharding ID 有 0 ～ 63 个，0 ～ 31 由 Zone 1 处理，32 ～ 63 由 Zone 2 进行处理。每个业务根据自己的需要比如根据用户 ID、IP、设备等信息计算出 Zone Sharding ID，然后根据规则落地到不同的 Zone 执行。

## ZSKey (Zone Sharding Key)
Zone Sharding Key（简写 ZSKey）是业务、数据库的概念，比如通过用户 ID 计算出来 Zone Sharding ID 那么用户 ID 就叫做 Zone Sharding Key。在数据层面，通过 `user_id` 字段计算出来 Zone Sharding ID 那么，这个字段就叫做 Zone Sharding Key。每个 Zone Sharding Key 得到的 ZSID 都是固定的，Zone Sharding 规则变更是值调整 ZSID 的规则，由此来实现 Zone Sharding 的改变。
