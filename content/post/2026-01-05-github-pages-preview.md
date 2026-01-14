---
layout: post
title: "Part 3 - Cost-Effective Blog Previews with GitHub Pages and Actions"
subtitle: "How I save Netlify credits by using GitHub Pages for preview deployments"
date: 2026-01-05 10:00:00
author: "Rene Welches"
description: "Save Netlify build credits by using GitHub Pages for preview deployments. Complete GitHub Actions workflow that deploys your preview branch to GitHub Pages while keeping production on Netlify's CDN."
image: "/img/red-hook-crit-1.jpg"
publishDate: 2026-01-05 10:00:00
tags:
  - Hugo
  - Tutorial
  - GitHub Pages
  - GitHub Actions
  - Netlify
  - DevOps
categories: [Hugo]
URL: "/2026/01/05/github-pages-preview"
---

When I first set up my Hugo blog on Netlify, I loved the automatic deploy preview feature for pull requests. However, I quickly realized that each preview deployment consumes free tier credits (currently 300 credits/month). With frequent updates and iterations, I was burning through my monthly allowance faster than expected.

Rather than upgrading to a paid plan for something I only needed once in a while, I implemented a dual-deployment strategy: **GitHub Pages for previews, Netlify for production**. This approach gives me unlimited preview deployments while keeping my production site on Netlify's excellent CDN.

## The Problem with Netlify Previews

Netlify's free tier includes:

- 300 build minutes per month
- Automatic deploy previews for pull requests
- Branch deployments

While generous, these limits can be consumed quickly when you're:

- Testing multiple design iterations
- Reviewing blog posts in different stages
- Experimenting with theme changes
- Making frequent small updates

Every preview build counts against your monthly quota. For a personal blog with frequent updates, this can become a limitation.

## The Solution: A Dual-Deployment Strategy

Here's the workflow I implemented:

1. **Preview Branch → GitHub Pages**: Any changes pushed to the `preview` branch automatically deploy to GitHub Pages via GitHub Actions
2. **Main Branch → Netlify**: Production deployments continue using Netlify's automatic deployment from the `main` branch

This gives me:

- **Unlimited preview builds** (GitHub Actions provides 2,000 free minutes per month for public repositories)
- **Cost-effective testing environment** without consuming Netlify credits
- **Production-grade hosting** for my main site via Netlify's CDN
- **Simple workflow** with automatic deployments for both environments

## Setting Up GitHub Pages Deployment

### Step 1: Enable GitHub Pages

First, enable GitHub Pages for your repository:

1. Go to your repository on GitHub
2. Navigate to **Settings** → **Pages**
3. Under "Build and deployment", select **Source**: "GitHub Actions"
4. Save your changes

### Step 2: Create the GitHub Actions Workflow

Create a workflow file at `.github/workflows/hugo.yml`:

```yaml
# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the preview branch
  push:
    branches: ["preview"]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.152.2
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Install Dart Sass
        run: sudo snap install dart-sass

      - name: Checkout
        uses: actions/checkout@v5
        with:
          submodules: recursive
          ref: preview

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"

      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: preview
          HUGO_TITLE: "PREVIEW René Welches"
        run: |
          hugo \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### Understanding the Workflow

Let's break down the key components:

#### Trigger Configuration

```yaml
on:
  push:
    branches: ["preview"]
  workflow_dispatch:
```

The workflow triggers on:

- **Pushes to the `preview` branch**: Automatic deployment when you push changes
- **Manual dispatch**: Ability to trigger the workflow manually from the Actions tab

#### Permissions

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

These permissions allow the workflow to:

- Read your repository contents
- Deploy to GitHub Pages
- Use OpenID Connect for secure deployment

#### Concurrency Control

```yaml
concurrency:
  group: "pages"
  cancel-in-progress: false
