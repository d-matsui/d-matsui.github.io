# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an AstroPaper blog site - a minimal, responsive, accessible and SEO-friendly Astro blog theme built with TypeScript and TailwindCSS.

## Key Commands

### Development
- `npm run dev` - Starts local dev server at localhost:4321
- `npm run build` - Type checks, builds site, generates pagefind search index, and copies to public folder
- `npm run preview` - Preview production build locally

### Code Quality
- `npm run format` - Format code with Prettier
- `npm run format:check` - Check code formatting
- `npm run lint` - Lint with ESLint
- `npm run sync` - Generate TypeScript types for Astro modules

### Docker
- `docker compose up -d` - Run AstroPaper in Docker
- `docker build -t astropaper .` - Build Docker image
- `docker run -p 4321:80 astropaper` - Run container

## Architecture and Structure

### Content Management
- Blog posts are stored in `src/data/blog/` as Markdown files
- Content is managed via Astro's Content Collections API defined in `src/content.config.ts`
- Posts support frontmatter with fields like title, pubDatetime, tags, draft status, etc.

### Core Configuration
- `src/config.ts` - Main site configuration (title, author, URLs, features)
- `astro.config.ts` - Astro build configuration with integrations (sitemap, TailwindCSS)
- Posts are paginated with configurable `postPerPage` and `postPerIndex` settings

### Key Features
- Dynamic OG image generation for posts at `/posts/[slug]/index.png`
- Pagefind static search with UI auto-generated at build time
- Light/dark theme toggle with system preference detection
- Sitemap and RSS feed generation
- Draft posts support (hidden in production)
- Optional Google Site Verification via environment variable

### Styling and UI
- TailwindCSS v4 with typography plugin
- Responsive design from mobile to desktop
- Custom color schemes configurable in `src/styles/base.css`
- Icons from Tabler Icons library

### Key Directories
- `src/pages/` - Astro page routes
- `src/layouts/` - Layout components
- `src/components/` - Reusable UI components
- `src/utils/` - Helper functions (post sorting, tag extraction, OG image generation)
- `public/` - Static assets served directly
- `public/pagefind/` - Auto-generated search index (created during build)

## Important Notes

- Environment variable `PUBLIC_GOOGLE_SITE_VERIFICATION` can be set for Google Search Console
- The site uses Astro's Image optimization with responsive layout
- Markdown files support Remark plugins for TOC and collapsible sections
- Code blocks support syntax highlighting with Shiki and custom transformers