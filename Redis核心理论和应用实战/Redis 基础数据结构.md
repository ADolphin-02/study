

# 目录

[TOC]



# Redis 基础数据结构

## string (字符串)

使用 JSON 序列化成字符串，然后将序列化后的字符串塞进 Redis 来缓存。同样，取用户信息会经过一次反序列化的过程

![image-20200814134706090](Redis 基础数据结构.assets/image-20200814134706090.png)

Redis 的字符串是**动态字符串**，是可以修改的字符串，内部结构实现上类似于 Java 的 ArrayList，采用**预分配冗余空间**的方式来减少内存的频繁分配，如图中所示，内部为当前字符串实际分配的空间 capacity 一般要高于实际字符串长度 len。当字符串长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间。需要注意的是字符串最大长度为 512M。

计数  如果 value 值是一个整数，还可以对它进行自增操作。自增是有范围的，它的范围是 signed long 的最大最小值，超过了这个值，Redis 会报错。

## list (列表)  

Redis 的列表相当于 Java 语言里面的 LinkedList，注意它是链表而不是数组。这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)，这点让人非常意外。

当列表弹出了最后一个元素之后，该数据结构自动被删除，内存被回收。

Redis 的列表结构常用来做异步队列使用。将需要延后处理的任务结构体序列化成字符串塞进 Redis 的列表，另一个线程从这个列表中轮询数据进行处理。

### 慢操作

![image-20200814141314174](Redis 基础数据结构.assets/image-20200814141314174.png)

### 快速列表

![image-20200814141359527](Redis 基础数据结构.assets/image-20200814141359527.png)

如果再深入一点，你会发现 Redis 底层存储的还不是一个简单的 linkedlist，而是称之为快速链表 quicklist 的一个结构。  

首先在列表元素较少的情况下会使用一块连续的内存存储，这个结构是 ziplist，也即是压缩列表。它将所有的元素紧挨着一起存储，分配的是一块连续的内存。当数据量比较多的时候才会改成 quicklist。因为普通的链表需要的附加指针空间太大，会比较浪费空间，而且会加重内存的碎片化。比如这个列表里存的只是 int 类型的数据，结构上还需要两个额外的指针 prev 和 next 。所以 Redis 将链表和 ziplist 结合起来组成了 quicklist。也就是将多个 ziplist 使用双向指针串起来使用。这样既满足了快速的插入删除性能，又不会出现太大的空间冗余。 

## hash (字典)

Redis 的字典相当于 Java 语言里面的 HashMap，它是无序字典。

![image-20200817103235180](Redis 基础数据结构.assets/image-20200817103235180.png)

 **rehash 的方式不一样**：Java 的 HashMap 在字典很大时，rehash 是个耗时的操作，需要一次性全部 rehash。Redis 
为了高性能，不能堵塞服务，所以采用渐进式 rehash 策略。

![image-20200817103348034](Redis 基础数据结构.assets/image-20200817103348034.png)

渐进式 rehash 会在 rehash 的同时，**保留新旧两个 hash 结构**，查询时会同时查询两个 hash 结构，然后在后续的**定时任务中以及 hash 的子指令**中，循序渐进地将旧 hash 的内容一点点迁移到新的 hash 结构中。 

当 hash 移除了最后一个元素之后，该数据结构自动被删除，内存被回收。 

hash 结构也可以用来存储用户信息，不同于字符串**一次性**需要全部**序列化整个对象**，hash 可以对用户结构中的每个字段单独存储。这样当我们需要获取用户信息时可以进行部分获取。而以整个字符串的形式去保存用户信息的话就只能一次性全部读取，这样就会比较浪费网络流量。  

hash 也有缺点，hash 结构的**存储消耗**要**高**于单个字符串，到底该使用 hash 还是字符串，需要根据实际情况再三权衡。 

hash 结构中的单个子 key 也可以进行**计数**，它对应的指令是 **hincrby**，和 incr 使用基本一样

## set (集合)

相当于 Java 语言里面的 HashSet，它内部的键值对是**无序的唯一**的。它的内部实现相当于一个特殊的字典，字典中所有的 value 都是一个值 NULL。

