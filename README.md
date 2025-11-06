# Rene Welches Hugo Blog

A personal blog built with Hugo, featuring content about HomeLab, AI, and technology.

## Project Setup

This Hugo site uses the [cleanwhite theme](https://github.com/zhaohuabing/hugo-theme-cleanwhite) as a Hugo module (not a Git submodule).

### Prerequisites

- [Hugo](https://gohugo.io/installation/) (extended version recommended)
- [Go](https://golang.org/dl/) 1.19 or later

### Installation

1. Clone the repository:
```bash
git clone https://github.com/renewelches/renewelches-hugo-blog-2.git
cd renewelches-hugo-blog-2
```

2. Download the Hugo theme module:
```bash
hugo mod get
```

3. Start the development server:
```bash
hugo server -D
```

The site will be available at `http://localhost:1313/`

### Project Structure

```
.
├── archetypes/          # Content templates
├── content/             # Markdown content files
├── data/                # Data files (JSON, YAML, TOML)
├── layouts/             # Custom layout overrides
├── static/              # Static files (images, CSS, JS)
├── hugo.toml            # Hugo configuration file
├── go.mod               # Go module file
└── go.sum               # Go module checksums
```

### Menu Structure

The site includes three main menu items:

- **HomeLab**: Content about home server and lab projects
- **AI**: Artificial intelligence related content
- **About**: Information about the author

### Creating Content

Create new posts for each section:

```bash
# HomeLab content
hugo new content/homelab/my-post.md

# AI content
hugo new content/ai/my-post.md

# About page
hugo new content/about/_index.md
```

### Building for Production

To build the static site for deployment:

```bash
hugo
```

The generated site will be in the `public/` directory.

### Theme Management

This project uses Hugo Modules for theme management. To update the theme:

```bash
hugo mod get -u github.com/zhaohuabing/hugo-theme-cleanwhite
hugo mod tidy
```

### Configuration

Main configuration is in `hugo.toml`. Key settings include:

- **baseURL**: Your site's base URL
- **title**: Site title
- **languageCode**: Default language
- **module.imports**: Theme module configuration
- **menu.main**: Top navigation menu items

## License

This project setup is provided as-is. Please refer to the theme's license for theme-specific terms.
