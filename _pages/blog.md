---
title: "Blog"
permalink: /blog/
layout: blog
author_profile: false
excerpt: "Technical notes and long-form writing."
---

{% assign posts = site.blog | sort: 'date' | reverse %}

<div class="blog-home">

  {% if posts.size > 0 %}
    <div class="blog-home__meta-row">
      <p>{{ posts.size }} post{% if posts.size > 1 %}s{% endif %}</p>
    </div>

    {% for post in posts %}
      <article class="blog-card">
        <div class="blog-card__topline">
          <p class="blog-card__eyebrow">{{ post.date | date: "%Y-%m-%d" }}</p>
          {% if post.tags and post.tags.size > 0 %}
            <p class="blog-card__tags">{% for tag in post.tags %}<span>{{ tag }}</span>{% endfor %}</p>
          {% endif %}
        </div>
        <h2 class="archive__item-title"><a href="{{ post.url | relative_url }}" target="_self">{{ post.title }}</a></h2>
        {% if post.excerpt %}
          <p class="archive__item-excerpt">{{ post.excerpt | markdownify | strip_html | strip_newlines }}</p>
        {% endif %}
        <p><a class="blog-card__link" href="{{ post.url | relative_url }}" target="_self">Read article</a></p>
      </article>
    {% endfor %}
  {% else %}
    <p>No blog posts yet. Add a Markdown file under <code>_blog/</code> to publish your first article.</p>
  {% endif %}
</div>
