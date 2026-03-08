---
layout: default
title: "LinksEnabled"
---

I ask DeepSeek how to make links to my posts available on the [main page](https://qp91mn64.github.io) instead of adding each link in README.md manually.

DeepSeek says that an `index.md` or `index.html` is required, and the `index.md` may look like this (AI generated):

```
---
layout: default
title: 我的博客
---
# 最新文章
{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
```

So I add an `index.md` into my repository like this, a little simplifier:

```
---
layout: default
---
{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
```

Commit and push the change, refresh the page, and now **the links does appear**.

Even the link of post without front matter (2026-03-07-HelloWorld.md) appears.

Now it is possible to update more posts, write new posts, etc., and the links will appear on the main page.
