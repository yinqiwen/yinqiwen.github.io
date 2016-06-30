---
layout: post
title: "Ardb与Redis中的复制机制简介与比较"
description: ""
category: Ardb
tagline: "Supporting tagline"
tags: [Ardb, Redis, Replication]
---

近段时间花了几天重新review了下Ardb中的主从复制的设计与实现，经过多次高压力环境下的测试，已经相当稳定了。这里简单介绍下Redis与Ardb中的主从复制实现， 以及两者简单比较。   

##Redis

###2.6之前
在Redis2.6及以前，主从复制机制比较简单， 基本可以描述为全量镜像复制 + 异步持续复制。大致的过程如下：  
 
- Slave发送Sync命令给Master   
- Master收到Sync命令后，开始全量dump内存到rdb文件中。于此同时， master接收到的后续写入命令都缓存到Slave的连接待发送缓冲区里；
- Master在dump期间，定时给Slave发送一个换行字符以保持TCP连接
- dump过程结束后，Master将此文件发送给Slave。 文件发送完毕后，继续发送待发送缓冲区replication数据；  后续收到的写操作直接发送给Slave。
- Slave将接收到的dump内容保存到rdb文件中，接受完毕后开始load到内存中；此过程是阻塞操作，期间不能处理任何其他命令。
- Slave加载完rdb文件后，即可认为Master/Slave已经是完成同步状态， Slave可以开始提供对外服务
- Master每间隔repl-ping-slave-period秒（默认10s）给所有完成同步的Slave发送ping； 若slave在repl-timeout秒（默认60s）内未收到复制命令（包括ping），则自动断开和master之间连接。
- 若Slave重新连接上Master， 则之上的步骤重新执行一次

Redis2.6及以前的主从复制机制相对比较简单， 很好的保证了数据一致性，复制效率的要求。不过在一些比较极端的场景下，这种复制机制有些缺陷。 考虑下面的场景： 

- Master存储了10G的数据，Slave与Master之间网络不稳定， 频繁断开；
- Master存储了100G的数据，Slave完成同步后，Master宕机。 Slave切换为Master， 而原先Master恢复后，将原Master改为新Master的Slave。（用redis sentinel可以自动模拟这种切换场景）

这两种场景都要求Master， Slave之间在每次连接时重新开始全量镜像复制。
                 
<!--more-->
				 
###2.8之后
在Redis2.8之后， 针对上面描述的第一个缺陷，增加了增量复制的能力，相对复杂了些。大致实现过程如下：

- Master在内存里维护一个repl-backlog-size大小（默认1M）的环形缓冲区，所有需要复制到slave的写入命令都会写入到此缓冲区里。Master同时记录缓冲区的长度， 当前复制偏移量（持续增加的int64），当前在缓冲区偏移量（缓冲区下标），以及一些其它值。
- Slave发送PSync `<runid> <replication offset>`到Master
- 若runid与Master不匹配，或者replication offset与Master当前的replication offset相差过大，Master会回复一个 +FULLRESYNC <runid> <replication offset>给Slave， 开始全量镜像复制。
- 全量镜像复制的过程以及之后的复制过程与上述2.6的复制中描述的一样。Slave在全量复制后， 需要将FULLRESYNC携带的runid， replication offset保存起来，后续每次接收到复制命令后， 更新replication offset
- 若master判断runid与Master匹配， replication offset也是一个正常范围内的取值（与当前replication offset相差小于repl-backlog-size大小）， 则Master给Slave回复一个+Continue， 开始增量更新
- 增量更新开始后，Master将Slave的replication offset到自己当前replication offset的内容发送给Slave；repl-backlog数据发送完毕后，后续则是将需要复制的命令逐一发送给Redis
- 若Slave与Master断开连接并重连， 由于Slave中保存有master的runid以及最新的replication offset， 在断开的时间足够短的情况下，Master会判定是增量更新，只需要发送很少的数据给Slave。

Redis2.8的增量更新基本解决了短时间断开重连后的全量复制的问题。但遇到稍长时间的断开（1小时左右）， 大部分情况会遇到需要全量复制的问题。

##Ardb
Ardb的主从复制在遵循Redis2.8的设计的基础上，增加了状态，复制日志持久化，复制内容校验功能。

- Mater/Slave通过内存映射文件（mmap）创建一个较大的缓冲区。缓冲区分为两部分， 头部固定长度， 用于保存各种replication状态， 包括replication offset/checksum等， 剩下部分是一个repl-backlog-size大小（默认1G）的环形缓冲区。
- 缓冲区更改的内容每5s flush到硬盘上
- Slave发送APSYNC `<runid> <replication offset> <checksum>`到Master
- 若runid与Master不匹配，或者replication offset与Master当前的replication offset相差过大， 或者checksum与需要复制的剩余内容计算后和当前的replication checksum不一致，Master会回复一个 +FULLRESYNC <runid> <replication offset> <checksum>给Slave， 开始全量镜像复制。
- 全量镜像复制的过程以及之后的复制过程与上述2.6的复制中描述的一样。Slave在全量复制后， 需要将FULLRESYNC携带的runid， replication offset，checksum保存起来，后续每次接收到复制命令后， 更新replication offset以及checksum。
- 若master判断可以增量更新，master给Slave回复一个+Continue， 开始增量更新（后续与Redis2.8一致）
- 在全量复制结束后， Ardb中相对Redis增加了一个Replication log的checksum计算过程，代表整个复制内容的checksum， 根据replication backlog内容更改而持续更新
- 若Slave与Master断开连接并重连， 由于master中的replication backlog比较大， 可以支撑更长时间的slave断开情况， 对于恢复时间较长的场景更有用。
- 若Master宕机， Slave切换到到Mater； 然后经过一段时间，Master恢复， 并被设置为新master的slave。 由于Master/Slave的各种replication状态是持久化的，新的master会认为这是一个断开重连的slave，并判断是否增量更新。

Ardb的增量更新机制基本解决了Master/Slave发生主从切换情况下的增量同步复制的问题。


##Redis Vs Ardb

###Redis的优势

- 代码相对更简单易于理解
- 用于主从复制的内存占用较少

###Ardb的优势

- 支持更长时间的Slave宕机情况(由repl-backlog-size大小以及写频率决定)
- Master/Slave发生主从切换，条件具备情况下，可以避免全量复制的发生
- 兼容Redis， 支持Redis作为主/从


## 目前设计实现的问题

- Slave复制只有一个线程执行，且与对外查询端口共享；而Master可能开启多个线程执行写入，因此理论上存在Slave复制无法赶上Master写入的问题（增加repl-backlog-size只能缓解，无法根本解决）；后续实现上可以修改解决
- 无法做到可选db粒度的复制（如mysql中slave可以配置只复制某一个database）；设计的选择，暂无法实现
