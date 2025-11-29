---
description: Create a new blog post with today's date
args:
  post_title:
    description: The title of the blog post
    required: true
---

Create a new blog post in the `_posts/` directory with today's date (2025-11-15) and the following title: "{{post_title}}".

The file should be named: `2025-11-15-{{post_title | slugify}}.md`

Use the following template for the post content:

```markdown
---
title: "{{post_title}}"
date: 2025-11-15
tags: []
toc: false
---

Write your content here.
```

After creating the file, confirm the file path to the user.