## zset (有序列表)

类似于 Java 的 SortedSet 和 HashMap 的结合体，，一方面它是一个 **set**，保证了内部 value 的唯一性，另一方面它可以给每个 value 赋予一个 **score**，代表这个 value 的排序权重。它的内部实现用的是一种叫着**「跳跃列表」**的数据结构。

### 跳跃列表

 zset 要支持随机的插入和删除

我们先看一个普通的链表结构。 

![image-20200817104756643](Redis 基础数据结构.assets/image-20200817104756643.png)

![image-20200817105017841](Redis 基础数据结构.assets/image-20200817105017841.png)

![image-20200817105250378](Redis 基础数据结构.assets/image-20200817105250378.png)

# 应用 1：分布式锁 

## 分布式锁 

 > set lock:codehole true ex 5 nx OK 
 >
 > ... 
 >
 > - do something critical
 >
 >  ... 
 >
 > del lock:codehole 

上面这个指令就是 setnx 和 expire **组合在一起的原子指令**，它就是分布式锁的奥义所在。

## 	超时问题 

Redis 的分布式锁不能解决超时问题，如果在加锁和释放锁之间的**逻辑执行的太长**，以至于超出了锁的超时限制，就会出现问题。

 set 指令的 value 参数设置为一个**随机数**，释放锁时先**匹配随机数**是否一致，然后**再删除** key。但是匹配 value 和删除 key 不是一个原子操作，Redis 也
没有提供类似于 delifequals 这样的指令，这就需要使用 **Lua 脚本**来处理了，因为 Lua 脚本可以保证连续多个指令的原子性执行。

## 可重入

需要对客户端的 set 方法进行包装，使用线程的 Threadlocal 变量存储当前持有锁的计数

以上还不是可重入锁的全部，精确一点还需要考虑内存锁计数的过期时间，代码复杂度将会继续升高。老钱不推荐使用可重入锁，它加重了客户端的复杂性，在编写业务方法时注意在逻辑结构上进行调整完全可以不使用可重入锁。

**基于 ThreadLocal 和引用计数**

```java
public class RedisWithReentrantLock {
            private ThreadLocal<Map> lockers = new ThreadLocal<>();
            private Jedis jedis;

            public RedisWithReentrantLock(Jedis jedis) {
                this.jedis = jedis;
            }

            private boolean _lock(String key) {
                return jedis.set(key, "", "nx", "ex", 5L) != null;
            }

            private void _unlock(String key) {
                jedis.del(key);
            }

            private Map<String, Integer> currentLockers() {
                Map<String, Integer> refs = lockers.get();
                if (refs != null) {
                    return refs;
                }
                lockers.set(new HashMap<>());
                return lockers.get();
            }

            public boolean lock(String key) {
                Map refs = currentLockers();
                Integer refCnt = refs.get(key);
                if (refCnt != null) {
                    refs.put(key, refCnt + 1);

                    return true;
                }

                boolean ok = this._lock(key);
                if (!ok) {
                    return false;
                }
                refs.put(key, 1);
                return true;
            }

            public boolean unlock(String key) {
                Map refs = currentLockers();
                Integer refCnt = refs.get(key);
                if (refCnt == null) {
                    return false;
                }
                refCnt -= 1;
                if (refCnt > 0) {
                    refs.put(key, refCnt);
                } else {
                    refs.remove(key);
                    this._unlock(key);
                }
                return true;
            }

            public static void main(String[] args) {
                Jedis jedis = new Jedis();
                RedisWithReentrantLock redis = new RedisWithReentrantLock(jedis);
                System.out.println(redis.lock("codehole"));
                System.out.println(redis.lock("codehole"));
                System.out.println(redis.unlock("codehole"));
                System.out.println(redis.unlock("codehole"));
            }
        }
```

# 应用 2：延时队列 （锁冲突处理

## 队列空了怎么办？

![image-20200817110929773](Redis 基础数据结构.assets/image-20200817110929773.png)

- 符 b 代表的是 blocking

阻塞读在队列没有数据的时候，会立即进入休眠状态，一旦数据到来，则立刻醒过来。消息的延迟几乎为零。用 `blpop/brpop` 替代前面的 `lpop/rpop`，就完美解决了上面的问题。

