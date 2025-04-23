---
layout: archive
permalink: /vulnerability-research/
title: "Vulnerability Research"
author_profile: true
---

{% assign tag_filter1 = "vulnerability-research" %}
{% assign tag_filter2 = "vuln-research" %}
{% assign filtered_posts = site.posts | where_exp: "post", "post.tags contains tag_filter1" %}
{% assign more_filtered_posts = site.posts | where_exp: "post", "post.tags contains tag_filter2" %}
{% assign final_posts = filtered_posts | concat: more_filtered_posts | uniq %}

<div class="tagged-posts">
    <h2 class="archive__subtitle">Vulnerability Research Posts</h2>
    {% if final_posts.size > 0 %}
        {% for post in final_posts %}
            {% include archive-single.html %}
        {% endfor %}
    {% else %}
        <p>No posts found with vulnerability research tags.</p>
    {% endif %}
</div>