---
layout: post
title: "利用Ardb/Redis构建基于位置实时服务"
description: ""
category: Ardb
tagline: "Supporting tagline"
tags: [Ardb, GeoHash]
---

Ardb早在半年前的0.7.0版本就已经具备了二维空间数据的存储/查询的能力；基于此能力，可以构建基于位置实时服务， 比如实时查找附近的地理位置，附近的人等LBS类型服务。以下介绍Ardb中空间索引的实现原理以及如何使用。

## 原理
Ardb中的二维空间索引实现从原理上说，可以简化为GeoHash + Sorted Set。 在比较早的时候写过一篇英文原理介绍在[这里](https://github.com/yinqiwen/ardb/wiki/Spatial-Index), 这里用中文再介绍一次。 

所谓二维空间索引， 就是对二维空间数据建立索引， 通常key-value形式的存储只能存储一维的数据， 针对二维数据， 就需要做降维处理。降维的方法有很多种，Ardb中采用的是成熟的[GeoHash](http://en.wikipedia.org/wiki/Geohash)。 GeoHash是一种对经纬度进行编码的实现，利用GeoHash可以将二维的坐标编码成一维的数据， 如字符串。这样，我们就可以将二维的坐标作为索引的key保存了。  

大部分的GeoHash实现都是将二维的坐标编码编码为Base32的字符串， 这种结果不适用于Sorted Set（只支持double类型的score），另外也存在精度问题。 由于Base32字符串中一个字符代表5个bit，和GeoHash编码步骤每次产生2bit不匹配，当Base32字符串长度为奇数时，此GeoHash值代表一个长宽不相等的矩形，这个对于计算半径精度误差较大。 此外，由于一个字符代表5个bit，意味着增加/减少位置精度需要每5bit增减，这对于范围查询时，存在较大的浪费。

我另外参考GeoHash原理写了另外一个[GeoHash-Int库](https://github.com/yinqiwen/geohash-int), 只生成64bit integer结果，每增加/减少位置精度只需要2bit增减。

关于GeoHash于GeoHash-Int的详细介绍，可以参考这篇文章[GeoHash介绍与改进的GeoHash-Int](http://yinqiwen.github.io/%E6%8A%80%E6%9C%AF/2014/08/24/geohash/).

关于Redis的Sorted Set， 可以暂时理解为一个排序的集合。 由于GeoHash有一个特点，相近的坐标的GeoHash值有很大几率非常接近，从属于同一个GeoHash编码区域，即使在边缘情况不相邻下，也可以通过通过查找邻居的方法找到所在GeoHash编码区域。 我们可以将GeoHash值作为score，坐标点的名称作为value存储到一个sorted set中。 这样，在查找相近坐标时，可以利用Sorted Set的`ZRANGEBYSCORE`方法找到相近的指定范围的坐标集合。

以下介绍在Ardb中的具体实现与相关协议命令。

<!--more-->
## 存储
Ardb扩展了一个Redis命令，用于存储二维坐标数据.  格式如下：

	GeoAdd key longitude latitude value [longitude latitude value ...]

GeoAddde的内部实现大致可以用两步描述：   

- 用[GeoHash-Int库](https://github.com/yinqiwen/geohash-int)， 按60bit编码将二维坐标编码为一个64bit integer，大约代表坐标系中一个方圆0.2m的矩形。
- 调用Redis命令 `ZAdd key score value` 将数据存储


## 查询
Ardb扩展了一个Redis命令，用于二维空间数据查询.  格式如下：

	   GEOSEARCH key MERCATOR|WGS84 x y  <GeoOptions>
       GEOSEARCH key MEMBER m            <GeoOptions>
     
       <GeoOptions> = RADIUS r
                      [ASC|DESC] [WITHCOORDINATES] [WITHDISTANCES]
                      [GET pattern [GET pattern ...]]
                      [INCLUDE key_pattern value_pattern [INCLUDE key_pattern value_pattern ...]]
                      [EXCLUDE key_pattern value_pattern [EXCLUDE key_pattern value_pattern ...]]
                      [LIMIT offset count]

查找时， 需要指定某坐标或者已存储的成员作为查找起始位置，同时指定查找半径范围。另外可选的参数支持输出坐标，输出距离，按距离排序， 外键关联，条件过滤，分页输出等。  
 
GeoSearch的内部实现可以大致描述为：  

- 查找起始位置坐标， 若指定起始位置是已存储的成员，需要找到对应score，反GeoHash得到坐标，由于存储的GeoHash值为60bit编码， 反GeoHash得到的坐标与原始坐标误差在0.2m以内，是可以近似的认为是一致的。
- 按指定半径估计GeoHash编码长度。从地球半径6372797.560856m为初始精度范围开始估计，GeoHash编码每增加2bit，意味着精度增加一倍（上一精度范围除2）， 若精度范围刚好为最小的大于指定半径的值时，则得到编码长度；
- 按GeoHash编码长度计算起始位置的GeoHash值；
- 计算相邻8个区域（东，西，南，北，东北，东南，西南，西北）的GeoHash值；
- 以上得到的9个GeoHash值，针对每个GeoHash值，生成一个 `[GeoHashIneger, GeoHashIneger + 1]`值对
- 每对值中每个GeoHash值按60bit对齐（左移补齐到60bit， 右侧0填充）， 得到`[GeoHashIneger_60bit, (GeoHashIneger+1)_60bit]`；
- 针对每对值`ZRANGEBYSCORE key GeoHashIneger_60bit (GeoHashIneger+1)_60bit WITHSCORES`范围查找
- 将所有结果按距离计算公式再过滤一次,过滤掉范围之外的结果， 需要用到反GeoHash计算原始坐标

可见，查询实现相比存储实现复杂很多，实际实现里还有很多优化实现（如合并查找区域，大范围查找区域分拆等），额外的功能实现等，这里只列出基本实现步骤，不再赘述。感兴趣的朋友可以参考下[源码](https://github.com/yinqiwen/ardb/blob/master/src/command/geo.cpp).

以下是一个简单例子， 存储两个坐标点：

    geoadd mygeo mercator 1000 1000 poi0 1001 1001 poi1

基于存储成员查找半径100m内的成员：

    geosearch mygeo member poi0 radius 100


## 查询性能优化
由于Ardb的默认底层存储是RocksDB/LevelDB/LMDB等持久化的存储实现，且GeoSearch实现涉及大量的范围查找，当数据量非常大的时候，查询性能较低。  
Ardb支持基于High Level的Key级别的cache， 对于Sorted Set而言，就是整个Sorted Set可以被cache到内存中，从而可以大大优化查询性能。默认情况下，cache是被关闭的，打开需要修改启动配置文件ardb.conf中的相应配置项并重启：  

     L1-zset-max-cache-size    10000    //cache中最大的sorted set个数
     L1-zset-read-fill-cache   yes      //sorted set读操作填充cache
     L1-zset-seek-load-cache   yes      //sorted set范围读取时触发加载到cache


## 性能
这里给出一个简单测试数据以供参考：  
机器配置：2个物理CPU、每个CPU 6核、12个Process， 未开启超线程，CPU型号：Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz， 内存64G，硬盘1.4T   
存储3000w+的坐标数据， Ardb启动时开启4线程   
cache开启：  半径50m， 22000 query/s； 半径100ms， 19000 query/s   
cache关闭：  半径50m， 9000 query/s； 半径100ms， 7000 query/s  （此结果存疑，应是disk cache影响，实际应用中性能应该远小于此）  



## Redis移植
由于实现基本原理可以简称为GeoHash + Sorted Set， 可以非常简单移植到Redis上；  

实际中也有朋友参考Ardb的实现在Redis上实现了一个[Redis-Geo](https://matt.sh/redis-geo), 作者mattsta是redis的主要contributor之一，稳定性可以保证，喜欢纯Redis解决方案的朋友可以试用下。  

喜欢自己动手的朋友也可以参考以上说明自行实现一个。
