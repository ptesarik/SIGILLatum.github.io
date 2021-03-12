---
---

SIGILLatum is my personal tech blog about all things related to Linux.

{% for post in site.posts %}

# [{{ post.title }}]({{ post.url }})

{{ post.date | date: "%a, %d %b %Y" }}

{% if post.shortinfo %}
{{ post.shortinfo }}
{% elsif post.description %}
{{ post.description }}
{% else %}
{{ post.excerpt }}
{% endif %}

{% endfor %}
