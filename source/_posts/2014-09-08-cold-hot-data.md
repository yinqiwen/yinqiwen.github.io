---
layout: post
title: "一种基于Redis/Ardb的混合部署存储方案"
description: ""
category: Ardb
tagline: "Supporting tagline"
tags: [Ardb, Redis]
---

考虑这样一种场景：每天有较多的数据更新，也有较多的数据被访问；从一个时间段总体看来，某份数据只在一定时间段内会被频繁访问，超过这个时间就极少被访问，这样的数据比率占到绝大部分。   
例如网站统计数据就与这种场景类似，只有当天的统计数据会被更新，只有最近7天（或30天）的数据有较大可能会被访问到，极少有人会主动访问历史数据。可以参考下百度统计，CNZZ， GA等系统的数据。
  
针对类似数据存储场景， 我们可以构建一个Redis为Master， Ardb为Slave的存储系统。Ardb需要更改两项默认配置为'yes'：    

    slave-ignore-expire   yes
    slave-ignore-del      yes


前端应用层写入数据只写入到Redis，每更新一次数据，设置expire时间为7天（或30天，自行控制）；该数据会被同步至Ardb， 同时被忽略掉expire设置； 另外由于Redis的expire机制是当expire触发时，redis生成一个'del'命令同步至slave, 所以Ardb slave也需要忽略del命令已保证数据不丢失。 伪码逻辑如下：

    redis.Set(key, value)
    redis.Expire(key, 7*24*3600)  //expired after 7day


<!--more-->	
前端应用层读取数据时，先从Redis读取，若数据不存在，则到Ardb Slave读取，伪码逻辑如下：

    if((data=redis.Get(key)) == null){
         data = ardb.Get(key)
    } 
    process(data)


如此我们可以构建这样一个存储系统，满足下面的优点：

- 更少的内存占用， 内存只存放热数据（hot data）
- 访问延迟基本不变 （热数据从Redis读取， 冷数据从Ardb读取）
- 单套环境支持更大的数据量存储
- 冷热数据不需要复杂的swap机制
- 兼容现有基础设施，不用修改部署现有的redis，只需要新部署Ardb Slave即可； 应用层稍作修改即可

Update:  
在这种场景下，Ardb中有一个配置选项值必须改为no（默认yes）， 否则在Ardb slave与Master之间可能在重新连接时遇到Master中过期的数据在slave中丢失问题：  

	slave-cleardb-before-fullresync no

另外，在这种部署场景中，Ardb slave会忽略所有从Master同步的'del'操作，若应用层需要删除数据，则需要主动在Ardb slave中执行'del', 如：  

    redis.Del(key)  
    ardb.Del(key)       //delete key at redis & ardb both
与此同时， 默认的配置‘slave-read-only’也需要改为‘no’， 否则所有写操作（包括’del‘）都会被拒绝。
   
    slave-read-only no