```

Ensures only one deployment runs at a time, but doesn't cancel in-progress deployments. This prevents race conditions while allowing queued deployments to complete.

#### Build Job Steps

**1. Install Hugo CLI**

```yaml
- name: Install Hugo CLI
  run: |
    wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
    && sudo dpkg -i ${{ runner.temp }}/hugo.deb
```

Downloads and installs Hugo Extended (required for SCSS/Sass processing). The version is controlled by the `HUGO_VERSION` environment variable.

**2. Install Dart Sass**

```yaml
- name: Install Dart Sass
  run: sudo snap install dart-sass
```

Many Hugo themes require Sass for styling. This step installs Dart Sass, the modern implementation of Sass.

**3. Checkout Code**

```yaml
- name: Checkout
  uses: actions/checkout@v5
  with:
    submodules: recursive
    ref: preview
```

Checks out the `preview` branch with all Git submodules (important if your theme is a submodule).

**4. Build with Hugo**

```yaml
- name: Build with Hugo
  env:
    HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
    HUGO_ENVIRONMENT: preview
    HUGO_TITLE: "PREVIEW René Welches"
  run: |
    hugo \
      --minify \
      --baseURL "${{ steps.pages.outputs.base_url }}/"
```

The build step includes several important configurations:

- **`HUGO_ENVIRONMENT: preview`**: Sets the environment to "preview", which you can use in your templates to add preview-specific banners or warnings
- **`HUGO_TITLE: "PREVIEW René Welches"`**: Overrides the site title to clearly indicate this is a preview site
- **`--baseURL`**: Dynamically sets the base URL to match your GitHub Pages URL
- **`--minify`**: Optimizes HTML, CSS, and JavaScript for faster loading

**5. Upload and Deploy**

```yaml
- name: Upload artifact
  uses: actions/upload-pages-artifact@v3
  with:
    path: ./public
```

The built site is uploaded as an artifact and then deployed to GitHub Pages in the separate deployment job.

## The Complete Workflow

### For Preview Changes:

1. Create or edit content in your repository
2. Push changes to the `preview` branch:

   ```bash
   git checkout preview
   git add .
   git commit -m "Preview: New blog post about GitHub Actions"
   git push origin preview
   ```

3. GitHub Actions automatically:
   - Builds your Hugo site
   - Deploys to GitHub Pages
   - Makes it available at `https://yourusername.github.io/your-repo/`

4. Review your changes on the preview site
5. When satisfied, merge to `main`:

   ```bash
   git checkout main
   git merge preview
   git push origin main
   ```

### For Production Deployment:

When you push to `main`, Netlify automatically:

1. Detects the new commits
2. Builds the production site
3. Deploys to your custom domain
4. Invalidates the CDN cache

## Configuring Netlify to Only Deploy Main

To ensure Netlify only deploys your `main` branch, update your `netlify.toml`:

```toml
[build]
  publish = "public"
  command = "hugo --gc --minify"

[build.environment]
  HUGO_VERSION = "0.152.2"
  HUGO_ENV = "production"

# Only deploy the main branch
[context.production]
  branch = "main"
```

Alternatively, configure this in Netlify's dashboard:

1. Go to **Site settings** → **Build & deploy**
2. Under **Deploy contexts**, set:
   - **Production branch**: `main`
   - **Branch deploys**: None
   - **Deploy previews**: None

This ensures Netlify ignores all other branches, including your `preview` branch.

## Benefits of This Approach

### Cost Savings

- **Netlify credits preserved**: Only production builds count against your quota
- **GitHub Actions free tier**: 2,000 minutes/month for public repositories
- **No upgrade needed**: Stay on free tiers for both services

### Flexibility

- **Unlimited iterations**: Test as many changes as you want
- **No quota anxiety**: Make frequent small commits without worrying about build minutes
- **Separate environments**: Clear distinction between preview and production

### Simplicity

- **Automatic deployments**: Both environments deploy automatically
- **No manual steps**: Push to branch, deployment happens
- **Standard Git workflow**: Uses familiar branching and merging

## Potential Drawbacks

