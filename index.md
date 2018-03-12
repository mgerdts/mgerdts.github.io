---
layout: default
---

<meta name="twitter:card" content="summary" />
<meta name="twitter:site" content="@OMGerdts" />
<meta name="twitter:title" content="Virtually Anything" />
<meta name="twitter:description" content="Musings on virtualization and other
stuff" />
<meta name="twitter:image" content="https://mgerdts.github.io/desktop.jpg" />

Under construction

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
