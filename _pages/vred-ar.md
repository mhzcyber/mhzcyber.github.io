---
layout: archive
permalink: /vred-ar/
title: "VRED-AR"
author_profile: true
---

{% assign tag_filter = "vred-ar" %}
{% assign filtered_posts = site.posts | where_exp: "post", "post.tags contains tag_filter" %}

<div class="tagged-posts">
    <h2 class="archive__subtitle">VRED-AR Posts</h2>
    {% if filtered_posts.size > 0 %}
        {% for post in filtered_posts %}
            {% include archive-single.html %}
        {% endfor %}
    {% else %}
        <p>No posts found with VRED-AR tags.</p>
    {% endif %}
</div>