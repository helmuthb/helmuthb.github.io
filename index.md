---
layout: default
title: Home
---

# Welcome to My Tech Blog

I'm a Solutions Architect specializing in Kubernetes, Cloud Computing, Machine Learning, and Generative AI. Here, I share my experiences, insights, and technical tutorials on these cutting-edge technologies.

## Latest Posts

{% for post in site.posts %}
* [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

## About Me

I work as a Solutions Architect, focusing on:
- Kubernetes and Container Orchestration
- Cloud Architecture and Infrastructure
- Machine Learning Operations (MLOps)
- Generative AI Implementation
- Cloud-Native Solutions

Feel free to follow me on [LinkedIn](https://linkedin.com/in/{{ site.linkedin_username }}) or check out my projects on [GitHub](https://github.com/{{ site.github_username }}).
