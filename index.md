---
layout: page
title: Welcome
tagline: 
---
{% include JB/setup %}

###在测试
 
###草木有本心，何求美人折？

###

###

 



    
    
##最近的声音：


<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



