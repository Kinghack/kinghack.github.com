---
layout: post
title: "aws中国区踩坑：spark从s3中读文件"
description: ""
category: 
tags: []
---
{% include JB/setup %}
Update in 2016.08.24. Succeed finlly. Please read this [blog](http://blog.qiuqiu.info/24/08/2016/thehardwayforsparktoreaddatafroms3).

今天一天掉在坑里起不来，所以用一篇blog记录下来折腾。

Have马上就会开始扩类目，而不是之前单纯的自行车。既然扩类目了，那我们的产品又没有设计一个实际的类目层，所以为了避免其他方面的用户进来后冲刷现在的体验，产品上要做的准备其实一个是加强对tag的使用，另一个就是走推荐的路子。那很明显，以我们现在的体量，说推荐都是有点虚的，可以做的就是基于用户的行为按照规则进行推荐。那技术上的储备就是将所有的数据存下并且可以分析。

Have的现在数据主要是两类，第一种是数据库中的数据，ad,user,like等等，代表着用户实际的行为，这部分数据在mongo中；另一部分数据其实就是api的log，这部分数据目前都在web机器的本地存着。前者虽然mongo是可以跑mapreduce的job的，但性能怎么样不好说，而且势必真正跑的时候会影响线上数据库的性能，因此直接用mongo跑第一部分的数据在我这里pass了；第二个log现在都在本地，现在我主要是用于监控api的响应时间，方法其实很土啦，就是把log下载下来然后写个awk程序分析；log的存储和分析其实门道很多，复杂的可以用ELK的一整套方案来解决，但这个方案第一，很重，很复杂的一套；第二，他的主要效能应该还是监控，而不是偏程序化的行为分析；第三，我其实是自己试了下的，搭建了[logstash](https://www.elastic.co/products/logstash)想收集下日志，但发现logstash很耗资源，感觉我现在用的机器都跑起来很累，现阶段为了监控加资源，我觉得必要性不是很大。因此第二种数据处理的ELK方案也pass了。

这个时候我有点晕，想，不会要我搭一套hadoop集群去做这个事情吧。感觉又是一套需要维护的东西，不是很愿意。想到hadoop的时候，自然也就想到了spark。在前家公司时，也是有几个月在玩spark的东西，spark在我看来的好处是，第一，快！第二，天生的集群支持；第三，我作为开发出生，其实写程序比裸写sql写的溜多了；（这也是我不愿意用刚刚在中国区aws上线的redshift的原因，还有个原因就是贵。）spark其实是一个编程接口，如果要引入spark，要解决的问题其实是从spark中读取datasouce。那很明显的，我并不想从spark中去读mongo，于是我就想到了s3。看spark的文档，发现他是完全支持从s3中读文件的，并且就可以部分当HDFS用。那我的两部分数据mongo可以定期用mongoexport成json格式到s3，而log文件就更可以上传上去了，而且s3又便宜，看上去很美好，于是我昨天花了点时间重新把spark捡起来，download了一份日志文件在本地，先试着写写想要算或者统计的东西，还挺顺序，虽然scala的环境搭得我又要吐血了，但好在，我还会Python...

统计的程序写得差不多的时候，我想，那今天主要就是从s3中读数据吧？没想到，这才是今天掉坑的开始。。。

首先，对于我们这种没有历史负担的产品来说，当然下载一个最新的spark prebuild with
hadoop
latest啦。下载下来，简单的把文件从本地路径换成s3协议，发现报错，提示的意思就是s3协议不认识。明明官方文档说可以读的，于是就搜啊搜，看到了[这个](http://stackoverflow.com/a/28033408/1072544)。大概意思就是出于某种原因，实现n3协议的jar包没被包在hadoop2.6中，下载2.4就解决啦。但一个prebuild有300MB，我用中国区的s3其实下载得很辛苦，不是很高兴再下载一遍，于是就继续找解决方案，发现可以通过指定一个package来强制使用一份实现，像这样：
`
/bin/spark-submit --packages org.apache.hadoop:hadoop-aws:2.6.0 log.py
`
这个时候经过一些包的下载和安装，可以发现s3协议是能认出的，但这个时候是会提示没有设置access
key和secret；虽然这台ec2上是配置好aws的credentials，但估计spark的应用还是需要单独配置的。于是按照文档，配置了以下几种形式：

`   

    #method one
    os.environ["AWS_ACCESS_KEY_ID"] = "key"
    os.environ["AWS_SECRET_ACCESS_KEY"] = "secret"
    #method two
    sc._jsc.hadoopConfiguration().set('fs.s3.awsAccessKeyId', 'key')
    sc._jsc.hadoopConfiguration().set('fs.s3.awsSecretAccessKey', 'secret')
    #method three   
    conf
= SparkConf().setAppName("HaveLog").set("fs.s3.awsAccessKeyId","key").set("fs.s3.awsSecretAccessKey",
"secret").set("fs.s3.endpoint", "s3.cn-north-1.amazonaws.com.cn")
    #method four
    textFile = sc.hadoopFile('s3a://bucket/1.b',
                                 'org.apache.hadoop.mapred.TextInputFormat',
                                 'org.apache.hadoop.io.Text',
                                 'org.apache.hadoop.io.LongWritable',
                                 conf = {
            "fs.s3.impl": "org.apache.hadoop.fs.s3native.NativeS3FileSystem",
            'fs.s3.awsAccessKeyId': 'key',
            'fs.s3.awsSecretAccessKey': 'secret',
        })
    #method five
    textFile = sc.textFile("s3://key:secret@bucket/1.b")`

发现，竟然无一被识别出来，可能有经验的同学会看出来，其实这里应该用s3n协议，而不是s3协议。于是我将上面的几种配置换成s3n的方式，依然不work。不得已，翻文档，还提到了一句裸设环境变量的方法。试验一下，竟然读到credentials了，不过也不能高兴，因为错误换成了无权限。但这个key我明明刚才本地还使用了，怎么会无权限呢？

搜索了一番网上，搜到了一篇吐槽中国区s3坑的[文章](http://www.jianshu.com/p/0d0fd39a40c9)。这个时候我想，千万不要是这种坑啊，于是就去aws的美区开了机器，上传了文件，跑得一切正常。。。此时我有点想要打人，但其实更令人崩溃的地方还在后面。

为什么中国区的s3有特殊呢？简而言之，aws的signature计算方式有几种，其中最新的是v4。在一些老的region，是v4与之前老的方式v3啦之类的并存，但beijing
region是新的区，不需要做兼容，因此只实现了v4的signature算法。而spark中要实现对s3文件协议的解析和读取，其实是使用了一个叫[jets3t](https://bitbucket.org/jmurty/jets3t/wiki/Home)的库，这个库是在[0.9.3](http://www.jets3t.org/RELEASE_NOTES.html)才支持v4的算法。关于对这个算法的支持可以见这个[讨论](https://bitbucket.org/jmurty/jets3t/issues/183/)。

所以可以想象，spark只要用的版本不够新，必然会导致无法解析协议。那猜猜spark用的是什么版本？答案是[0.7.0](https://github.com/apache/spark/blob/master/pom.xml#L149)。。。这也是为什么美国区可以运行而中国区不行的原因。这个时候我长叹了一口气，难道要我重新编译spark？虽然看着是蛮好玩的事情，但第一费时间，第二费机器，并不太想这么干。但如果不得不这么做，只要解决问题，也未尝不可。但继续搜索一番之后，我觉得这可能是一个[神坑](https://github.com/apache/spark/pull/9306)。于是重新编译就pass了。

只能继续寻找方案了。但坑爹的是，网上的方案提得很少，终于在spark的邮件列表中搜到了用s3a协议代替s3n的[推荐](https://mail-archives.apache.org/mod_mbox/spark-user/201503.mbox/%3C9E0825F6-5BD2-48B3-B396-A6A151445BCC@hortonworks.com%3E)以及另一个人的[blog](http://deploymentzone.com/2015/12/20/s3a-on-spark-on-aws-ec2/)。而关于s3,s3n,s3a的区别可见这个[link](https://wiki.apache.org/hadoop/AmazonS3)。这次我学聪明了，先去美区跑一跑，发现很顺利的读取了，于是再回中国区测试，这次不能用hadoop-aws:2.6.0了，而要用一个更新的2.7.1，因为这个版本才实现了s3a。不出意外的，中国区继续挂。不过这次提示不是没有权限，而是bad
request。

踩坑至此，坑爹的一天总算结束了，没错，中国区aws中spark读s3的问题依然没有解决，不过放在眼前的路已经蛮清晰了：

* 攻克那个v4签名的问题
* 找aws的技术看s3a bad request的问题
* 实现功能优先，老子现在又不是本地存不下





