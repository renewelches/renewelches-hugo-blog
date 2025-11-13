---
layout: post
title: "Part 2 - Deploying Your Hugo Blog to Netlify"
subtitle: "A complete guide to publishing your Hugo site from GitHub to Netlify"
date: 2025-11-07 11:00:00
author: "Rene Welches"
image: "/img/red-hook-crit.jpg"
publishDate: 2025-11-07 11:00:00
tags:
  - Hugo
  - Tutorial
  - Netlify
  - Deployment
categories: [Hugo]
URL: "/2025/11/07/deploy-to-netlify"
---

After building your Hugo blog locally, the next step is getting it online for the world to see. Netlify makes this process incredibly smooth with automatic deployments from GitHub. In this guide, I'll walk you through deploying your Hugo site to Netlify, including a crucial step: creating the `netlify.toml` configuration file.

## Why Netlify?

Netlify offers several advantages for hosting static sites like Hugo blogs:

- **Free tier** with generous limits
- **Automatic deployments** from Git commits
- **Built-in CDN** for fast global delivery
- **HTTPS** by default
- **Custom domains** support
- **Deploy previews** for pull requests

## Prerequisites

Before we begin, make sure you have:

- A Hugo site with content (see my [previous post on setting up Hugo](/2025/11/06/setting-up-clean-white-theme))
- Your Hugo site pushed to a GitHub repository
- A Netlify account (sign up at [netlify.com](https://www.netlify.com/))

## The Critical Step: Creating netlify.toml

Here's something that caught me by surprise: **without a `netlify.toml` file, Netlify will likely fail to build your Hugo site.** This configuration file tells Netlify exactly how to build your Hugo project.

Create a `netlify.toml` file in the root of your repository with the following content:

```toml
[build]
  publish = "public"
  command = "hugo --gc --minify"

[build.environment]
  HUGO_VERSION = "0.139.3"
  HUGO_ENV = "production"
  HUGO_ENABLEGITINFO = "true"

[context.production.environment]
  HUGO_VERSION = "0.139.3"
  HUGO_ENV = "production"

[context.deploy-preview]
  command = "hugo --gc --minify --buildFuture -b $DEPLOY_PRIME_URL"

[context.deploy-preview.environment]
  HUGO_VERSION = "0.139.3"

[context.branch-deploy]
  command = "hugo --gc --minify -b $DEPLOY_PRIME_URL"

[context.branch-deploy.environment]
  HUGO_VERSION = "0.139.3"

# Security headers
[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"

# Cache static assets for 1 year
[[headers]]
  for = "/css/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "/js/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "/img/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

### Understanding the Configuration

Let's break down what each section does:

#### Build Configuration

```toml
[build]
  publish = "public"
  command = "hugo --gc --minify"
```

- **`publish = "public"`**: Tells Netlify which directory contains your built site (Hugo outputs to `public/` by default)
- **`command = "hugo --gc --minify"`**: The build command that runs to generate your site
  - `--gc`: Runs garbage collection to clean up unused cache files
  - `--minify`: Compresses HTML, CSS, and JavaScript for faster loading

#### Environment Variables

```toml
[build.environment]
  HUGO_VERSION = "0.139.3"
  HUGO_ENV = "production"
  HUGO_ENABLEGITINFO = "true"
```

- **`HUGO_VERSION`**: Specifies which Hugo version to use (critical for build consistency)
- **`HUGO_ENV = "production"`**: Sets the environment to production (can affect theme behavior and analytics)
- **`HUGO_ENABLEGITINFO = "true"`**: Enables Git metadata like last modified dates from commits

#### Deploy Preview Configuration

```toml
[context.deploy-preview]
  command = "hugo --gc --minify --buildFuture -b $DEPLOY_PRIME_URL"
```

When you create a pull request, Netlify generates a preview deployment. This configuration:

- Uses `--buildFuture` to show posts with future dates in previews
- Sets `-b $DEPLOY_PRIME_URL` to use the correct base URL for the preview site

#### Branch Deploy Configuration

```toml
[context.branch-deploy]
  command = "hugo --gc --minify -b $DEPLOY_PRIME_URL"
```

Similar to deploy previews, but for non-production branches you want to deploy for testing.

#### Security Headers

```toml
[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
```

These headers add an extra layer of security to your site:

- **`X-Frame-Options = "DENY"`**: Prevents your site from being embedded in iframes (protects against clickjacking)
- **`X-XSS-Protection`**: Enables the browser's built-in XSS filter
- **`X-Content-Type-Options`**: Prevents MIME-sniffing attacks
- **`Referrer-Policy`**: Controls what referrer information is sent to other sites

#### Cache Headers

```toml
[[headers]]
  for = "/css/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"
```

Tells browsers to cache static assets (CSS, JS, images) for 1 year (31536000 seconds). The `immutable` flag means these files never change, so browsers don't need to revalidate them. This significantly improves page load times for returning visitors.

### Finding Your Hugo Version

To ensure compatibility, use the same Hugo version locally and on Netlify. Check your version with:

```bash
hugo version
```

You'll see output like:

```
hugo v0.139.3+extended darwin/arm64 BuildDate=unknown
```

Update the `HUGO_VERSION` in your `netlify.toml` to match this version number.

## Step-by-Step Deployment Guide

### Step 1: Prepare Your Repository

Make sure your `.gitignore` file excludes build artifacts:

```
# Hugo build artifacts
.hugo_build.lock
# We don't want these in git and have Netlify create them later
public/
resources/

# Operating system files
.DS_Store
Thumbs.db
```

### Step 2: Commit netlify.toml

Add the `netlify.toml` file to your repository:

```bash
git add netlify.toml
git commit -m "Add Netlify configuration for deployment"
git push origin main
```

### Step 3: Connect GitHub to Netlify

1. **Log in to Netlify**

   - Go to [app.netlify.com](https://app.netlify.com/)
   - Sign in with your GitHub account (recommended) or create a new account

2. **Import Your Project**

   - Click "Add new site" → "Import an existing project"
   - Choose "Deploy with GitHub"
   - Authorize Netlify to access your GitHub account if prompted

3. **Select Your Repository**
   - Find and select your Hugo blog repository from the list
   - Click on the repository name

### Step 4: Configure Site Settings

Since you have a `netlify.toml` file, Netlify will automatically detect most settings:

- **Site name**: Choose a custom subdomain (e.g., `my-awesome-blog.netlify.app`) or accept the auto-generated name
- **Branch to deploy**: `main` (or your default branch name)
- **Build command**: Should auto-populate as `hugo --gc --minify` (from netlify.toml)
- **Publish directory**: Should auto-populate as `public` (from netlify.toml)

Click **"Deploy site"** to start your first build.

### Step 5: Monitor the Build

You'll be redirected to your site's dashboard where you can:

- Watch the build logs in real-time under the "Deploys" tab
- See build progress and status
- View detailed error messages if the build fails

Common build errors include:

- **Incorrect Hugo version**: Update `HUGO_VERSION` in netlify.toml
- **Missing theme or modules**: Ensure `go.mod` and `go.sum` are committed
- **Content errors**: Check for invalid front matter or broken shortcodes

### Step 6: View Your Live Site

Once the build completes successfully (you'll see a green "Published" status):

- Your site is live at `https://your-site-name.netlify.app`
- Click the URL on the dashboard to view your deployed site
- Share the link with others

## Automatic Deployments

The beauty of this setup is that every time you push changes to GitHub, Netlify automatically:

1. Detects the new commits
2. Pulls the latest code
3. Runs the Hugo build command
4. Deploys the updated version
5. Invalidates the CDN cache

Your workflow becomes incredibly simple:

```bash
# Create or edit a blog post
hugo new post/my-new-post.md

# Preview locally
hugo server -D

# Commit and push
git add .
git commit -m "Add new blog post about Netlify deployment"
git push origin main

# Netlify automatically builds and deploys!
```

## Adding a Custom Domain

Want to use your own domain instead of `*.netlify.app`?

1. Go to **Site settings** → **Domain management**
2. Click **"Add custom domain"**
3. Enter your domain name (in my case, `blog.renewelches.com`)
4. Follow Netlify's instructions to setup you DNS.
   In my case I had to create a TXT record for the subdomain verfication. And then add CNAME record which mapped `blog.renewelches.com` to the netlify DNS `renewelches-hugo-blog.netlify.app`.

Netlify will automatically:

- Provision a free SSL certificate via Let's Encrypt
- Enable HTTPS for your custom domain
- Redirect HTTP to HTTPS

### Optional: Redirect Netlify Subdomain to Custom Domain

If you want to ensure all traffic uses your custom domain, add this to your `netlify.toml`:

```toml
[[redirects]]
  from = "https://renewelches-hugo-blog.netlify.app*"
  to = "https://blog.renewelches.com/:splat"
  status = 301
  force = true
```

This creates a permanent redirect (301) from your Netlify subdomain to your custom domain. The `:splat` preserves the URL path (e.g., `/about/` stays `/about/`), which is important for SEO.

## Deploy Previews for Pull Requests

One of Netlify's most powerful features is automatic deploy previews:

1. Create a new branch for your changes:

   ```bash
   git checkout -b new-feature
   ```

2. Make changes and push the branch:

   ```bash
   git push origin new-feature
   ```

3. Create a pull request on GitHub

4. Netlify automatically:
   - Builds a preview of your changes
   - Adds a comment to the PR with the preview URL
   - Updates the preview when you push new commits

This is perfect for:

- Reviewing blog posts before publishing
- Testing theme changes
- Sharing work-in-progress with others

## Troubleshooting Common Issues

### Build Fails with "Module not found"

If you're using Hugo modules for your theme, ensure `go.mod` and `go.sum` are committed:

```bash
git add go.mod go.sum
git commit -m "Add Go module files"
git push
```

### Different Appearance Than Local

Check that your `HUGO_VERSION` in netlify.toml matches your local version. Hugo updates can introduce breaking changes.

### Images Not Loading

Verify your image paths are correct and relative to your content. Hugo is case-sensitive, so `Image.jpg` and `image.jpg` are different files.

### Build Takes Too Long

If builds are slow:

- Remove `--gc` from the build command if you don't need garbage collection
- Check if you're generating too many pages (pagination, taxonomies)
- Consider using Hugo's caching features

## Monitoring and Analytics

Netlify provides built-in analytics:

- Go to **Analytics** in your site dashboard
- View page views, top pages, and traffic sources
- Upgrade to Netlify Analytics for more detailed insights (paid feature)

Alternatively, you can integrate Google Analytics, Plausible, or other analytics tools into your Hugo theme.

## Next Steps

Now that your blog is deployed:

- ✅ Set up automatic deployments from GitHub
- ✅ Configure security and cache headers
- ✅ Enable deploy previews for pull requests
- 🔲 Add a custom domain
- 🔲 Set up form handling with Netlify Forms
- 🔲 Configure Netlify Functions for dynamic features
- 🔲 Enable branch deployments for staging environments

## Conclusion

Deploying a Hugo site to Netlify is straightforward with the right configuration. The `netlify.toml` file is essential—it ensures Netlify knows exactly how to build your site and prevents frustrating build errors.

With automatic deployments, deploy previews, and a global CDN, Netlify makes publishing and maintaining your Hugo blog a breeze. Now you can focus on what matters most: creating great content!

Happy blogging!
