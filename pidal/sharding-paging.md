# 跨 Sharding 分页的难题

分页需求在互联网业务中是很常见的需求。在单机模式下，通过 `offset 20 limit 10` 是很容易实现的。但是当数据量达到一定量级，对一个表的数据进行[分库分表](/pidal/sharding)之后，事情就变得麻烦了。不管使用那种 sharding 算法，都需要解决这个问题。下面就介绍几种解决方法。

假设场景为：分页拉取为 ID 为 1 的视频点赞的用户。数据按照 `user_id % 2` sharding，两个表的表结构一致：
```
CREATE TABLE `user_like_video` (
 `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增ID',
 `user_id` bigint(20) NOT NULL DEFAULT '0' COMMENT '用户 ID',
 `video_id` bigint(20) NOT NULL DEFAULT '0' COMMENT '视频 ID',
 `create_time` bigint(20) NOT NULL DEFAULT '0' COMMENT '创建时间',
 PRIMARY KEY (`id`),
 UNIQUE KEY un_user_video(user_id, video_id)
) ENGINE=InnoDB
```
两个表各自的数据为
user_like_video_1 :

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 1 | 1 | 10000001 |
| 3 | 1 | 10000003 |
| 5 | 1 | 10000005 |
| 7 | 1 | 10000007 |
| 9 | 1 | 10000009 |
| 11 | 1 | 10000011 |
| 13 | 1 | 10000013 |
| 15 | 1 | 10000015 |

user_like_video_2 :

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 2 | 1 | 10000002 |
| 4 | 1 | 10000004 |
| 6 | 1 | 10000006 |
| 8 | 1 | 10000008 |
| 10 | 1 | 10000010 |
| 12 | 1 | 10000012 |
| 14 | 1 | 10000014 |
| 16 | 1 | 10000016 |

## 一些方法的介绍
### 全局视野法
比如原始 SQL 为：`select user_id from user_like_video where video_id = 1 order by create_time offset 6 limit 2`，标示要获取第 6 到 8 个数据。全局视野法或分表向两张表获取前 offset + limit 行数据。
```
# user_like_video_1 查询语句
select create_time, user_id from user_like_video_1 where video_id = 1 order by create_time offset 0 limit (6 + 2)

# user_like_video_2 查询语句
select create_time, user_id from user_like_video_2 where video_id = 1 order by create_time offset 0 limit (6 + 2)
```
这样拿到两个表最前面的 8 条数据，然后对这 8 条数据进行排序，从拿到第 6 到 8 条。
#### 优点：
1. 可以精准的返回需要的数据。

#### 缺点：
1. 数据量大，需要所有 sharding 返回从 0 开始的所有数据，也增加了网络传输。
2. 需要对每个 sharding 进行重新排序和计算。增加二次消耗。
3. 随着 offset 的增加性能会下降，比如放 offset 到 1000 的时候，就需要每个 sharding 返回 1000 多条数据。需要在内存中排序 2000 多条数据。这种方式在实际场景中几乎不会被采用。

### 只提供下一页功能
这种需要业务允许才能使用，比如在上文提到的场景中，需要每次请求提供 `create_time` 作为辅助。
每页获取 2 条数据。获取第一页数据的时候，因为没有 `create_time`，所以 `create_time` 当作 0 处理，分表向两个 sharding 发送查询：
```
# user_like_video_1 查询语句
select create_time, user_id from user_like_video_1 where video_id = 1 and create_time > 0 order by create_time offset 0 limit 2

# user_like_video_2 查询语句
select create_time, user_id from user_like_video_2 where video_id = 1 and create_time > 0 order by create_time offset 0 limit 2
```
这样总共拿到 4 条数据。内存按照 create_time 排序后返回最前面两条结果。
| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 1 | 1 | 10000001 |
| 2 | 1 | 10000002 |
| 3 | 1 | 10000003 |
| 4 | 1 | 10000004 |

获取下一页的时候带上上次返回的结果里面的 `create_time` `10000002`，这样分表向两个 sharding 发送的查询：
```
# user_like_video_1 查询语句
select create_time, user_id from user_like_video_1 where video_id = 1 and create_time > 10000002 order by create_time offset 0 limit 2

# user_like_video_2 查询语句
select create_time, user_id from user_like_video_2 where video_id = 1 and create_time > 10000002 order by create_time offset 0 limit 2
```
对结果里的 4 条语句在进行排序、返回即可。
#### 优点
1. 计算量小，每个 sharding 只返回一页的数据量。
2. 性能稳定，不会随着页数的增加而引起性能下降。
#### 缺点
1. 需要业务配合传输排序依赖的字段。
2. 不能跳跃页数，只能一页一页的顺序拉取。

### 二次查询法
二次查询法本来不在考虑之列因为这种方法可能会返回错误的结果。这种方法只能适用于 sharding 分布十分均匀的场景。以上面的场景为例子：  
比如原始 SQL 为：`select user_id from user_like_video where video_id = 1 order by create_time offset 6 limit 2`，标示要获取第 6 到 8 个数据。二次查询法首先发起第一次查询，向两个 sharding 查询 `offset floor(6/2) limit 2` 的结果：
```
# user_like_video_1 第一次查询语句
select create_time, user_id from user_like_video_1 where video_id = 1 order by create_time offset floor(6/2) limit 2

