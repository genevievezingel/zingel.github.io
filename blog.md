---
layout: default
title: "Writing — Genevieve Zingel"
permalink: /blog/
---
<a class="brand-fixed" href="{{ '/' | relative_url }}">genevieve<span>.</span>zingel</a>
{% include theme-toggle.html %}

<main class="wrap blog-wrap">
  <a href="{{ '/' | relative_url }}" class="view-all blog-back">← Back to home</a>
  <h2 class="blog-title">Writing</h2>
  <p class="blog-intro">Notes on data engineering and software — pipelines, tooling, and the occasional hard-won lesson.</p>

  <div class="grid sm:grid-cols-2 gap-6">
    {% for post in site.posts %}
    <a class="thought" href="{{ post.url | relative_url }}">
      <div class="thought-meta">
        <span>{{ post.date | date: "%b %Y" }}</span>
        <span>{{ post.read }}</span>
      </div>
      <h3>{{ post.title }}</h3>
      <p>{{ post.description }}</p>
      <span class="read-more">Read more →</span>
    </a>
    {% endfor %}
  </div>
</main>

<style>
  .blog-wrap { max-width: 880px; padding-top: 96px; padding-bottom: 96px; }
  .blog-back { margin-bottom: 36px; }
  .blog-title { margin-bottom: 10px; }
  .blog-intro { color: var(--muted); font-size: 1.1rem; max-width: 560px; margin-bottom: 40px; }
</style>
