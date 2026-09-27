---
layout: default
---

有了 AI 依然踩坑。

{% for post in site.posts %}
- <small>{{ post.date | date: "%Y-%m-%d" }}</small> [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
