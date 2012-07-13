---
layout: page
title: The art has gone.
tagline: 
---
{% include JB/setup %}

草木有本心，何求美人折?
<br/>
All stories, if continued far enough, end in *death*.
<br/>
The void is *not* nothing.
<br/>
<br/>
<br/>
<meta property="wb:webmaster" content="d77dbd93aa59a3b4" />

		  
##Recently post：


<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



