---
layout: default
title: Blog
---
## All Posts

{% for post in site.posts %}

---
### [{{ post.title }}]({{ post.url }})
Posted on {{ post.date | date_to_string }} | by {{ site.author }}

> {{ post.excerpt }}

[Continue reading..]({{ post.url }})
{% endfor %}
