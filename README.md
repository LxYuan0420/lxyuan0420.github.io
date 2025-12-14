## lxyuan0420.github.io

Personal notes / knowledge dump published via GitHub Pages.

### Theme

- `remote_theme: b2a3e8/jekyll-theme-console`
- `style: light`

### Publishing

Publish by pushing to the branch GitHub Pages builds from (Settings → Pages).

1. Write Markdown.
2. `git commit`
3. `git push`

### New post

Create a Markdown file under `_posts/`:

`_posts/YYYY-MM-DD-your-title.md`

Template:

```md
---
layout: post
title: "TIL: ..."
description: "One-line summary"
---

Write here.
```

Code blocks:

````md
```bash
echo "hello"
```
````

### Images

- Put images under `assets/images/` (e.g. `assets/images/documentation/foo.png`).
- Reference them from posts using either:
  - Markdown: `![alt text](/assets/images/documentation/foo.png)`
  - Or the helper include: `{% include image.html path="documentation/foo.png" alt="alt text" %}`
