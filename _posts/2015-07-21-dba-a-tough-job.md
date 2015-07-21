---
layout: post
title: "SA&DBA真是很有技术含量的工作"
description: ""
category: 
tags: []
---
{% include JB/setup %}
一周前脑子一热加入了一个全新的项目，之前在公司的体系下开发，有专门的SA和DBA工程师，所有的东西不用管配置，只管在业务开发上直接使用就好，当然滥用资源也会被拉出去骂。这个新项目目前就我一个技术，各种数据库配置之前都没玩过，今天配下来，还算顺利。尽管只是简单配置，还不涉及服务器稳定以及性能调优，就已经挺花时间了，所以发出了标题的感叹。

本文稍微记录下步骤，防止之后忘记。。大部分是通过官方文档的，少部分是网络找的。

本文的目的是将postgresql和mongo能够在服务器运行，并且能通过账户远程访问。

先讲[postgresql](http://www.postgresql.org)。

装数据库这种就完全不用说了吧，我是直接用yum安装的，不过ec2的AMI包里只有9.2的版本，即使按照文档添加了9.4的源，仍然在包依赖的时候出问题。但考虑到这不是我今天想解决的主要问题，就先不管了。

官方文档是建议在运行postgresql时建一个专门的用户运行。当然还是尽可能使用推荐的最佳实践了。

于是添加用户和改密码：

`sudo adduser postgres`

`sudo passwd postgres`


再然后为运行的数据库建立folder：

`mkdir /usr/local/pgsql/data`

接着改权限：

`chown postgres /usr/local/pgsql/data`

接着就可以切换用户去运行啦：

`su postgres`

`initdb -D /usr/local/pgsql/data`

接着是创建用户及权限：

`psql -U postgres`

`ALTER USER postgres with password 'pwd';`

`CREATE USER have WITH PASSWORD 'pwd';`

`CREATE DATABASE have OWNER have;`

`GRANT ALL PRIVILEGES ON DATABASE have to have;`

但这样运行的数据库，是不需要任何密码就直接登录的，当时我就直接吓尿了，因为还在测试环境，所以这个数据库跑的ec2还是在公网环境下的，还是需要尽快加上权限。

在刚才数据库的目录下，找到postgresql.conf配置文件，改一个配置项：

`listen_addresses = '*'`

再找到pg_hba.conf这个配置文件，改下以下行为：

`host all all 0.0.0.0/0 md5`

以上还可以稍微修改下，来限制只能通过特定的IP访问才能访问数据库，应该会更安全些。

改完后，再以目前的超级用户登录进去：

`psql -U postgres -d have -h 127.0.0.1 -p 5432`

再执行：

`SELECT pg_reload_conf();`

就能直接reload配置啦。

有了postgresql的经验，再配mongo就更顺利些了。安装启动什么的继续不说了。

首先在没任何配置的情况下，先mongo shell进去：

`mongo`

接着创建一个database:

`use offhave`

接着为这个数据库创建两个用户：

`db.createUser({user:"admin",pwd:"pwd",roles：["readWrite","dbOwner","userAdmin"]})`

`db.createUser({user:"web",pwd:"pwd",roles:["readWrite"]})`

这两个用户，admin我准备用来后面的管理，而web我就准备给web程序用，只有读写权限。

接着去mongo的配置文件，去掉这行的注释：

`auth=true`

接着注释掉这行：

`bind_ip=127.0.0.1,x.x.x.x`

这样的意思是原来只允许本机访问，注释了以后就能通过任何地址访问了。当然上面的格式也支持之后可以指定地址。

做完这些后，重启mongo服务：

`service mongod restart`

接着就能用如下命令登录啦：

`mongo offhave -u admin --authenticationDatabase offhave`


之后我应该会研究mongo最简单的集群配置，因为不想跑在单机模式下，但之后量上来改成集群又很麻烦，毕竟不是专业的DBA和SA，想一开始把各种可能先考虑到。

