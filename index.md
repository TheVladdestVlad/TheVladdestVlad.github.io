## Index page is currently under construction

```
Will add some more content to the index page soon
```

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

{% include archive.html %} 

{% include social-media-links.html %}
