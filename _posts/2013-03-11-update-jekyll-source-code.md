---
layout: post
title: "更新jekyll源代码"
description: ""
category: 技术折腾 
tags: [git]
---
{% include JB/setup %}
太久没跟着项目用git开发，一些协作的东西都忘记了。今天想更新jekyll源代码时，都忘记怎么弄了。

这个blog搭建于一年前，当时[jekyll](https://github.com/mojombo/jekyll)还是0.2的版本，今天过去看了下，已经到了0.3版本了。于是想更新。

注意，我的blog是直接folk了这个[项目](https://github.com/plusjade/jekyll-bootstrap)。


首先，添加远程仓库：


>git remote add jekyll git@github.com:plusjade/jekyll-bootstrap.git

然后，从远程仓库抓取数据：


>git fetch jekyll

最后，将抓取的数据合并到自己的分支中，可能有冲突需要解决。

>git checkout master

>git merge jekyll/master


之前自己以为过程很简单，结果一直死在这个[问题](http://stackoverflow.com/questions/3965676/why-did-git-detach-my-head)上。

最后参考这份[资料](http://blog.jobbole.com/25808/)更新成功。

