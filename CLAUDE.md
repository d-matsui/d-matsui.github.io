# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal website and blog for Daiki Matsui, built with Astro. Features a homepage with self-introduction, projects showcase, blog posts, and tag-based navigation.

## Pre-commit Checklist (CRITICAL)

Before committing ANY changes, you MUST run these commands to ensure CI passes:

1. `npm run lint` - Fix any linting errors (unused variables, etc.)
2. `npm run format` - Format code with Prettier
3. `npm run build` - Verify build succeeds

**CI will fail if any of these checks don't pass!**

## Key Commands

### Development
- `npm run dev` - Start local dev server at localhost:4321
- `npm run preview` - Preview production build locally
- `npm run sync` - Generate TypeScript types for Astro modules

### Build & Deploy
- `npm run build` - Type check, build site, generate search index
- Deploys automatically to GitHub Pages via GitHub Actions on push to main

### Code Quality
- `npm run lint` - ESLint check
- `npm run format` - Auto-format with Prettier
- `npm run format:check` - Check formatting without fixing

## Project Structure

### Content
- `src/data/blog/` - Blog posts in Markdown format
- `src/pages/` - Page routes (index, projects, posts, tags)
- `docs/reference/` - Reference documentation (not published)

### Configuration
- `src/config.ts` - Site configuration (title, author, features)
- `astro.config.ts` - Build configuration
- `.github/workflows/` - CI/CD pipelines (uses npm, not pnpm!)

### Key Features
- Homepage with integrated About section
- Projects showcase page
- Blog with tags and pagination
- Light/dark theme toggle
- Static search with Pagefind
- RSS feed at `/rss.xml`
- Dynamic OG images for posts

## Writing Blog Posts

Create new posts in `src/data/blog/` with this frontmatter:

```markdown
---
author: Daiki Matsui
pubDatetime: 2025-09-22T10:00:00Z
title: Post Title
slug: url-slug
featured: false
draft: false
tags:
  - tag1
  - tag2
description: Brief description for SEO
---
```

### IMPORTANT: Date Validation Checklist

**When CREATING new blog posts:**

1. **Use current date**: Run `date -Iseconds` to get the correct current timestamp
2. **Verify the date**: Double-check the year and month are correct (common mistake: writing `2025-01-XX` instead of `2025-09-XX`)
3. **Check before commit**: Ensure the date is not in the future and matches the intended publish date

**When EDITING existing blog posts:**

- **DO NOT change `pubDatetime`** - this is the original publication date and should remain unchanged
- Only update the content, not the publication date

## Site Navigation Structure

- **Home** (/) - Self-introduction and recent posts
- **Projects** (/projects) - Project showcase
- **Posts** (/posts) - All blog posts
- **Tags** (/tags) - Browse posts by tag

## Important Notes

- Site deploys to https://d-matsui.github.io/
- Archives feature is disabled (`showArchives: false`)
- No GitHub edit links on posts (`editPost.enabled: false`)
- Language is set to English (`lang: "en"`)
- Remember to run pre-commit checks before pushing!