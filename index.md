---
title: home
---

## Posts:
<ul>
{% for post in site.posts %}
<li> 
    [{{ post.title }}]({{ post.url }})
    {{ post.excerpt}}
</li>
{% endfor %}
</ul>
