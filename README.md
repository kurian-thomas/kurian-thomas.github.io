# Nova's Blog

## Where to add posts

Add all blog posts under `content/posts/`.

- Top-level post example: `content/posts/my-first-post.md`
- Nested post example: `content/posts/linux/arch/hyprland-notes.md`

The homepage listing includes posts from `content/posts/` and any nested subdirectories, and renders them in one unified list.

## Post front matter

Use TOML front matter like:

```toml
+++
title = "My Post"
date = 2026-07-25T18:00:00-07:00
draft = false
group = "ungrouped"
+++
```

If `group` is omitted, templates default it to `ungrouped`.
