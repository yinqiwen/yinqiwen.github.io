---
layout: post
title: "Ardb-RocksDB的一些生产环境实践总结"
description: ""
category: Ardb
tagline: "Supporting tagline"
tags: [Ardb, RocksDB]
---

## 部署运营情况
Ardb（RocksDB）目前在生产环境中是作为一个在线推荐/查询系统的存储服务部署，持续运行了约半年，相关的一些运营情况

- 1T SSD机器，主从互备
- 共存储约450GB的原始数据（integer变长存储，snappy压缩并compact后120GB），每天约更新150GB的数据（存放到redis同样结构，大约700G内存占用）
- 最大写入qps为20000（Ardb配置中限制）， 大多为hmset，sadd，geoadd，zrem，del等写/删除命令；写入数据由后台例行任务调用redis命令
- 峰值查询qps在2000以上， 平均则近600， 查询命令多为hgetall， smembers， geosearch
- 查询平均延迟不到1ms，准确说0.3ms左右

<!--more-->

## 实践总结以及Ardb的改进

- RocksDB相比LevelDB在读写延迟方面改进不少，当半年前使用LevelDB作为底层存储， 当数据量上升到150G左右时，遇到严重延迟问题，查询会偶发性的阻塞长约几十秒；替换后，阻塞次数与时间都有下降， 但仍有10s内的查询延迟情况
- latency过长的主要原因在于rocksdb/leveldb compact时大量系统IO占用会阻塞某些涉及磁盘IO的操作
- rocksdb支持后台多线程执行compact， 一般设置两个优化选项即可，可以大大提升compact的效率
        
        m_options.OptimizeLevelStyleCompaction();
        m_options.IncreaseParallelism();
- rocksdb 3.4后可以设置compact时的最大磁盘IO量（字节/s）， 可以进一步降低compact对外部操作影响，缺点是会延长comapct时间。
- 写入命令仍可能有几十ms的延迟；需要将读写线程分离降低对读延迟的影响
- 多端口绑定以及端口IO线程分离；Ardb支持服务启动时绑定多个端口，每个端口绑定固定数目IO线程， 这样当端口A大量命令涌入时，理论上不会影响阻塞端口B的命令处理
- 定期compact合并；大量重复修改覆盖某个key，会导致该key的多份数据跨越多个level，影响查询效率；尤其是对迭代器的迭代效率影响极大。Ardb中可以设置写入50000000次后触发主动compact操作，以及在有写操作情况下至少12小时触发一次compact操作
- 执行上述改进后，80%的查询延迟在10ms内， 99.1%的查询延迟控制在100ms以内
- rocksdb/leveldb默认内置的Cache表现较差；基于block的cache，对于随机的查询key， 理论上block cache会频繁换出；对于全量数据每天例行更新的场景，cache的命中率会更差。
- 近段时间加上一种high level的基于key级别的LRU cache，可以兼顾例行更新的情况， 小数据量实验中cache命中率最高可到28%，平均查询延迟降低到了2ms
- list/hash/set/sorted set等redis数据结构若成员数目不多的情况下，可以参考redis的实现，将所有成员压缩合并为一个key-value对存储， 对于hgetall/smembers等获取全部成员的操作效率提升极为明显；否则整个操作会涉及更多IO操作。
- 数据成员为integer格式时， 按[zigzag](https://developers.google.com/protocol-buffers/docs/encoding)变长方式存储， 可以大大降低存储占用磁盘空间；450GB的原始数据中，integer变长存储，snappy压缩并compact后磁盘占用120GB。
- 针对合并压缩存储的数据结构，增加了HReplace/SReplace命令，更新覆盖该key时，可以减少一次del操作
- rocksdb在没有外部命令写入触发compact时，作为一个只读查询DB，表现优秀，1000 qps的查询压力下平均查询延迟小于1ms
- 运行内存占用比预期要大，block_cache_size 设置为10G时，无内存泄漏情况下，最后占用内存22G； leveldb占用内存更大，这个可能每个sst文件大小有关， rocksdb大约64MB一个(rocksdb支持配置修改)， leveldb则默认2MB一个。按[basho leveldb](https://github.com/basho/leveldb)的计算公式：

        memory_usage = write_buffer_size*2  + max_open_files*4194304 + (size in NewLRUCache initialization)

- rocksdb默认的max_manifest_file_size没有限制大小，当大量写入命令执行中重启服务，会导致启动时阻塞在recover极长时间， 我们遇到过的一次停顿了大约5个小时；可以设置max_manifest_file_size为2G降低启动时的recover时间