While this approach works well, there are some trade-offs:

1. **Different URLs**: Preview and production use different domains (GitHub Pages vs. Netlify)
2. **No PR previews**: Unlike Netlify's automatic PR previews, you need to push to the preview branch
3. **Manual branch management**: You need to remember to push to `preview` first, then merge to `main`

For my use case, these trade-offs are worth the cost savings and flexibility and I also got a first dip into Github Actions.

## Alternative: Branch Protection + Review Workflow

You can combine this with GitHub's branch protection to create a more robust workflow:

1. **Create feature branches** for new content:

   ```bash
   git checkout -b new-blog-post
   # Make changes
   git push origin new-blog-post
   ```

2. **Create a PR to preview branch** for initial review
3. **Merge to preview** to see it on GitHub Pages
4. **Create a PR from preview to main** for final review
5. **Merge to main** to deploy to production

This gives you a two-stage review process with automatic deployments at each stage. I did not implement it, as I am the only one writing blog posts and it would have been overkill for my use case.

## Monitoring Your Preview Site

### Viewing Deployment Status

Monitor your deployments in GitHub:

1. Go to the **Actions** tab in your repository
2. Click on the latest workflow run
3. View detailed logs for each step
4. Check deployment status and any errors

### Finding Your Preview URL

Your preview site will be available at:

```
https://yourusername.github.io/your-repo-name/
```

For example, mine is at:

```
https://renewelches.github.io/renewelches-hugo-blog/
```

## Tips and Best Practices

### 1. Add Preview Information to your Site

Update your Hugo templates to show a preview banner when `HUGO_ENVIRONMENT` is "preview":

```html
{{ if eq (getenv "HUGO_ENVIRONMENT") "preview" }}
<div class="preview-banner">
  ⚠️ This is a preview site. Visit <a href="https://blog.renewelches.com">blog.renewelches.com</a> for the production site.
</div>
{{ end }}
```

Or add PREVIEW to the title and set it as variable in your github action (that's what I did)

```yaml
...
- name: Build with Hugo
  env:
    HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
    HUGO_ENVIRONMENT: preview
    HUGO_TITLE: "PREVIEW René Welches"
...
```

### 2. Use Environment-Specific Configuration

Create a `config/preview/config.toml` for preview-specific settings:

```toml
title = "PREVIEW - René Welches"
[params]
  environment = "preview"
```

### 3. Keep Your Hugo Versions in Sync

Use the same Hugo version across:

- Your local development environment
- GitHub Actions workflow (`HUGO_VERSION` env var)
- Netlify configuration (`netlify.toml`)

### 4. Monitor Build Times

Keep an eye on your build times in GitHub Actions. If builds are slow:

- Remove unnecessary build steps
- Use Hugo's caching features
- Consider caching Node.js dependencies

### 5. Regular Cleanup

Periodically clean up your preview branch:

```bash
git checkout preview
git fetch origin main
git reset --hard origin/main
git push origin preview --force
```

This keeps your preview branch in sync with production and prevents drift.

## Conclusion

By using GitHub Pages for preview deployments and Netlify for production, I've created a cost-effective workflow that gives me unlimited preview builds without consuming my Netlify credits. The setup is straightforward with GitHub Actions, and the dual-deployment strategy provides clear separation between preview and production environments.

This approach is perfect for:

- Personal blogs with frequent updates
- Projects with limited budgets
- Teams that need extensive preview testing
- Anyone wanting to maximize free tier benefits

The initial setup takes about 15 minutes, but the long-term savings and flexibility make it worthwhile. Now I can iterate freely on blog posts and design changes without worrying about depleting my Netlify credits.

Happy blogging!

## Resources

- [GitHub Pages Documentation](https://docs.github.com/en/pages)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)
- [Hugo Environment Variables](https://gohugo.io/getting-started/configuration/#configure-with-environment-variables)
- [My previous post on deploying to Netlify](/2025/11/07/deploy-to-netlify)
