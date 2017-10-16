---
layout: default
title: Categories
permalink: /categories/
---

<link rel="stylesheet" href="/assets/css/globals/responsive.css">
<link rel="stylesheet" href="/assets/css/globals/index.css">

<section class="container content">
    <div class="columns">
        <div class="column three-fourths">
            <article class="article-content markdown-body">
                <section class="container posts-content">
                    {% assign sorted_categories = site.categories | sort %}
                    {% for category in sorted_categories %}
                    <h3>{{ category | first }}</h3>
                    <ol class="posts-list" id="{{ category[0] }}">
                        {% for post in category.last %}
                        <li class="posts-list-item">
                            <span class="posts-list-meta">{{ post.date | date:"%Y-%m-%d" }}</span>
                            <a class="posts-list-name" href="{{ post.url }}">{{ post.title }}</a>
                        </li>
                        {% endfor %}
                    </ol>
                    {% endfor %}
                </section>
                <!-- /section.content -->
            </article>
        </div>
        <div class="column one-fourth">
            {% include sidebar-categories-nav.html %}
        </div>
    </div>
</section>


