---
layout: post
title: "Ardb内部编解码格式"
description: ""
category: Ardb
tagline: "Supporting tagline"
tags: [Ardb]
---

在Ardb的设计中， 编解码层是一个很重要的环节。编解码层的存在， 屏蔽了各种kv存储实现的不同，可以在任意一个简单的kv存储引擎上， 封装实现string， hash， list， set， sorted set等复杂类型的数据结构。这里简单介绍下Ardb中的编解码层的设计实现。

## Overview
首先，有这样的设计约定：  

- 所有的数据结构，都可以被一对或者多对简单key-value表示
- 每种数据结构，至少包含一个元信息key-value对(meta)，用于保存额外的信息（db，类型，expire时间等）；如hash、set、list、sorted set等复杂结构均是每个key对应有一个meta


## 内部Key的格式
对于所有的Key， 包含同样的前缀，编码格式定义如下：

     [<namespace>] <key> <type> <element...>

Note： namespace 为可选部分。
<!--more-->

- namespace用于支持类似redis中的库概念；可以为任意字符串
- key则是一个变长二进制字符串
- type用于定义一个简单key-value的类型，此类型隐含表明key的数据结构类型;一个字节
- meta信息的key中type固定为KEY_META；具体类型将在value中定义（参考下一节）
- 除以上三部分外，不同类型的key可能有附加字段；如Hash的key可能需要附加field字段
     

## 内部Value的格式
内部Value则比较复杂，编码均以type开始， type取值即上节定义的KeyType
 
     <type> <element...>

后续格式根据各种类型定义不同.

## Codec
所有编码格式如下：


                         KeyObject                                                   ValueObject
    String      [<namespace>]  <key> KEY_META                                   KEY_STRING <MetaObject>
    Hash        [<namespace>]  <key> KEY_META                                   KEY_HASH <MetaObject>
                [<namespace>]  <key> KEY_HASH_FIELD <field>                     KEY_HASH_FIELD <field-value>
    Set         [<namespace>]  <key> KEY_META                                   KEY_SET <MetaObject>
                [<namespace>]  <key> KEY_SET_MEMBER <member>                    KEY_SET_MEMBER
    List        [<namespace>]  <key> KEY_META                                   KEY_LIST <MetaObject>
                [<namespace>]  <key> KEY_LIST_ELEMENT <index>                   KEY_LIST_ELEMENT <element-value>
    Sorted Set  [<namespace>]  <key> KEY_META                                   KEY_ZSET <MetaObject>
                [<namespace>]  <key> KEY_ZSET_SCORE <member>                    KEY_ZSET_SCORE <score>
                [<namespace>]  <key> KEY_ZSET_SORT <score> <member>             KEY_ZSET_SORT

## Del的实现
所有的数据结构都有保存meta的一个key-value，而meta信息的key编码格式是统一的，因此在ardb中是不可能出现不同数据结构有相同名字的情况。  
Del实现中会先查询meta的key-value，得到具体数据结构类型，然后执行对应的删除工作， 类似如下的步骤：  

- 查询指定key的meta信息，得到数据结构类型
- 根据具体类型，执行删除工作
- 所以一次del至少需要 一次读 + 后续删除写操作 


## Expire的实现
在一个key-value存储引擎上支持复杂数据结构的expire过期数据的实现比较困难，ardb中则用几个特殊技巧实现了对所有数据结构的过期（expire）的支持，感兴趣的朋友可以一起讨论下。  
具体实现如下： 

- meta的value中保存expire信息， 用绝对unix时间（ms）保存；
- 基于以上设计，ttl/pttl等查询ttl的操作只需要一次读meta即可完成；
- 基于以上设计，任何对meta信息的读取，都会触发expire的判断，由于对meta信息的读操作是必须的步骤，这里无需额外的读操作
- 创建一个namespace `TTL_DB`专门存放TTL排序信息。
- 保存设置expire时间到meta时， 当expire时间非0时，额外保存一个key-value， type为`KEY_TTL_SORT`； key的编码格式为 `[TTL_DB] "" KEY_TTL_SORT <ttl> <key's namespace> <key>`, value为空；所以类似`expire <key>`等设置过期时间的操作，在ardb的实现中将会多一次写操作；
- 在自定义的comparator中，对`KEY_TTL_SORT`类型的key比较规则为先比较`<expire time>`，这样`KEY_TTL_SORT`数据将会以过期时间远近保存在一起
- ardb中独立启动一个线程，每隔一定时间（100ms）顺序扫描`KEY_TTL_SORT`类型数据；当过期时间小于当前时间，即可触发删除操作；当过期时间大于当前时间，即可终止本次扫描

