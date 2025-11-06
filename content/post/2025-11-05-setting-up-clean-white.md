---
layout:     post
title:      "Setting Up The Clean White Theme"
subtitle:   ""
date:       2025-11-06 11:00:00
author:     "Rene Welches"
image: "/img/2018-05-06-cryptocurrency_week1/bitcoin_header.jpg"
publishDate: 2025-11-06 11:00:00
tags:
    - Hugo 
    - clean-white-theme
categories: [ Hugo ]
URL: "/2025/11/06/setting-up-clean-white-theme"

---

> This series of articles about hugo and setting up the clean white theme.

## Table of Content 
{:.no_toc}

* Table of Content
{:toc}

## Create an empty project with Claude Code
I started playing around with claude code and my first project is obviously this blog. It took me several attempts to come up with a somewhat right prompt. First I created an emtpy project on github. Which will be later used to deploy the blog to netlify.

```bash src/content/post/2025-11-05-setting-up-clean-white.md
# Initialize a new Hugo project
git clone git@github.com:renewelches/renewelches-hugo-blog.git
cd renewelches-hugo-blog

#start claude code
claude
```
Then you get presented with the following screen. Claude code is now running within your project folder. 

```
╭─── Claude Code v2.0.32 ───────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                    │ Tips for getting started                                                     │
│                 Welcome back René!                 │ Run /init to create a CLAUDE.md file with instructions for Claude            │
│                                                    │ Run /install-github-app to tag @claude right from your Github issues and PRs │
│                       ▐▛███▜▌                      │ ──────────────────────────────────────────────────────────────────────────── │
│                      ▝▜█████▛▘                     │ Recent activity                                                              │
│                        ▘▘ ▝▝                       │ No recent activity                                                           │
│                                                    │                                                                              │
│               Sonnet 4.5 · Claude Pro              │                                                                              │
│   /Users/rene/Workspace/renewelches-hugo-blog.bak  │                                                                              │
╰───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
> Try "edit <filepath> to..."
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  ? for shortcuts
```

After several attempts I settled on the following prompt.

```
Create a new Hugo project with the template from github.com/zhaohuabing/hugo-theme-cleanwhite as a module,not a GitHub submodule. 
Initialize the project with GitHub.com/renewelches/renewelches-hugo-blog. 
Include a README file explaining the project setup. 
Add .hugo_build.lock, .DS_Store, and the public directory to .gitignore.
```
The `Initialize the project ...` prompt is necessary, otherwise claude will configure the theme as submodule in the hugo.toml but it is not getting applied properly.

I also tried adding the creation of menu items to the prompt. Claude did a good job in adding the menu configurations to the hugo.toml and the folder and files to the content folder. Unfortunately the clean-white-template did ignore them. By default it renders menu items based on blog post categories. Addiotionallyyou can work with params for additional menu items. 
```
 [[params.additional_menus]]
    title = "ABOUT"
    href = "/about/"
```
vs. the standard way of defining menu items
```
  [[menu.main]]
    name = "About"
    url = "/about/"
    weight = 1
```




