---
layout:     post
title:      Redis Big key（大Key)的处理
subtitle:   如何解决 Redis 大key的问题
date:       2026-04-12
author:     BY
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Redis
---

### 什么是Big Key

大 Key 是指单个 Key 的 Value 占用内存过大，或者集合类元素过多的 Key。

| 类型 | 大 Key 判定参考值  |
|-----|-----|
| String | Value > 10KB（大于 5MB 严重）  |
| 复合类型（List、Hash、Set、Sorted Set 等） | 元素数量 > 5000，或总内存 > 10MB |

### Big Key产生的原因

通常由于下面的原因产生的：

- 程序设计不当，比如直接使用String类型存储较大文件对应的二进制数据。
- 对业务的数据规模考虑不周，比如使用集合类型的时候没有考虑到数据量的快速增长。
- 未及时清理垃圾数据，比如哈希中冗余了大量的无用键值对。

### Big Key的危害
主要有三大危害：

- 阻塞Redis的主线程：Redis单线程模型下，对 Big Key的操作（如DEL、HGETALL）会长时间占用主线程，导致所有其他命令排队等待。不然删除一个100MB的key，是否内存可能会需要数百毫秒。
- 网络拥塞：读取Big Key时需要一次性传输大量数据，可能会占满服务器网卡宽带，导致其他客户端的请求超市。
- 内存倾斜：在Redis Cluster中，Big Key集中在某个节点上，导致该节点内存远高于其他节点，可能提前触发淘汰策略或OOM。

此外，Big Key 过期时的被动删除/持久化时 fork 子进程复制 Big Key，都会消耗额外的资源。

### 如何发现Big Key

#### 1.  `redis-cli --bigkeys` (Redis自带的`--bigkeys`参数)
```bin
# redis-cli --bigkeys -i 0.1

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

  0.00% ------------------------------------------------------------
Keys sampled: 0

-------- summary -------

Total key length in bytes is 0 (avg len 0.00)


0 sets with 0 members (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 lists with 0 items (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
0 strings with 0 bytes (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```
关键点：

-  使用Scan命令遍历Redis中所有的Key，会对Redis的性能有一点影响
-  `-i 0.1` 表示扫描过程中每次扫描后休息的时间间隔为 0.1 秒。
-  每次只能找出每种数据类型最大的一个Key，无法列出所有Big Key。

#### 2. 使用Redis自带的Scan命令

```bin
# 查看某个 Key 占用的内存字节数
MEMORY USAGE user:1001
# 返回：5452000（约 5.2MB）

# 批量检查
redis-cli SCAN 0 COUNT 100 | while read key; do
    echo "$key: $(redis-cli MEMORY USAGE $key) bytes"
done
```
关键点：
- 使用 `SCAN` 脚本批量扫描，找出超过阈值的所有Key。
- 结合 `MEMORY USAGE` 精准返回键值对占用的内存空间。

#### 3. RDB文件分析工具

关键点：
- Redis采用了RDB持久化。
- 适合定期巡检，生成Big Key 报告。
- 常用工具： [`redis-rdb-tools`](https://github.com/sripathikrishnan/redis-rdb-tools) [`rdb_bigkeys`](https://github.com/weiyanwei412/rdb_bigkeys)

### Big Key的解决方案

- **拆分**：把一个Big Key拆成多个小Key。例如，将一个含有上万字段的Hash按照一定策略拆分为多个Hash。
- **压缩**：存入前先压缩，减少Value的体积。
- **部分读取**：不要使用 `HGETALL`/ `SMEMBERS` 全量读取，改用 `HSCAN` / `HGET` 分批或按需读取。
- **安全删除**：不要用 `DEL` 删除Big Key，用 `UNLINK`（Redis 4.0+）异步删除，或用 `HSCAN` + `HDEL` 分批删除。
- **懒删除配置**：开启 `lazyfree-lazy-expire yes`，让过期删除在后台线程异步执行。
 


### Big Key预防措施

1.设计阶段

- 避免一个Key存所有数据
- 集合类型提前规划分片策略
- String类型超过10KB就要考虑拆分或压缩

2. 开发阶段

- 封装Redis工具类，自动拆分大Value
- 禁止使用 `HGETALL` / `SMEMBERS` 操作大集合
- 所有 Key 设置合理的 TTL

3. 运维阶段

- 定期巡检：redis-cli --bigkeys
- 监控告警：单个Key内存超过阈值自动告警
- 开启 lazyfree 相关配置


相关问题：
1. `UNLINK` 和 `DEL` 的区别
`DEL`是同步删除，主线程直接释放内存，Big Key 会阻塞。`UNLINK` 是 Redis 4.0 引入的异步删除命令，主线程只做引用技术减1，真正的内存释放是后台 bio 线程异步完成，不会阻塞主线程。生成环境删除Big Key应该用 `UNLINK`.

2. Big Key 对持久化有生命影响
Redis做RDB持久化时需要 fork 子进程。如果Redis中存在Big Key，内存占用高， fork的耗时也会增加。同时 AOF 重写时也需要处理 Big Key，可能导致 AOF 重写耗时过长。另外，Big Key过期删除时的删除操作也会影响 AOF 文件体积。

### 总结

Redis的 Big Key问题是指单个 Key的 Value过大或集合元素过多，会导致 **阻塞主线程**、**网络拥塞**、**内存倾斜**等严重问题。
发现 Big Key可以使用 `redis-cli --bigkeys`、`MEMORY USAGE`，RDB文件离线分析等手段。
解决方案的核心是 **拆分 Big Key为多个小 Key**，配合压缩、部分读取、`UNLINK` 异步删除、开启 lazyfree配置等手段。

### 参考
 
- https://javaguide.cn/database/redis/redis-questions-02.html 
- https://www.quanxiaoha.com/java-interview/redis-big-key-problem-solution