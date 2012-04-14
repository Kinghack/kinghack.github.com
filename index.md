---
layout: page
title: Welcome
tagline: 
---
{% include JB/setup %}

###这可能是我的第一个真正意义上的个人BLOG。

###在这个世界上存在了这么久，总觉得这个世界是牛人的世界。

###吾辈只需要看看牛人的输出就好了。

###但又如何甘心不留下些痕迹呢？
 
###草木有本心，何求美人折？



    
    
##最近的吐槽：


<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



