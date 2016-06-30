---
layout: post
title: "Ardb简介"
description: ""
category: Ardb
tags: [Ardb, Redis]           
---  


## 简介
Ardb是一个C/C++开发的完全兼容Redis协议的NoSQL存储服务，支持绝大部分Redis命令以及数据结构如list/hash/set/sorted set，你可以很容易将依赖redis的业务在Redis和Ardb之间来回迁移。 
 
Ardb的存储基于现有成熟的KV存储引擎实现，理论上任何类似B-Tree/LSM Tree实现的KV存储实现均可作为Ardb的底层存储实现，目前Ardb支持LevelDB/RocksDB/LMDB.  

Ardb项目的目的不是为了替换Redis（这点很重要）， 而是作为Redis的补充存在。Ardb重点解决的是在性能可接受的情况下单机支持远超过内存限制的数据量， 以及与Redis之间的互通的问题。

## 特性
最新的Ardb版本支持如下特性：   

- 完全支持Redis协议， 包括早期的协议（可用telnet连接Ardb实例）
- 支持绝大部分redis命令，所有数据结构Set/Hash/List/Sorted Set， Lua脚本支持， PubSub，Transaction等等
- 支持所有数据结构的Key的Expire过期自动删除机制
- 兼容Redis主从复制机制。你可以把Ardb作为Redis实例的Slave或者将Redis作为Ardb的Slave，支持Redis2.6以及2.8+。
- 兼容Redis Sentinel的主从切换机制，你可以用Redis Sentinel监控主从节点并在节点异常情况下切换。
- 支持Redis的dump备份数据文件通过import命令在线导入；Save/BGSave命令将Ardb数据用Redis Dump格式导出
- 支持类似Redis的分库机制， 库ID范围为0到0xFFFFFF,无需预先指定
- **内置二维空间索引实现，基于扩展的GeoAdd/GeoSearch命令实现二维空间查找特性。**
- 支持LevelDB/RocksDB/LMDB作为底层的存储引擎
- 兼容所有的Redis的Client实现，你可以用任意Redis的client实现连接Ardb实例
- **多端口服务支持，读写端口分离， QPS限制， IP白名单等高级特性**
- 支持作为一个库内置到其他服务中

<!--more-->
## Ardb支持的Redis命令情况
以Redis2.8.9为比较标准，未列出的均是已经支持且与Redis语义相同的命令： 
  
### 不支持
  - OBJECT
  - DEBUG


### 支持，但语义有不同之处
  - dbsize， 返回磁盘占用空间（字节数）
  - setbit/getbit/bitcount/bitop, Ardb中用bitset结构保存而非string，可以支持非常大的offset
  - info， 和redis的输出大有不同


### 新增
  - GeoAdd/GeoSearch， 空间索引
  - Cache， High Level的cache
  - Import， 在线导入备份数据，支持Redis Dump以及Ardb Dump
  - Compact/CompactAll， 后台comapct数据， 对LevelDB/RocksDB有效


## Ardb VS Redis

### 劣势
- 性能，Redis大部分读写操作是O(1),且完全在内存上执行；KV存储引擎均是类似Tree结构（B-Tree， LSM-Tree等），读一般是Log（n），写性能LSM-Tree可能视作O(1)。
- 集群，Redis3.0已经支持集群，Ardb目前不支持
- 原子性（Atomic）， 由于Redis是单线程的服务，天生所有操作均具有原子性
- 稳定性
- 功能

### 优势
- 单机存储数据量以及同等数据量下的存储成本
- 读写多线程，可以充分利用多核机器的CPU
- 支持二维空间索引
- 读写端口分离， QPS限制， IP白名单等高级特性

### 复杂度
Resis的存储结构基本是hash， 所以大部分读写是O(1); Ardb底层的存储引擎都是某种Tree的实现，若数据未在缓存中，则读复杂度基本和Tree的查询复杂度是一个数量级；写操作在LSM-Tree可能视作O(1)。另外需要差别较大需要注意的是， sorted set的类似rank操作在Ardb中都是类似O（n）的复杂度。

## Benchmark
- 可以用Redis自带的benchmark测试Ardb服务.
- 现有的一些测试数据可以在github项目首页找到. [Ardb](https://github.com/yinqiwen/ardb)


## 编译部署

执行如下命令下载编译最新源码   

	git clone --depth=1 https://github.com/yinqiwen/ardb.git
    cd ardb
    make

若选择其他存储引擎实现， 如rocksdb，则需要在make前制定环境变量storage_engine， 如：
	
	storage_engine=rocksdb make

编译完成后，在src目录下生成ardb-server, ardb-test两个可执行文件，前者为服务， 后者为单元测试程序。配置文件模板为ardb.conf, 在ardb根目录下。按如下方式启动：

	cd src
    ./ardb-server ../ardb.conf


