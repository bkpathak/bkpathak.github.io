---
layout: index
---

Hi! I am Bijay K Pathak and welcome to my corner in the Internet.
I am currently working as Software/Data Engineer in [Nike](http://www.nike.com). I graduated from [Michigan Tech](http://www.mtu.edu) in 2014 with M.Sc. in Computer Science and Engineering with focus on machine learning and optimization. I am interested in data intensive application, functional programming and distributed system.  
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

# Side Projects
---
Update coming soon.

Last updated on {% include last-updated.txt %}
