# Renewelches Hugo Blog

A Hugo blog using the Clean White theme, configured with Hugo Modules.

## Theme

This blog uses the [Clean White theme](https://themes.gohugo.io/themes/hugo-theme-cleanwhite/) by Huabing Zhao.

## Prerequisites

- [Hugo Extended](https://gohugo.io/installation/) (v0.112.0 or later)
- [Go](https://golang.org/dl/) (v1.20 or later) - required for Hugo Modules

## Setup

1. Clone this repository:
   ```bash
   git clone <your-repo-url>
   cd renewelches-hugo-blog
   ```

2. Download the theme and dependencies:
   ```bash
   hugo mod get -u
   hugo mod tidy
   ```

3. Run the development server:
   ```bash
   hugo server -D
   ```

4. Visit `http://localhost:1313` in your browser

## Project Structure

```
renewelches-hugo-blog/
├── archetypes/          # Content templates
├── content/             # Blog posts and pages
│   ├── about/          # About section
│   ├── homelab/        # Homelab section
│   └── ai/             # AI section
├── layouts/             # Custom layout overrides
│   └── shortcodes/     # Custom shortcodes
├── static/              # Static assets (images, css, js)
├── hugo.toml           # Site configuration
├── go.mod              # Hugo modules configuration
└── README.md           # This file
```

## Creating Content

### New Blog Post

```bash
hugo new content/posts/my-post-title.md
```

### New Page

```bash
hugo new content/about.md
```

## Customization

### Layouts

Custom layouts and overrides go in the `layouts/` folder. These will override the theme's default layouts.

### Shortcodes

Custom shortcodes for reusable content snippets go in `layouts/shortcodes/`.

Example usage in markdown:
```markdown
{{< shortcode-name >}}
```

## Building for Production

```bash
hugo --minify
```

The built site will be in the `public/` directory.

## Deployment

Add your deployment instructions here based on your hosting platform (Netlify, Vercel, GitHub Pages, etc.).

## Hugo Modules

This site uses Hugo Modules instead of Git submodules for theme management. The theme is imported in `hugo.toml`:

```toml
[module]
[[module.imports]]
  path = 'github.com/zhaohuabing/hugo-theme-cleanwhite'
```

To update the theme:
```bash
hugo mod get -u
hugo mod tidy
```

## License

Your license here
