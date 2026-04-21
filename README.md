# Trysil Blog

Source of the [Trysil Blog](https://trysil.lastrucci.net) — tutorials, deep dives,
and release notes for [Trysil](https://github.com/davidlastrucci/Trysil),
an ORM framework for Delphi.

Built with [Jekyll](https://jekyllrb.com/) and the
[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy/) theme.
Hosted on GitHub Pages.

## Writing a new post

Create a Markdown file under `_posts/` named `YYYY-MM-DD-slug.md` with front matter:

```yaml
---
title: "Post title"
date: 2026-04-20 12:00:00 +0100
categories: [Category]
tags: [trysil, delphi]
---
```

## Local preview

```shell
bundle install
bundle exec jekyll serve
```

## License

Content © David Lastrucci. Theme released under the [MIT License](LICENSE).
