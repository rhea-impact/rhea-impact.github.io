---
layout: home
title: Home
---

# Navigating the AI Future

Welcome. I'm **Daniel Shanklin**, Executive Director of [Rhea Impact](https://github.com/rhea-impact), AGI researcher, and patented AI engineer.

This site exists because the AI conversation needs more signal and less noise. Here you'll find analysis grounded in technical reality—not hype, not fear, just clarity about what's actually happening and what it means.

---

## Latest Posts

{% for post in site.posts limit:5 %}
### [{{ post.title }}]({{ post.url | relative_url }})
<small>{{ post.date | date: "%B %-d, %Y" }}</small>

{{ post.excerpt }}

---
{% endfor %}

[View all posts →](/archive)

---

## About Rhea Impact

Rhea Impact is a Dallas–Fort Worth nonprofit exploring what happens when robots help the people who need them most. We focus on seniors living alone, single parents, and people with mobility limitations.

We also release [free, open-source software](https://github.com/rhea-impact) for coding agents and AI copilots.

[Learn more about our work →](/about)
