# Jonathan Vazquez Personal Site

Personal portfolio and blog built with Astro and deployed to GitHub Pages.

The site is designed as a lightweight technical space for writing, project notes, career context, and engineering decisions. It should stay focused on static content, clear navigation, fast builds, and maintainable content contracts.

## Stack

- Astro 6
- TypeScript
- Markdown and MDX content collections
- Astro assets pipeline
- RSS feed
- Sitemap generation
- GitHub Pages deployment

## Project Structure

```text
├── public/
│   ├── favicon.ico
│   ├── favicon.svg
│   └── jonathan-vazquez-cv.pdf
├── src/
│   ├── assets/
│   ├── components/
│   ├── content/
│   │   ├── blog/
│   │   └── projects/
│   ├── layouts/
│   ├── pages/
│   ├── consts.ts
│   ├── content.config.ts
│   └── styles/
├── astro.config.mjs
├── AGENTS.md
├── FRONT.md
├── package.json
└── README.md
```

## Content Model

Content is managed through Astro content collections in `src/content.config.ts`.

Blog posts live in `src/content/blog` and require:

- `title`
- `description`
- `pubDate`
- optional `updatedDate`
- optional `heroImage`

Project writeups live in `src/content/projects` and require:

- `title`
- `description`
- `promoImage`
- `githubUrl`
- optional `pubDate`
- optional `stack`

Keep frontmatter valid and predictable. These schemas are the contract between Markdown content, generated routes, SEO metadata, RSS, and project/blog index pages.

## Main Routes

- `/` - Home page
- `/about` - Professional profile and contact context
- `/blog` - Blog index
- `/blog/[slug]` - Blog detail pages
- `/projects` - Project index
- `/projects/[slug]` - Project detail pages
- `/rss.xml` - RSS feed

## Design Rules

Before changing UI, layout, typography, colors, buttons, animations, or responsive behavior, read `FRONT.md`.

The current design direction is editorial, minimal, and content-first:

- Wide `1200px` page containers for main views
- White background with black text and restrained teal accents
- `Nunito` for headings and UI labels
- `Courier Prime` for descriptions, metadata, and content-heavy UI
- Subtle transitions and reduced-motion support

## Engineering Rules

Before changing project behavior or integration flow, read `AGENTS.md`.

Important project rules:

- Stay within the personal-site context.
- Do not introduce backend services, databases, pipelines, or infrastructure unless explicitly requested.
- Raise a warning and ask for confirmation before changes that affect structure, routing, deployment, schemas, build behavior, or user-facing functionality.
- Inspect branch and working tree state before staging, committing, merging, or integrating.
- Do not merge into `main`, force-push, or rewrite shared history without explicit approval.

## Development

This project requires Node.js `>=22.12.0`.

Install dependencies:

```sh
npm install
```

Start the local development server:

```sh
npm run dev
```

Build the static site:

```sh
npm run build
```

Preview the production build:

```sh
npm run preview
```

Run Astro CLI commands:

```sh
npm run astro -- --help
```

## Validation

Before finishing code or content-contract changes, run:

```sh
npm run build
```

The build validates content schemas, generated routes, optimized assets, RSS output, and sitemap generation.

If the local shell uses an older Node version, switch to a compatible Node runtime instead of changing project files only to satisfy the local environment.

## Deployment

Deployment is handled by GitHub Actions in `.github/workflows/deploy.yml`.

The workflow:

- Runs on pushes to `main`
- Supports manual runs through `workflow_dispatch`
- Uses `withastro/action@v3`
- Deploys the static output to GitHub Pages

Keep deployment changes scoped and review branch strategy before integrating anything into `main`.
