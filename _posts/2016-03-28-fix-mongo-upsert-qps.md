---
layout: post
title: "Mongo QPS排查"
description: ""
category: 
tags: []
---
{% include JB/setup %}
本文记录了一次对于线上Mongo QPS低的调查。

事情是这样的，我发现线上最大的一张表，某些操作的时候，QPS只有10左右。慢得不正常。开始不是非常在意，因为觉得很有可能是我们机器配置低的原因。就无视了几天。但每当执行这操作的时候，不但QPS低，还会把机器的LA变得很高。为了保险起见，有一天上午花时间看了下。

这个操作其实是[upsert](https://docs.mongodb.org/v3.0/reference/method/Bulk.find.upsert/)。写这个代码的时候，主要是为了方便，且这个upsert还是bulk操作的。于是在测试前，我大概想了几个原因，

* 第一，首先验证upsert的QPS。（在空机器上验证，保证没有干扰）
* 第二，验证upsert慢，同时试试insert速度；（看看upsert里哪部分慢）
* 第三，看看bulk操作的batch count的影响；
* 第四，观察机器LA和mongotop的变化；
* 第五，换更好的机器配置，看看提升，为之后扩容心里留个底；

采用cloud的好处就是，我很简单的就是为数据盘建了个snapshot，然后再从中建了一块硬盘，很轻松的就部署了一台机器。于是开始第一步，验证upsert的数据，的确非常慢，QPS很低；然后验证update操作的速度，依然很慢；这个时候我开始想，之前有人提到，mongo只适合insert，不要频繁去update的事，想，果然如此啊。

之后将batch
count的调整了下，发现在QPS提升上没有什么提升。系统LA也没什么变化。接着，就开始测试insert的操作，结果让我非常震惊：QPS提升到了**14000**左右！这个时候，我脑中想得还是看样子mongo不能去update呀。看到insert的QPS可以提升以后，我心里大概就有底了。因为upsert主要是为了省事，大不了我就分开了写。

接着验证换配置对性能的提升。将机器的配置翻了个倍。验证upsert，很沮丧的发现，QPS*竟然没有*提升，update结果也差不多，但系统的LA较小。反而是insert继续提升了些。因为业务上其实大部分还真的就是insert，所以我很快将代码稍微改了下，判断了需要update和insert的情况，然后乐呵呵的去吃午饭了。

边吃午饭的时候，边想，mongo的upsert也实现得太差了吧，有机会要去看看他的实现。同时也有点担心，因为第一，发现机器配置的提升竟然没什么作用；第二，虽然大部分情况是insert，但一旦遇到update，还是要SB。就边吃饭边想怎么处理这个情况。想着想着，想upsert不就是先find，然后再update或者insert吗？insert这么快，不就是find慢吗？那find慢的原因不一般就是**索引没加**吗？但是我这个表是加了索引的呀。难道索引不对？

午饭后，继续开测。将刚才的测试机器开起来，首先原来索引的情况下，将upsert的查询条件弄进去explain了下，悲剧的发现，果然是**collscan**。。。于是马上换索引，重新测试，upsert的QPS也正常了。这个时候继续想，看样子之前配置升级没效果的原因其实很简单，因为大部分时间都在等磁盘IO了吧，CPU根本就跑不满。现在索引对了，理论上配置升级应该有提升，果断验证了下，发现如预想一致。

至此，一块心病算解决了。

分析了下悲剧的原因：主要是自己只考察了所有项目中查找的情况，按照查找的逻辑建了索引，直接说，就是搜了项目中find的地方去分析，却完全忘记了update/upsert这样的操作也是要先去find的。于是顺手就继续排查了下其他的update和upsert。

在第一波测试的时候，盯着[mongotop](https://docs.mongodb.org/manual/reference/program/mongotop/)看，发现坑爹的是，mongotop在upsert处理时，read是0ms，而write是一个正常的数值。如果能正常区分read和write，也许我能更快找到问题。

