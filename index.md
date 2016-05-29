---
layout: index
---

I am Bijay K Pathak.

{% include cv.md %}

# Recent Blog Posts

<ul>
  {% for post in site.posts limit:5 %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> &mdash; <span>{{ post.date | date_to_string }}</span>
    </li>
  {% endfor %}
</ul>
<h4><a href="/blog">View all</a></h4>

# Fun Side Projects
---

Last updated on {% include last-updated.txt %}
