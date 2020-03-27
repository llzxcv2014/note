# Redis深度历险笔记

## Redis可以用来做什么

Redis 是互联网技术领域使用最为广泛的存储中间件，它是「**Re**mote **Di**ctionary **S**ervice」的首字母缩写，也就是「远程字典服务」。

Redis最常用的就是缓存，比Memcache更易于理解、使用、控制。

1. 记录帖子的点赞数、评论数和点击数 (hash)。
2. 记录用户的帖子 ID 列表 (排序)，便于快速显示用户的帖子列表 (zset)。
3. 记录帖子的标题、摘要、作者和封面信息，用于列表页展示 (hash)。
4. 记录帖子的点赞用户 ID 列表，评论 ID 列表，用于显示和去重计数 (zset)。
5. 缓存近期热帖内容 (帖子内容空间占用比较大)，减少数据库压力 (hash)。
6. 记录帖子的相关文章 ID，根据内容推荐相关帖子 (list)。
7. 如果帖子 ID 是整数自增的，可以使用 Redis 来分配帖子 ID(计数器)。
8. 收藏集和帖子之间的关系 (zset)。
9. 记录热榜帖子 ID 列表，总热榜和分类热榜 (zset)。
10. 缓存用户行为历史，进行恶意行为过滤 (zset,hash)。

## Redis的基础使用

### Redis的基础数据结构

#### String（字符串）

Redis 所有的数据结构都是以唯一的 key 字符串作为名称，然后通过这个唯一 key 值来获取相应的 value 数据。不同类型的数据结构的差异就在于 value 的结构不一样。

常见可将JSON序列化后存进redis。

**Redis 的字符串是动态字符串**，是可以修改的字符串，内部结构实现上类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配

当字符串长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间。需要注意的是字符串最大长度为 512M

```
set name codehole
```

批量修改和获取

```
mset name1 boy name2 girl name3 unknown
mset name1 boy name2 girl name3 unknown
```

设置过期时间

```
set name codehole
expire name 5

setex name 5 codehole  # 5s 后过期，等价于 set+expire
```

计数

```
set intVal val
incr intVal
incrby intVal 5
```

范围是signed long 的最大最小值

#### list（列表）

Redis 的列表相当于 Java 语言里面的 LinkedList。

Redis 的列表结构常用来做异步队列使用。

**右边进左边出：队列**

```
rpush books python java golang
lpop books # "python"
```

**右边进右边出：栈**

```
rpush books python java golang
rpop books
```

**慢操作**

lindex 相当于 Java 链表的`get(int index)`方法，它需要对链表进行遍历，性能随着参数`index`增大而变差。

#### hash

Redis 的字典相当于 Java 语言里面的 HashMap，它是无序字典.内部实现结构上同 Java 的 HashMap 也是一致的，同样的数组 + 链表二维结构

Redis的hash只能用来存储字符串，rehash方式也不一样，redis采用了渐进hash的策略。

渐进式 rehash 会在 rehash 的同时，保留新旧两个 hash 结构，查询时会同时查询两个 hash 结构，然后在后续的定时任务中以及 hash 操作指令中，循序渐进地将旧 hash 的内容一点点迁移到新的 hash 结构中

```
hset books java "think in java"  # 命令行的字符串如果包含空格，要用引号括起来
hset books golang "concurrency in go"
hgetall books
```

#### Set(集合)

Redis 的集合相当于 Java 语言里面的 HashSet，它内部的键值对是无序的唯一的。

```
sadd books python
smembers books
sismember books java
scard books  # 获取长度相当于 count()
spop books  # 弹出一个
```

#### zset(有序集合)

类似于 Java 的 SortedSet 和 HashMap 的结合体，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以给每个 value 赋予一个 score，代表这个 value 的排序权重。它的内部实现用的是一种叫做「跳跃列表」的数据结构。

zset 可以用来存粉丝列表，value 值是粉丝的用户 ID，score 是关注时间。我们可以对粉丝列表按关注时间进行排序。

zset 还可以用来存储学生的成绩，value 值是学生的 ID，score 是他的考试成绩。我们可以对成绩按分数进行排序就可以得到他的名次。

```
zadd books 9.0 "think in java"
zadd books 9.0 "think in java"
zrange books 0 -1  # 按 score 排序列出，参数区间为排名范围
zrevrange books 0 -1  # 按 score 逆序列出，参数区间为排名范围
zscore books "java concurrency"  # 获取指定 value 的 score
zrank books "java concurrency"  # 排名
zrangebyscore books 0 8.91  # 根据分值区间遍历 zset
zrangebyscore books -inf 8.91 withscores # 根据分值区间 (-∞, 8.91] 遍历 zset，同时返回分值。inf 代表 infinite，无穷大的意思。
zrem books "java concurrency"  # 删除 value
```

#### 容器型数据结构的通用规则

1. create if not exists 容器不存在，那就创建一个
2. drop if no elements 容器里元素没有了，那么立即删除元素，释放内存

#### 过期时间

Redis 所有的数据结构都可以设置过期时间，时间到了，Redis 会自动删除相应的对象。需要注意的是过期是以对象为单位，比如一个 hash 结构的过期是整个 hash 对象的过期，而不是其中的某个子 key。

**如果一个字符串已经设置了过期时间，然后你调用了 set 方法修改了它，它的过期时间会消失**

### 延时队列

Redis 的 list(列表) 数据结构常用来作为异步消息队列使用，使用`rpush/lpush`操作入队列，使用`lpop 和 rpop`来出队列。

#### 队列空了怎么办

可是如果队列空了，客户端就会陷入 pop 的死循环。通常我们使用 sleep 来解决这个问题

#### 队列延迟

睡眠会导致消息的延迟增大。

使用blpop/brpop，前缀字符`b`代表的是`blocking`，也就是阻塞读

#### 闲时连接自动断开

闲置过久，服务器一般会主动断开连接，减少闲置资源占用。这个时候`blpop/brpop`会抛出异常来。

#### 锁冲突的处理

1. 直接抛出异常，通知用户稍后重试；
2. sleep 一会再重试；
3. 将请求转移至延时队列，过一会再试；