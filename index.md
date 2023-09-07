---
title: home
---

## Posts:
<ul>
{% for post in site.posts %}
<li> 
    <a href="{{ post.url }}">{{ post.title }}</a>
    
    <p>
    {{ post.excerpt}}
    </p>
    <hr>
</li>
{% endfor %}
</ul>
