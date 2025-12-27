---
layout: home
---

<style>
  body { filter: grayscale(100%); font-family: monospace; }
  a { color: #000; text-decoration: underline; }
</style>

## Articles

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a> - {{ post.date | date: "%Y-%m-%d" }}
    </li>
  {% endfor %}
</ul>