# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Jekyll-based static site blog called "SLOG" (Life Hacks and Programming) hosted on GitHub Pages at `https://stevedoyle.github.io`. The site uses the Minimal Mistakes theme and focuses on programming content including Rust, C++, ESP32 development, and productivity topics.

## Development Commands

### Setup
```bash
# Install Ruby dependencies
bundle install

# Update dependencies
bundle update
```

### Local Development
```bash
# Serve site locally with live reload (default: http://localhost:4000)
bundle exec jekyll serve

# Serve with drafts included
bundle exec jekyll serve --drafts

# Build site without serving
bundle exec jekyll build

# Clean build artifacts
bundle exec jekyll clean

# Check site health
bundle exec jekyll doctor
```

## Content Creation

### Blog Posts
- Create new posts in `_posts/` with format: `YYYY-MM-DD-title.md`
- Use YAML front matter with required fields: `title`, `date`, optional: `tags`, `toc`
- Images go in `/assets/images/` with organized subdirectories per post

### Pages
- Static pages in `_pages/` directory
- Navigation configured in `_data/navigation.yml`

## Architecture

### Key Directories
- `_posts/`: Blog posts in Markdown format
- `_pages/`: Static pages (About, Archives, etc.)
- `_data/`: Structured data files (navigation, etc.)
- `assets/images/`: All images and diagrams
- `_site/`: Generated output (ignored by git)

### Theme and Configuration
- Uses `mmistakes/minimal-mistakes@4.27.1` remote theme
- Main configuration in `_config.yml`
- Pagination enabled via `jekyll-paginate-v2`
- SEO optimization via `jekyll-seo-tag`

### Deployment
- Automatic deployment via GitHub Pages on push to `main` branch
- Custom domain configured: `https://stevedoyle.github.io`
- No manual deployment steps required

## Content Guidelines
- Posts focus on programming (Rust, C++, embedded systems) and productivity
- Use code blocks with appropriate language highlighting
- Include table of contents for longer posts with `toc: true` in front matter
- Organize images in subdirectories matching post topics