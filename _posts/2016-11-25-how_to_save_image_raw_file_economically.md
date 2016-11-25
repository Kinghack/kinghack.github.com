---
layout: post
title: "爱拍照的程序员该怎么备份RAW照片？"
description: ""
category: 
tags: []
---
{% include JB/setup %}
写在前面，本文不是关于程序员写程序解决备份问题，虽然这是个不错的**sideproject**方向，但对于目前的我比较耗时间。本文是关于利用现有开源工具，开源硬件，付费商业服务来廉价安全的备份数据的思考和折腾过程。关键词： **Seafile**, **Amazon Cloud Drive**, **Raspberrypi**, **acd_cli**。


我们知道，现在信息是**爆炸**的，各种数据测出不穷，文件数据类的数据属于比较敏感，但体积不大的；而生活中更常见的是照片、视频类似的数据存储需求，这类东西除了有癖好的个别人，大部分属于不算敏感，但体积实在太大的。尤其是现在动不动就是高像素的存raw，以及高分辨率的拍1080P，这类文件真是实在太大了。

这些数据不存吧，总感觉有各种回忆，存吧，怎么解决呢？这是我作为一个只拍raw格式的人始终在想办法解决的问题。这篇Blog也主要是讨论raw的照片怎么存储；我大概列一下可能的几种方案：

1. 云存储，类似Dropbox, Google Drive, iCloud, Box, Amazon Cloud Drive,百度云盘等等；
2. NAS类解决方案；
3. 自己移动硬盘存；


先说最后一种移动硬盘存好了。这种方式**优点**就是便宜，方便，任何用户都可以上京东随便买个大牌的移动硬盘，然后就把数据一股脑的丢进去就好。需要用数据的时候，把硬盘连上电脑即可；然后当数据继续扩张的时候，继续买硬盘，然后就会有很多硬盘存在。缺点嘛，硬盘会坏，虽然目前这个事情我没有遇到过，但随着你硬盘越来越多，遇到的机率是会增大的，而一旦硬盘坏掉，数据的安全性就有很大的疑问。所以这个方式虽然我一直将就着，但也一直在想更优的办法；

