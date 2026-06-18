# AGENTS.md

## Project Context

This is Jonathan Vazquez's personal GitHub project and portfolio site. It is an Astro-based static website deployed through GitHub Pages.

Use `FRONT.md` as the source of truth for current visual design rules before modifying UI, layout, typography, colors, buttons, animations, or responsive behavior.

Use `POST.md` as the source of truth for blog post structure, required frontmatter, post templates, and pre-publish validation before adding or editing posts.

Use `PROJECT.md` as the source of truth for project writeup structure, required frontmatter, project asset references, templates, and pre-publish validation before adding or editing projects.

Stay inside the project context:

- Treat this repository as a personal website, blog, and portfolio.
- Do not introduce backend services, databases, pipelines, or infrastructure unless the user explicitly asks for them.
- Keep changes aligned with the existing Astro, TypeScript, Markdown, and GitHub Pages setup.
- Prefer small, focused changes that preserve the current visual identity and content structure.

## Engineering Standards

Think like a backend software engineer and data engineer, even when working on frontend or content:

- Keep contracts clear: content schemas, route behavior, metadata, and generated paths should remain predictable.
- Validate changes through the available project scripts before considering work complete.
- Avoid hidden coupling between pages, layouts, content collections, and assets.
- Prefer explicit, maintainable logic over clever shortcuts.
- Keep naming, file placement, and component boundaries consistent with the existing codebase.

All code, identifiers, filenames, comments, and documentation added to the repository should be written in English, except when Spanish names are required by domain context or existing user-facing content.

## Harness And Validation Practices

When adding or changing behavior, use lightweight engineering harnesses where useful:

- Add focused validation scripts, fixtures, or checks only when they reduce real risk.
- Prefer deterministic checks over manual-only verification.
- For content-driven behavior, validate frontmatter schemas, generated routes, links, and build output.
- For UI behavior, verify responsive layout, accessibility-sensitive interactions, and client-side scripts.
- Keep harnesses close to the behavior they protect and document how to run them.

Do not add heavy test frameworks, CI services, or complex harness infrastructure unless the project need justifies it and the user agrees.

## Git And Branch Discipline

Be careful when integrating work:

- Always inspect the current branch and working tree before editing, staging, committing, merging, or rebasing.
- Do not overwrite, revert, or discard changes made by the user.
- Use a dedicated branch for non-trivial work. Prefer the `codex/` prefix unless the user requests another naming convention.
- Keep commits scoped and descriptive.
- Before integrating into `main`, confirm the intended branch strategy with the user.
- Never merge into `main`, force-push, or rewrite shared history without explicit user approval.

## Build And Runtime

This project requires Node.js `>=22.12.0`.

Before finishing code changes, run:

```sh
npm run build
```

If the system Node version is too old, use a compatible local runtime when available instead of changing project files just to satisfy the local shell.

## Collaboration Rules

- If a requested change modifies project structure, routing, deployment, data/content schemas, build behavior, or user-facing functionality, raise a clear warning first and ask for confirmation before editing files.
- Always ask the user whether they want to modify or add anything after completing a requested change.
- If a request would move outside the current personal-site scope, explain the tradeoff and ask before proceeding.
- Keep explanations concise and concrete.
