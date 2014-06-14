---
layout: post
title: "Elasticsearc geo related feature折腾"
description: ""
category: 
tags: []
---
{% include JB/setup %}

这两天都在折腾es相关的geo特性，主要是按距离排序以及满足一定距离范围你这样的查询。

总结一下：

###背景知识

要想将geo信息作为一个字段build进es，有两种mapping type可选： [geo_point](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/mapping-geo-point-type.html), [geo_shape](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/mapping-geo-shape-type.html)。

两种的区别是，geo_shape主要是用于按图形范围内查找，虽然也可以用图形的范围来满足距离范围内的需求，但当距离是一个range，就不能支持，只能通过多次查找并且再筛来解决；geo_shape也不支持按照距离排序这一属性；

使用geo_shape这样mapping，就将主要使用[geo shape query](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-geo-shape-query.html)和[geo shape filter](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-geo-shape-filter.html)。

而使用geo_point时，则使用[distance filter](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-geo-distance-filter.html)和[distance range filter](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-geo-distance-range-filter.html)。

###使用场景

两者的区别描述清楚以后，再来看使用场景。(只考虑用es**自身能力**解决)

  1. 查询一定距离内（geo_point, distance filter）
  2. 查询距离范围内（geo_point, distance range filter）
  3. 按距离排序（geo_point, sort）
  4. 指定形状内（geo_shape, geo shape query/geo shape filter）
  
###性能比较

两者的性能也做了粗略的比较。

环境：在本地新建两个新的index：ad3和ad4。都搜出5000条数据出来往里面build，唯一不同的是ad3中build的是geo_shape这样mapping的字段，而ad4中则为geo_point。

两者的index大小基本无差距。

![image](http://ww1.sinaimg.cn/large/697dc0c0gw1ehdfwki6uej20cl02oq31.jpg)

然后开始搜索，所有查询都是执行1000次，执行时间单位为ms，filter的cache没开。

第一个是搜ad3，查找一个envelop框内的数据，一共耗时6790.77。

![image](http://ww4.sinaimg.cn/large/697dc0c0gw1ehdfxpmxfcj20v010en0j.jpg)


第二次查询ad4，用distance filter，一共耗时3407.832。

![image](http://ww3.sinaimg.cn/large/697dc0c0gw1ehdfyjnkzwj20rc0ii769.jpg)

第三次依然查询ad4，但同样的距离范围内，改用distance range filter，一共耗时3383.519。

![image](http://ww1.sinaimg.cn/large/697dc0c0gw1ehdg03o5hbj20u80jcwgm.jpg)

![image](http://ww1.sinaimg.cn/large/697dc0c0gw1ehdg4qm28lj218g044t9s.jpg)


看上去geo_point这样的type带来的性能**更好些**。

当然数据不多，测试可能不够严谨。

###结论

但就目前的测试结果来看，无论从性能上，还是从未来的需求上来看，我们都应该将现在的type由geo_shape换成geo_point。

###TODO
  1. 更严格的性能测试；
  2. 加入geo related filter后的性能调优；

