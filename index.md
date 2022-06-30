---
layout: default
list_title: Read My Latest Posts
title: ''
---

## Welcome to my blog

I write about tech stuffs I find interesting. It's possible you might find them interesting too. So, if you do, enjoy!

<div class="blog-index">  
  {% assign post = site.posts.first %}
  {% assign content = post.content %}
  {% include post_detail.html %}
</div>