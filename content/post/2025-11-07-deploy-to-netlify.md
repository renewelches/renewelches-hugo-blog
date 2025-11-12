---
layout: post
title: "Deploy Hugo Blog to Netlify"
subtitle: "A step-by-step guide to creating a Hugo blog from scratch to Deployment"
date: 2025-11-07 11:00:00
author: "Rene Welches"
image: "/img/2018-05-06-cryptocurrency_week1/bitcoin_header.jpg"
publishDate: 2025-11-07 11:00:00
tags:
  - Hugo
  - Tutorial
  - Netlify
categories: [Hugo]
URL: "/2025/11/07/deploy-to-netlify"
---

## Netlify Config

The netlify.toml file configures how Netlify builds and deploys your Hugo blog. Let me break down each section:
Build Configuration

```[build]
publish = "public"
command = "hugo --gc --minify"
```

publish: Tells Netlify to deploy the public folder (where Hugo outputs your built site)
command: Runs Hugo with --gc (garbage collection to clean up unused cache) and --minify (compress HTML/CSS/JS for faster loading)
Environment Variables
[build.environment]
HUGO*VERSION = "0.152.2"
HUGO_ENV = "production"
HUGO_ENABLEGITINFO = "true"
HUGO_VERSION: Ensures Netlify uses the same Hugo version you're using locally
HUGO_ENV: Sets environment to "production" (can affect theme behavior and analytics)
HUGO_ENABLEGITINFO: Enables Git metadata like last modified dates from commits
Context-Specific Builds
Deploy Previews
[context.deploy-preview]
command = "hugo --gc --minify --buildFuture -b $DEPLOY_PRIME_URL"
When you open a pull request, Netlify creates a preview. The --buildFuture flag shows posts with future dates, and -b $DEPLOY_PRIME_URL sets the correct base URL for the preview.
Branch Deploys
[context.branch-deploy]
command = "hugo --gc --minify -b $DEPLOY_PRIME_URL"
Similar to previews but for non-production branches you want to deploy.
Security Headers
[[headers]]
for = "/*"
[headers.values]
X-Frame-Options = "DENY"
X-XSS-Protection = "1; mode=block"
...
Applies security headers to all pages:
X-Frame-Options: Prevents your site from being embedded in iframes (clickjacking protection)
X-XSS-Protection: Enables browser's XSS filter
X-Content-Type-Options: Prevents MIME-sniffing attacks
Referrer-Policy: Controls what referrer info is sent to other sites
Cache Headers
[[headers]]
for = "/\_.css"
[headers.values]
Cache-Control = "public, max-age=31536000, immutable"
Tells browsers to cache static assets (CSS, JS, images) for 1 year (31536000 seconds). The immutable flag means these files never change, so browsers don't need to revalidate them.
Redirects
[[redirects]]
from = "https://renewelches.netlify.app/*"
to = "https://renewelches.com/:splat"
status = 301
force = true
Permanently redirects (301) from your default Netlify subdomain to your custom domain. The :splat preserves the path (e.g., /about/ stays /about/). This helps with SEO by avoiding duplicate content.
