---
layout: post
title: "快速Interleaving bits"
description: ""
category: 技术
tagline: "Supporting tagline"
tags: [GeoHash]
---

昨天在reddit上提交了[geohash-int](http://www.reddit.com/r/programming/comments/2gg2i7/a_fast_c99_geohash_library_which_only_provide/) , 引来了一些讨论， 其中一位外国朋友Steve132给出了一个[更快的实现](https://github.com/yinqiwen/geohash-int/pull/1), 虽然里面有些较低级bug，但修改编译后，简单测试了下，100万次encode/decode性能提升了6倍之多，耗时统计如下：  
	
	Cost 425ms to encode
	Cost 75ms to fast encode
	Cost 671ms to fast decode
	Cost 98ms to fast decode

GeoHash encode/decode算法中一个关键步骤是Interleaving bits（位交错运算？可能需要更好的翻译）， 大致相当于将一个64bit的int，按二进制位将此int拆分为两个32bit的int， 其中偶数位的为一个， 奇数位的二进制位为另一个。 Steve132给出的实现主要就是优化这一个操作。 原来我的做法是老老实实做位偏移合并运算，算法复杂度为O(N), 其总N为二进制位个数。  

<!--more-->
Steve132给出的实现的复杂度则几乎是O（1），其中没有任何循环，原理则比较难以解释，感兴趣的朋友可以参考下面几个链接：  

- [How to efficiently de-interleave bits (inverse Morton)](http://stackoverflow.com/questions/4909263/how-to-efficiently-de-interleave-bits-inverse-morton)
- [Interleave bits by Binary Magic Numbers](http://graphics.stanford.edu/~seander/bithacks.html#InterleaveBMN)
- [How to de-interleave bits (UnMortonizing?)](http://stackoverflow.com/questions/3137266/how-to-de-interleave-bits-unmortonizing)

另外，以上第二个链接收集了很多关于位操作的"奇技淫巧"， 对于很多高性能工程场景很有用，感兴趣的朋友可以同时关注下。
