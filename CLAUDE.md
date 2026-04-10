# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A personal Hugo blog deployed to two targets:
- **Production** (Netlify): `main` branch → `https://blog.renewelches.com/`
- **Preview** (GitHub Pages): `preview` branch → GitHub Pages URL

The site uses the [cleanwhite theme](https://github.com/zhaohuabing/hugo-theme-cleanwhite) as a Hugo module (not a submodule). Hugo version: **0.152.2 extended**.

## Common Commands

```bash
# Download/update theme module (run once after cloning)
hugo mod get

# Local dev server (includes drafts)
hugo server -D

# Build for production
hugo --gc --minify

# Update the theme
hugo mod get -u github.com/zhaohuabing/hugo-theme-cleanwhite
hugo mod tidy
```

## Creating a New Post

Posts live in `content/post/` and follow the filename convention `YYYY-MM-DD-slug.md`.

Required frontmatter fields (use YAML front matter `---`, not TOML `+++`):

```yaml
---
layout: post
title: "Post Title"
subtitle: "Optional subtitle"
date: 2026-01-01
author: "Rene Welches"
description: "SEO description for the post"
image: "/img/banner-image.svg"
URL: "/YYYY/MM/DD/slug/"
tags:
  - Tag1
categories: [CategoryName]
---
```

Banner images go in `static/img/`. The `URL` field sets a clean permalink independent of the file name.

## Deployment

- Pushing to `preview` triggers the GitHub Actions workflow (`.github/workflows/*.yml`) which builds and deploys to GitHub Pages.
- Pushing to `main` triggers a Netlify build (configured in `netlify.toml`). The Netlify production command is `hugo --gc --minify`.
- The `public/` directory is the build output — it is committed to the repo and used by the preview deployment.

## Architecture Notes

- **Theme overrides**: Custom layouts go in `layouts/` (currently only `layouts/partials/`). Hugo's lookup order means files here take precedence over the theme.
- **No assets pipeline**: The `assets/` and `data/` directories are empty. No JS bundling or preprocessing is in use.
- **Taxonomies**: `tags`, `categories`, and `archives` are all active (configured in `hugo.toml`). Categories appear in the site menu automatically.
- **baseURL**: Set to `'/'` in `hugo.toml` for flexibility; the actual base URL is injected at build time via environment variables (`HUGO_BASEURL` in Netlify, `--baseURL` flag in the GitHub Actions workflow).
