# FRONT.md

## Design Context

This project is a personal portfolio and blog with a quiet editorial style. The interface should feel personal, readable, technical, and lightweight.

Keep the current design direction:

- White background, black text, restrained accents, and generous whitespace.
- Editorial layouts with illustration support, not dashboard or SaaS-style surfaces.
- Simple navigation, content-first pages, and readable long-form content.
- Minimal decoration. Avoid gradients, decorative blobs, heavy shadows, or noisy backgrounds.

## Typography

The current font system is defined in `astro.config.mjs` and loaded through `BaseHead.astro`.

- Page titles and strong UI labels use `var(--font-nunito), sans-serif`.
- Descriptions, subtitles, metadata, body copy in feature sections, and content-heavy UI use `var(--font-typewriter), ui-monospace, monospace`.
- Global fallback body typography uses `var(--font-atkinson)`.

Title rules:

- Use `Nunito` for `h1`, section titles, card/index titles, navigation labels, and CTA labels.
- Use heavy weights, usually `700` or `800`.
- Keep `letter-spacing: 0` unless matching an existing local pattern.
- Use `clamp()` for responsive page-level headings.
- Do not scale font size directly with viewport width outside controlled `clamp()` values.

Description rules:

- Use `Courier Prime` through `--font-typewriter`.
- Keep descriptions compact, calm, and readable.
- Typical description sizes are `0.78rem` to `0.9rem`.
- Use line-height around `1.55` to `1.85`.

## Color System

Core colors:

- Background: `#ffffff` or `#fff`.
- Primary text: `#111`.
- Body text: `#222`, `#333`, or `rgb(var(--gray-dark))`.
- Muted text: `#444240`, `#555`, `#777`, `#b3afa9`.
- Accent: `#4e8d86`.
- Accent dark: `#3a6963`.
- Subtle borders: `#e6e4e1`.
- Soft code/background blocks: `#f8f7f5`.

Use the existing CSS variables when possible:

- `--accent`
- `--accent-dark`
- `--black`
- `--gray`
- `--gray-light`
- `--gray-dark`

Avoid introducing a new dominant palette unless the user explicitly asks for a redesign.

## Buttons And Links

Primary buttons:

- Use black background: `#111`.
- Use white text: `#fff` or `#FFFFFF`.
- Use pill radius: `border-radius: 999px`.
- Use subtle hover movement: `transform: translateY(-2px)`.
- Hover background can move to `#000`.

Index/list links:

- Keep links mostly text-based.
- Use horizontal movement on hover: `transform: translateX(10px)`.
- Keep transitions short and subtle.

Navigation links:

- Use `Nunito`, weight `800`, around `0.86rem`.
- Active and hover states use a thin underline implemented with `::after`.
- Do not turn navigation into boxed buttons.

## Layout And Containers

Main wide pages use:

```css
width: min(1200px, 100% - 3em);
max-width: none;
margin-inline: auto;
```

Long-form default content can use the global narrower main width:

```css
width: 720px;
max-width: calc(100% - 2em);
```

Layout patterns:

- Home, blog index, and projects index use strong two-column layouts on desktop.
- Blog and project detail pages use a left summary column and right content column.
- About uses wide editorial sections with timeline, detail blocks, skill pills, CV, and contact sections.
- Mobile breakpoints generally collapse at `720px`.
- Mobile layouts should become single-column and allow scrolling.

Avoid nested card layouts. Use sections, grids, lists, and figures instead.

## Images And Assets

Current visual assets are SVG/PNG/JPG files under `src/assets`.

- Use real project images or existing illustrations when possible.
- Keep image corners at `8px` when framed.
- Decorative support images should not overpower content.
- Detail page visuals should remain inspectable and clean.

## Motion

Motion should be subtle and functional.

Current animation patterns:

- Page enter: fade in around `180ms`.
- Section/page content: fade up around `0.5s`.
- View transitions: around `220ms`.
- Image viewer overlay: `180ms` to `220ms`.
- Link hover movement: `0.2s` to `0.28s`.

Always support reduced motion:

```css
@media (prefers-reduced-motion: reduce) {
	/* Disable non-essential animation here. */
}
```

Do not add large, looping, or attention-heavy animations unless requested.

## Content UI Patterns

Blog and project indexes:

- Ordered index lists with small muted numbers.
- Thin dividers between entries.
- `Nunito` entry titles.
- Typewriter descriptions.
- Optional tool/stack pills for projects.

Skill and tool pills:

- Use pill radius.
- Use typewriter typography.
- Keep colors semantic and restrained.
- Prefer existing tool color mappings for Astro, TypeScript, Markdown, Python, SQL, and Databricks.

Code and prose:

- Inline code uses muted gray backgrounds and small radius.
- Code blocks use dark backgrounds on detail pages.
- Prose should remain narrow enough for comfortable reading.

## Change Control

Before modifying the visual system, warn the user and ask for confirmation if the change affects:

- Page structure or layout model.
- Typography system.
- Global colors or button styles.
- Navigation behavior.
- Animations or transitions.
- Responsive breakpoints.
- Shared layouts or reusable components.

Small content-only edits can follow the existing style without a warning, but design-system changes need explicit approval.
