# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Jekyll-based personal blog hosted on GitHub Pages at https://martoc.github.io. The site covers topics including AWS, Kubernetes, AI, and Data.

## Build Commands

```bash
# Install dependencies
bundle install

# Build the site (outputs to ./_site)
bundle exec jekyll build

# Serve locally with live reload
bundle exec jekyll serve
```

## Architecture

- **Theme**: Uses `bulma-clean-theme` gem
- **Posts**: Markdown files in `_posts/` with format `YYYY-MM-DD-slug.md`
- **Pages**: Root-level markdown files (`index.md`, `about.md`)
- **Config**: `_config.yml` defines site settings, pagination (5 posts per page), and defaults

## Post Front Matter

Posts require YAML front matter:
```yaml
---
title: Post Title
subtitle: Optional subtitle
layout: post
author: martoc
image: https://martoc.github.io/blog/images/example.webp
---
```

## Deployment

Automated via GitHub Actions on push to `master` branch. The workflow (`.github/workflows/jekyll.yml`) builds with Jekyll and deploys to GitHub Pages.