# user_like_video_2 第一次查询语句
select create_time, user_id from user_like_video_2 where video_id = 1 order by create_time offset floor(6/2) limit 2
```
这样返回 4 条数据：  
user_like_video_1

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 7 | 1 | 10000007 |
| 9 | 1 | 10000009 |

user_like_video_2

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 8 | 1 | 10000008 |
| 10 | 1 | 10000010 |

然后从 4 条结果中找到最小的 `create_time` `10000007` ，然后在获取每个 sharding 结果里面最大的 `create_time` 分别是：`10000009`、`10000010`。这之后发起第二次查询，查询 sharding 里面 `create_time between 10000007 and 当前 sharding 最大值`：

```
# user_like_video_1 第二次查询语句
select create_time, user_id from user_like_video_1 where video_id = 1 and create_time between 10000007, 10000009

# user_like_video_2 第二次查询语句
select create_time, user_id from user_like_video_2 where video_id = 1 and create_time between 10000007, 10000010
```
这样返回：


| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 7 | 1 | 10000007 |
| 9 | 1 | 10000009 |

user_like_video_2

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 8 | 1 | 10000008 |
| 10 | 1 | 10000010 |

从第一次查询可以知道 10000007 在 sharding 里面的顺序是 `4`，10000008 在 sharding 里面的顺序也是 `4`，由此可以知道 `create_time < 10000007` 的一共有 6 个，所有就能确认每条数据在全局的排序。

| user_id | video_id | create_time | 全局顺序 |
| :-- | :-- | :-- | :- |
| 7 | 1 | 10000007 | 7 |
| 8 | 1 | 10000008 | 8 |
| 9 | 1 | 10000009 | 9 |
| 10 | 1 | 10000010 | 10 |

这样从结果及里面取第 6 到 8 条数据即可。

**但是如果 sharding 分布不均匀。**结果就不准确了。比如当前每个 sharding 的数据分布情况为：

user_like_video_1 :

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 1 | 1 | 10000001 |
| 3 | 1 | 10000003 |
| 5 | 1 | 10000005 |
| 7 | 1 | 10000007 |
| 9 | 1 | 10000009 |


user_like_video_2 :

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 6 | 1 | 10000006 |
| 8 | 1 | 10000008 |

这个时候原始 SQL ：  
`select user_id from user_like_video where video_id = 1 order by create_time offset 2 limit 2`  
第一次查询：  
`select create_time, user_id from user_like_video_x where video_id = 1 order by create_time offset floor(${offset}/n) limit ${limit}`   
获取到结果为：

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 3 | 1 | 10000003 |
| 5 | 1 | 10000005 |

user_like_video_2

| user_id | video_id | create_time |
| :-- | :-- | :-- |
| 8 | 1 | 10000008 |

然后在执行第二次查询  
`select create_time, user_id from user_like_video_x where video_id = 1 and create_time between ${min}, ${max}`   
返回的结果汇总，但是在计算全局排序的时候，因为 10000008 是在自己的sharding 是第二，10000006 才是第一，这就造成 10000003 在全局里面的顺序无法被正确计算。造成返货错误的结果。

| user_id | video_id | create_time | 全局顺序 | 正确的全局顺序 |
| :-- | :-- | :-- | :-- | :-- |
| 3 | 1 | 10000003 | 3 | 2 |
| 5 | 1 | 10000005 | 4 | 3 |
| 6 | 1 | 10000006 | 5 | 4 |
| 8 | 1 | 10000008 | 6 | 5 |

而且放一个 sharding 里面数据为空的时候情况更加明显。
所以这个方法在实际场景中很难保证绝对的均匀那么这个方法出问题的概率就会很大。不推荐使用。

## 有没有完美的方法？
上面介绍的方法或多或少都有问题和局限。那么有没有完美的方法呢？首先我们看一下为什么上面的方法都不行？是因为查询需要跨 sharding。为什么需要跨 sharding？因为数据分片的时候是以 user_id 为标准所以在从 video_id 角度查询的时候是无法在同一个 sharding 上完成的？那么如果从 video_id 角度对数据做分片，这样同时有 video_id 和 user_id 这种分片是不是就可以了？

答案是「**是的。**」。

但是，如果数据有两份那就需要保证两份数据是一致的。如果是从一张表异步同步到另一张表，又会有延迟的问题，最好是在同一个操作里面，同时维护两个维度的 sharding。  


PiDAL 里的「**双 sharding 表**」就是为这一问题设计的结局方案。
1. 自动切换 sharding，PiDAL 会解析 SQL，自动分析出应该使用那个 sharding，避免出现跨 sharding 的分页。
2. 一致性保障，借助 PiDAL 的[事务保障](/pidal/transaction) 机制，可以保障两份数据一致。
3. 实时性保障，数据在写入的时候实现双写，不会出现异步延迟之类的问题。

[如何使用双 sharding 表？]()