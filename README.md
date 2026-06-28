# Starlight

A documentation site built with [Starlight](https://starlight.astro.build/), the docs framework for [Astro](https://astro.build/).

> Project named in tribute to its creators — the name fit too well to change.

## Requirements

- Node.js 22+ (even-numbered versions only)

## Getting Started

```bash
# Install dependencies
npm install

# Start the dev server (http://localhost:4321)
npm run dev

# Build for production (outputs to ./dist)
npm run build

# Preview the production build locally
npm run preview
```

## Project Structure

```
.
├── src/
│   ├── content/
│   │   └── docs/          # Markdown/MDX pages (file-based routing)
│   └── content.config.ts  # Content collection schema
├── astro.config.mjs       # Starlight config (sidebar, theme, etc.)
└── package.json
```

## Adding Content

Drop a `.md` or `.mdx` file into `src/content/docs/` with frontmatter:

```md
---
title: My Page
description: A short description.
---

Your content here.
```

Every file becomes a page automatically. `index.mdx` is the homepage.

## Configuration

Site-wide settings (title, sidebar, social links, theme) live in `astro.config.mjs`. See the [Starlight configuration reference](https://starlight.astro.build/reference/configuration/).