## 空闲连接自动断开 

如果线程一直阻塞在哪里，Redis 的客户端连接就成了闲置连接，闲置过久，服务器一般会主动断开连接，减少闲置资源占用。这个时候 blpop/brpop 会抛出异常来。 

**所以编写客户端消费者的时候要小心，注意捕获异常，还要重试。...** 

## 锁冲突处理

1、直接抛出异常，通知用户稍后重试； 

2、sleep 一会再重试； 

3、将请求转移至延时队列，过一会再试； 

### 延时队列的实现

1. 延时队列可以通过 Redis 的 zset(有序列表) 来实现。我们将**消息序列化**成一个字符串作为 zset 的 **value**，这个消息的到期处理时间作为 **score**，然后用多个线程**轮询 zset 获取到期的任务进行处理**，多个线程是为了保障可用性，万一挂了一个线程还有其它线程可以继续处理。因为有多个线程，所以需要考虑并发争抢任务，确保任务不能被多次执行。 
2. **zrem 方法**（移除有序集 `key` 中的一个或多个成员，不存在的成员将被忽略。
3. Redis 的 **zrem 方法**是多线程多进程争抢任务的关键，它的返回值决定了当前实例有没有抢到任务，因为 loop 方法可能会被多个线程、多个进程调用，同一个任务可能会被多个进程线程抢到，通过 zrem 来决定**唯一的属主**。  同时，我们要注意一定要对 handle_msg 进行异常捕获，避免因为个别任务处理问题导致循环异常退出。以下是 Java 版本的延时队列实现，因为要使用到 Json 序列化，所以还需要 fastjson 库的支持。 

**源文件查看代码**

