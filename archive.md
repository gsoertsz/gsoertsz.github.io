---
layout: default
---

{% assign dates = site.posts | map: "date" %}
{% assign years = '' | split: '' %}
{% for d in dates %}
    {% assign year = d | date: "%Y" | split: '_' %}
    {% assign years = years | concat: year %}
{% endfor %}

{% assign unique_years = years | uniq %}

<div class="container">
    {% for y in unique_years %}
        {% assign this_years_posts = site.posts | where: "date", y %}
        <div class="row pt-4">
            <div class="col-2 mt-2">
                <p>{{ y }}</p>
            </div>
            <div class="col">
            {% for p in this_years_posts %}
                <div class="container">
                    <div class="row mt-2">
                        <div class="col-2">
                            <span class="post-date">{{ p.date | date: "%b %-d" }}</span>
                        </div>
                        <div class="col-10">
                            <a class="post-link" href="{{ p.url | relative_url }}">{{ p.title }}</a>
                        </div>
                    </div>
                </div>
            {% endfor %}
            </div>
        </div>
        <hr class="mt-5 mb-1"/>
     {% endfor %}
</div>

<ul class="posts">
  
</ul>