然后说说NAS；这个其实是非常专业的方案，NAS可以组各种RAID0啊10啊什么的，有各种数据安全与空间的折衷衡量可能。他还有个优点是，NAS通常都在自己的办公室或者家里，相比云盘，拿回这些数据的便利性其实比移动硬盘还要高，如果考虑现在手机上各种APP访问的话，前提是你有足够快的内网。但缺点也有，一个是成本，NAS本身硬件加软件就是一个商业解决方案的成本，便宜的话，1K搞定，要更专业的话，是可以[**很贵很贵**](http://item.jd.com/1277027.html)的。。。另一个是可扩容性，现在看到的偏小型商业应用的NAS方案，最大似乎也就16TB，如果拍照片拍的勤快些，真的能撑很久吗？我表示怀疑。我现在相机里的双卡就都是64GB的，未来RAW的照片也只会越来越大。

剩下的就是基于Cloud的方案了。这也是本文想说的**重点**。当然，如果用云的话，有两个主要的**缺点**，第一个数据离得比较远，无法随时访问到，尤其是没有网络的时候，这里因为我主要的需求是安全便捷经济的存储，随时可访问可以在存储的基础上，进一步优化；第二个缺点是，云的速度问题，尤其是在墙内的我们。

那如何选云呢？我首先排除了国内的，虽然速度有优势，但是现在国内的云一家挨着一家在收紧，未来不可控性太大。那国外的云又怎么选呢？

来列列看：

1. [Dropbox](https://www.dropbox.com)；大牌，技术应该足够好；缺点是，贵，被墙；所以PASS；
2. [Google Drive](https://drive.google.com)；更大牌；缺点，贵，被墙；所以PASS；当然，如果你有Google Apps或者Google EDU的话，且又在墙外的话，选这个吧；
3. [iCloud](https://www.icloud.com)，似乎没看到有谁用他，用于同步设备状态还好，同步数据还是算了吧；


最早我看得就这几个，但无疑都比较贵，且空间有限；所以当时我是用了Google的付费方案，把比较重要的数据同步上去，大部分不是那么重要的数据，就先不去占cloud的空间了；

因为现在的项目都跑在AWS上，用了比较多的S3服务，慢慢又在想，是不是个人用S3存其实成本也是能接受的？虽然没尝试，但后来看到Amazon.com竟然有单独的Amazon Cloud Drive的服务，而更吸引人的是，他有非常廉价的**unlimited photo plan**，而且支持RAW格式！这对我的需求来说，简直是超级利好；而之前Google Photos改过一次版，支持unlimited photo，但前提是第一个必须是jpg，第二是一定分辨率下；所以Google 这个服务可以当作是一个浏览照片的**补充**方案。

选定了 Amazon Cloud Drive，剩下的问题是怎么同步上去；他有MAC的客户端，但要我开着电脑同步10GB级别的数据，那我不如买NAS了。所以就需要一个更低成本的方案。正好家里有个树莓派连着路由器跑着，所以就决定用他。可以在晚上的时间慢慢跑，而Amazon Cloud Drive总是用api开放的吧，总是有工具能做的话，找了找，果然找到了一个叫 [acd_cli](https://acd-cli.readthedocs.io/en/latest/usage.html)的工具，虽然授权需要去网页做，但授权回来的token自己存到命令行下去也就可以run起来的，但问题是什么呢？问题是需要refresh token，而refresh token是要去appspot上挂的一个小代理做得，这个功能是**被墙**的。。。导致自动run起来的备份方案一旦遇到刷新token的时候，就会失败。那我可以配翻墙上传吗？理论上是可以的，但因为我现在的翻墙方案是在树莓派上跑的kcp proxy，而kcp似乎在上传的时候会很严重的吃掉带宽和树莓派的性能，进而导致其他的正常网络服务受影响，因此一直没继续折腾下去，直到前两天我看到了 [Seafile](https://www.seafile.com)这个神器。

简单来说， Seafile就是个人搭建私有云的开源工具，应该是国人作品，做得很棒，文档什么的非常好，很轻松的就在我的VPS上搭建起来。他也有官方的MAC客户端，试了下效果，除了VPS网速本身的问题外，都工作的非常好。于是我就想，要不我就这样，用树莓派不翻墙的通过 Seafile 备份数据去 VPS，然后VPS 通过刚才的acd_cli备份数据去 Amazon Cloud Drive；基本上经过之前的折腾，都是现成的，除了，seafile没有官方的树莓派的client，需要自己去编译。还好，不是很难。

因为国内的树莓派镜像似乎缺少一些包，所以还是先把软件镜像配到官方去，然后先

`sudo apt-get update
sudo apt-get upgrade`

然后，安装，

`apt-get install doxygen cmake sqlite3 libsqlite3-dev openssl libssl-dev libevent-2.0-5 libevent-dev python-pip libjansson-dev automake libtool libglib2.0-dev uuid-dev valac libfuse-dev libcurl4-gnutls-dev qtbase5-dev qttools5-dev qttools5-dev-tools qt4-qmake git`

然后，clone这三个包，

`git clone https://github.com/haiwen/ccnet.git
`
`git clone  https://github.com/haiwen/seafile.git
`
`git clone https://github.com/haiwen/seafile-client
`

再分别进去这三个包，分别安装下，
`./autogen.sh
`
`./configure
`
`make
`
`sudo make install
`
装完，就会有一个 seaf-cli可以使用了。

先init

`seaf-cli init -d /your_to_sync_work_space
`

然后

`seaf_cli start
`

接着看看Seafile有什么 Library 可以同步，

`seaf-cli list-remote -s “your seafile server address" -u “username"
`

当需要同步本地的某个文件价过去的时候，

`seaf-cli sync -l “library" -s "your seafile server address" -u “yourusername" -d syncfolder/
`
同步状态可以下面的命令来查看

`seaf-cli status
`

当同步完成以后，就登录上你的VPS，如果之前acd_cli就配好的话，就可以先列一下 Amazon Cloud Drive上的文件目录

`acd_cli --utf ls
`

然后因为Seafile存储文件是通过block的方式存的，在VPS上不能直接找到文件，需要用一个叫FUSE的插件，运行如下命令就可以

`bash seaf-fuse.sh start /your_mount_folder_for_file/
`

接着就是直接同步

`acd_cli /your_mount_folder_for_file/  amazon_path_id
`

VPS上上传10MB的速度就分分秒秒上传完了，真是爽翻天。

至此，整体流程已经全部通了。

简单总结下，在墙内备份RAW照片可能是一个比较独特的需求。本文中，使用 Seafile 同步我树莓派和VPS中的数据，然后通过 acd_cli 同步 VPS和 Amazon Cloud Drive的数据；

你需要的是：
一台安装了 acd_cli和 Seafile的VPS，一台安装了sea-cli的本地24小时在线的机器，一个Amazon Cloud Drive帐号。



有点***不折腾会死星人***的意思。。。。

![](https://ooo.0o0.ooo/2016/11/25/5837f56da8538.png)
