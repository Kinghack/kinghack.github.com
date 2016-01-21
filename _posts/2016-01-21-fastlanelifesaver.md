---
layout: post
title: "FastLane真是我生活的救星"
description: ""
category: 
tags: []
---
{% include JB/setup %}
因为现在[Have](https://itunes.apple.com/cn/app/have/id1060985983)的协作方式是两个远程的iOS开发和我，将他们的apple
id加入公司的开发帐号后，他们都能做本地的开发，但分发的证书目前还是只有我这里有，因此每次更新需要发adhoc版本给team里人测的时候，都是我这边来弄的。

我之前没搞过iOS开发，因此第一次弄的时候，各种certificate, provisioning
file都搞得我头非常大，最后虽然都能正常build了，但当临近发版的时候，大概提一次代码就需要发版一次，这一次发版需要我做如下事情：

- Merge pr
- Pull code to local
- Change config file by **hand**
- Switch scheme for adhoc
- Archive
- Export by adhoc method
- Upload ipa file to test environment
- Write release notes by commit msg
- Notify team
- Change back config file by hand **again**

当高峰期需要大概10分钟就做一次以上步骤的时候，你的生活是不会幸福的。因此一直是想着快点上持续集成，能够回归正常的生活。但发了[帖子](http://www.v2ex.com/t/244046#reply4)收mac
mini也杳无音信，让我现在买新的我也心有不甘，所以这事儿就一直黄着。

昨天下午稍微空一些了，突然想到前两周swift大会时[喵神](http://onevcat.com)提到过的[FastLane](https://fastlane.tools)，就想去看看能不能部分的解决我的问题。就过去看了下。实乃神器！

之前是根据这篇[文章](https://engineering.circle.com/different-app-icons-for-beta-dev-and-release-builds/)配置了不同的compile策略使得测试版，开发版，正式版能共存与手机上。这个为背景。下面是主要的FastLane的配置：

    lane :beta do
        gym(scheme: "HaveAdHoc", clean: true, use_legacy_build_api: true,
    export_method: "ad-hoc", output_directory: "~/Desktop/", output_name:
    "HaveAdHoc", silent: true)
            # Build your app - more options available
        commitmsg = changelog_from_git_commits(
            pretty: '- (%ae) %s', # Optional, lets you provide a custom format to
    apply to each commit when generating the changelog text
            include_merges: false # Optional, lets you filter out merge commits
        )
        slack(
          message: commitmsg,
          channel: "#newversion",  # Optional, by default will post to the default
    channel configured for the POST URL.
          success: true,        # Optional, defaults to true.
          payload: {            # Optional, lets you specify any number of your own
    Slack attachments.
            'Build Date' => Time.new.to_s,
            'Built by' => 'qiuqiu',
          },
        )
        add_git_tag(
            tag: Time.new.strftime("%Y%jT%H%MZ"),
        )
        sh "your upload script"
        slack(
          message: "down load from here http://xxxxx",
          channel: "#newversion",  # Optional, by default will post to the default
    channel configured for the POST URL.
        )
    end

经过以上配置，现在要发achoc版本的时候，只需要：

- Merge pr
- Git pull code
- Fastlane beta
- Drink coffee

今天上海那么冷的天，也瞬间觉得温暖了呢。

![](https://ooo.0o0.ooo/2016/01/20/56a03dae09e52.jpg
)
