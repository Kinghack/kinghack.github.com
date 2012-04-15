---
layout: page
title: The art has gone.
tagline: 
---
{% include JB/setup %}

##草木有本心，何求美人折?
<br/>
###All stories, if continued far enough, end in *death*.
<br/>
###The void is *not* nothing.
<br/>
<br/>
<br/>
		  
##最近的声音：


<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