> 总结：(先把锁序列化成zset，加个延迟时间，多线程去拿`zrangeByScore(queueKey, 0, System.currentTimeMillis(), 0, 1);` 时间为0的时候的，然后被删除`zrem`,然后是实现延迟 了。）**多个进程取到之后再使用 zrem 进行争抢**

**多个进程取到之后再使用 zrem 进行争抢**那些没抢到的进程都是白取了一次任务，这是浪费。可以考虑使用 lua scripting 来优化一下这个逻辑，将
zrangebyscore 和 zrem 一同挪到服务器端进行原子化操作，这样多个进程之间争抢任务时就不会出现这种浪费了

# 应用 3：位图（签到）

## 基本使用

位图数据结构，这样每天的签到记录只占据一个位，365 天就是 365 个位，46 个字节 (一个稍长一点的字符串) 就可以完全容纳下，这就大大
节约了存储空间。 

位图不是特殊的数据结构，它的内容其实就是普通的字符串，也就是 byte 数组。我们可以使用**普通**的 get/set 直接获取和设置整个位图的内容，也可以使用位图操作 `getbit/setbit` 等将 byte 数组看成**「位数组」**来处理。 

![image-20200817140057520](Redis 基础数据结构.assets/image-20200817140057520.png)

![image-20200817140135230](Redis 基础数据结构.assets/image-20200817140135230.png)![image-20200817140243542](Redis 基础数据结构.assets/image-20200817140243542.png)

## 统计和查找

**统计**指令 `bitcount` 和位图**查找**指令 `bitpos`，bitcount 用来统计指定位置范围内 **1** 的个数，bitpos 用来查找指定范围内出现的第一个 **0 或 1**。  

范围参数[start, end]，start 和 end 参数是字节索引，也就是说指定的位范围必须是 8 的倍数，

![image-20200817140947722](Redis 基础数据结构.assets/image-20200817140947722.png)

## 魔术指令 bitfield 

 bitfield 有三个子指令，分别是 `get/set/incrby`，它们都可以对指定位片段进行读写，但是最多**只能处理 64 个连续的位**，如果
超过 64 位，就得使用多个子指令，bitfield 可以一次执行多个子指令。 

![image-20200817141332852](Redis 基础数据结构.assets/image-20200817141332852.png)

![image-20200817141428041](Redis 基础数据结构.assets/image-20200817141428041.png)

![image-20200817141603799](Redis 基础数据结构.assets/image-20200817141603799.png)

![image-20200817141639265](Redis 基础数据结构.assets/image-20200817141639265.png)

**溢出策略子指令 overflow**：bitfield 指令提供了溢出策略子指令 overflow，用户可以选择溢出行为，默认是折返 (wrap)，还可以选择失败 (fail) 报错不执行，以及饱和截断 (sat)，超过了范围就停留在最大最小值。overflow 指令只影响接下来的第一条指令，这条指令执行完后溢出策略会变成默认值折返 (wrap)。 

![image-20200817141713441](Redis 基础数据结构.assets/image-20200817141713441.png)

# 应用 4： HyperLogLog

- UV的方案一：当一个请求过来时，我们使用 sadd 将用户 ID 塞进去就可以了**（面几千万的 UV，你需要一个很大**
  **的 set 集合来统计，这就非常浪费空间。）**
- UV的方案二：

HyperLogLog 提供不精确的去重计数方案，PV但是UV 不一样，它要**去重**。（**标准误差是 0.81%，**）

## 使用方法

HyperLogLog 提供了两个指令 `pfadd 和 pfcount`

![image-20200817143149023](Redis 基础数据结构.assets/image-20200817143149023.png)

## pfmerge 适合什么场合用？ 

![image-20200817143506634](Redis 基础数据结构.assets/image-20200817143506634.png)



## 注意事项

## 原理（不懂）

# 应用 5： 布隆过滤器 

场景：推送不重复的抖音视频。

# 应用 6： 简单限流 （自己学）

优点：能够精确的统计次数。

缺点：就是zset的数据结构会越来越大

求打造成一个zset数组，当每一次请求进来的时候，value保持唯一（唯一用户），而score可以用当前时间戳表示，因为score我们可以用来计算当前时间戳之内有多少的请求数量。

zremrangeByScore：删除时间窗口以外的请求

每一个行为到来时，都维护一次时间窗口。将时间窗口外的记录全部清理掉，只保留窗口内的记录。zset 集合中只有 score 值非常重要，value 值没有特别的意义，只需要保证它是唯一的就可以了。 

```java
public class SimpleRateLimiter {
    private Jedis jedis;

    public SimpleRateLimiter(Jedis jedis) {
        this.jedis = jedis;
    }

    public boolean isActionAllowed(String userId, String actionKey, int period, int maxCount) {
        String key = String.format("hist:%s:%s", userId, actionKey);
        long nowTs = System.currentTimeMillis();
        Pipeline pipe = jedis.pipelined();
        pipe.multi();
        // value 没有实际的意义，保证唯一就可以
        pipe.zadd(key, nowTs, "" + nowTs);
        pipe.zremrangeByScore(key, 0, nowTs - period * 1000);
        Response<Long> count = pipe.zcard(key);
        pipe.expire(key, period + 1);
        pipe.exec();
        pipe.close();
        return count.get() <= maxCount;
    }

    public static void main(String[] args) {
        Jedis jedis = new Jedis();
        SimpleRateLimiter limiter = new SimpleRateLimiter(jedis);
        for (int i = 0; i < 20; i++) { //相当于请求
            System.out.println(limiter.isActionAllowed("laoqian", "reply", 60, 5));
        }
    }
}
```

# 应用 7： 漏斗限流 

## 前面还有，不会

## Redis-Cell

Redis 4.0 提供了一个限流 Redis 模块，它叫 redis-cell。该模块也使用了漏斗算法，并提供了`原子的限流指令`。有了这个模块，限流问题就非常简单了。 

该模块只有 1 条指令 `cl.throttle`，参数和返回值

![image-20200818145931369](Redis 基础数据结构.assets/image-20200818145931369.png)

上面这个指令的意思是允许「用户老钱回复行为」的频率为每 60s 最多 30 次(漏水速率)，漏斗的初始容量为 15，也就是说一开始可以连续回复 15 个帖子，然后才开始受漏水速率的影响。我们看到这个指令中漏水速率变成了 2 个参数，替代了之前的单个浮点数。

![image-20200818150935161](Redis 基础数据结构.assets/image-20200818150935161.png)

























