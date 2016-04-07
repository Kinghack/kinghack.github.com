---
layout: post
title: "安利下netdata"
description: ""
category: 
tags: []
---
{% include JB/setup %}
[Have](https://itunes.apple.com/cn/app/id1060985983)上线后，所有的对系统的监控都是登录到机器后敲命令实现的。虽说现在量很少，这样做没有问题，但缺少直观的感受。这个时候我就无比想念原公司搭建的[Zabbix](http://www.zabbix.com)那一套，各种线上的数据都收集着，很容易搜log，定位问题，所以之前一段时间，我一直有点想花点时间给Have也搭建监控系统。一直没具体做的原因是：第一是怀疑性价比；第二也是真的一个版本接着一个版本的在发，没时间。

于是在考虑一些其他的方案。一开始想看得是小米的[falcon](http://open-falcon.org)，看到口碑不错；但感觉这套东西还是太大太重了。然后想尝试的是[oneapm](http://www.oneapm.com)。看oneapm的介绍是相当不错的，于是就搜了下这个东西，然后搜索结果中就出现了[netdata](https://github.com/firehol/netdata.git)。不得不说，第一眼数据可视化做的非常漂亮，然后看介绍，开箱即用，有自己的webserver,也挺简单的。于是就在测试环境中部署了。的确是开箱可用。正式环境当然不能直接放出来用，虽然官方有放在nginx后面的授权的[方案](https://github.com/firehol/netdata/wiki/Running-behind-nginx)，但我嫌麻烦，以及要更改nginx的配置，再加上Have的后台系统的授权已经做好了，想直接放在我们的后台系统后面，于是就用了下golang
的[reverseproxy](https://golang.org/src/net/http/httputil/reverseproxy.go)几行代码将对netdata的请求转发到了它的webserver。然后再在swift中的配置侦测哪些机器。

这个库具体应该还有很多配置可以用，但我现在暂时都用的默认，除了针对不同机器配不同的application外。如果要说目前的不足的话，一个是我现在只能一个个机器的去check，并没有一个很好的方式汇总所有的图表；第二就是，他拉取数据的方式是去一秒pull一下，其实挺傻的，官方issues里有对[这个](https://github.com/firehol/netdata/issues/130)的讨论。

netdata配好后，只是方便了我很简单的监控机器，但对于日志的分析还没有。现在对日志的分析是通过将日志全部汇总起来，然后写awk程序分析的，有点弱，后面也准备开始引入工具做这个事情。原来公司是使用的ELK一套，我还是觉得太重了。

因为本身不是sa出身，好好研究netdata中默认监控的数据也是个挺有意思的事情。 下一步还可以沉挖下netdata对特定应用的监控可以做得多详细，比如redis, mongo的各种数据。

最后截下图，还是很漂亮的。

![](https://ooo.0o0.ooo/2016/04/07/57062c67e40db.png)

