---
layout: post
title: "Setting Up a Hugo Blog with the Clean White Theme"
subtitle: "A step-by-step guide to creating a Hugo blog from scratch"
date: 2025-11-06 11:00:00
author: "Rene Welches"
image: "/img/2018-05-06-cryptocurrency_week1/bitcoin_header.jpg"
publishDate: 2025-11-06 11:00:00
tags:
  - Hugo
  - clean-white-theme
  - Tutorial
categories: [Hugo]
URL: "/2025/11/06/setting-up-clean-white-theme"
---

Want to create a beautiful blog with Hugo? This guide will walk you through setting up a Hugo site with the Clean White theme from scratch. We'll be using Hugo modules (the modern approach) instead of Git submodules.

## Prerequisites

Before we start, make sure you have:

- [Hugo installed](https://gohugo.io/installation/)
- [Git installed](https://git-scm.com/install/)
- [Go installed](https://go.dev/doc/install) (required for Hugo modules)
- A GitHub account

## Step 1: Create Your GitHub Repository

First, create a new repository on GitHub for your blog. You can name it something like `your-username-hugo-blog`. This repository will eventually be used to deploy your blog to Netlify (or GitHub Pages).

Clone your empty repository:

```bash
git clone git@github.com:your-username/your-hugo-blog.git
cd your-hugo-blog
```

## Step 2: Initialize Hugo Project

Initialize a new Hugo site in your current directory:

```bash
hugo new site . --force
```

The `--force` flag allows Hugo to create the site in an existing directory.

## Step 3: Initialize Go Modules

Since we're using Hugo modules for the theme, we need to initialize Go modules:

```bash
hugo mod init github.com/your-username/your-hugo-blog
```

Replace `your-username/your-hugo-blog` with your actual GitHub repository path.

## Step 4: Configure the Clean White Theme

We are going to import the theme as [go module](https://gohugo.io/hugo-modules/use-modules/) and not as git submodule.

Create or edit your `hugo.toml` (or `config.toml`) file to include the Clean White theme as a module:

```toml
baseURL = "https://your-domain.com/"
languageCode = "en-us"
title = "Your Blog Title"
theme = "hugo-theme-cleanwhite"

[module]
  [[module.imports]]
    path = "github.com/zhaohuabing/hugo-theme-cleanwhite"
```

Download the theme module:

```bash
hugo mod get -u
```

## Step 5: Create Essential Files

### .gitignore

Create a `.gitignore` file to exclude unnecessary files:

```
# Hugo build artifacts
.hugo_build.lock
# we don't want these in git and have netlify create them later
public/
resources/

# Operating system files
.DS_Store
Thumbs.db
```

### README.md

Create a `README.md` to document your project:

````markdown
# My Hugo Blog

A blog built with Hugo and the Clean White theme.

## Development

Run locally:

```bash
hugo server -D
```
````

Build for production:

```bash
hugo
```

````

## Step 6: Create Your First Blog Post

Create your first post:

```bash
hugo new post/my-first-post.md
````

Edit the created file in `content/post/my-first-post.md` with your content.

## Step 7: Run Your Site Locally

Start the Hugo development server:

```bash
hugo server -D
```

Visit `http://localhost:1313` in your browser to see your new blog!

## Understanding Clean White Theme Menus

One unique aspect of the Clean White theme is how it handles navigation menus. Unlike standard Hugo themes, Clean White has its own approach:

### Category-Based Menus

By default, Clean White automatically generates menu items based on your blog post categories. If you add `categories: [Hugo]` to your post's front matter, "Hugo" will appear in the navigation.

### Custom Menu Items

For additional menu items (like About, Contact, etc.), the Clean White theme uses a special parameter syntax:

```toml
[[params.additional_menus]]
    title = "ABOUT"
    href = "/about/"

[[params.additional_menus]]
    title = "CONTACT"
    href = "/contact/"
```

**Note:** The standard Hugo menu syntax (`[[menu.main]]`) doesn't work with this theme:

```toml
# This won't work with Clean White
[[menu.main]]
    name = "About"
    url = "/about/"
    weight = 1
```

## Next Steps

Now that your blog is set up, you can:

- Customize your `hugo.toml` with your personal information
- Add more blog posts
- Customize the theme's colors and styling
- Set up deployment to Netlify or GitHub Pages

Happy blogging!
