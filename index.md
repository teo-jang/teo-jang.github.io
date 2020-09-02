---
layout: default
crawlertitle: "Teo develog"
title: "develog"
---

{% for post in site.posts %}
    <article class="index-page">
        <h3><a href="{{post.url}}">{{ post.title }}</a></h3>
        {{ post.excerpt }}
    </article>
{%endfor}