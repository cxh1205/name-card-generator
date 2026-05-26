# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install          # install dependencies
npm run dev          # start dev server (Vite HMR)
npm run build        # production build
npm run preview      # preview production build locally
```

No tests, linter, or CI are configured.

## Architecture

Single-page Vue 3 (Composition API, `<script setup>`) app built with Vite. The entire application lives in one file: `src/App.vue` (no router, no state management, no child components).

### Screen vs Print dual rendering

The same DOM produces two different visual outputs via CSS strategy:

**Screen (preview):**
- Cards laid out in a CSS Grid with `repeat(N, minmax(0, 1fr))` — column count (1/2/3) set by user
- Each `.card` has `aspect-ratio: 1` and `width: 100%`, making it a responsive square that fills its grid cell
- Font size uses CSS Container Queries: `.half` has `container-type: size`, and `.name` uses `font-size: calc(1cqh * var(--font-pct))` — this makes font scale relative to the half-card's rendered height
- Card size slider (`cardSize` in mm) does NOT affect the screen preview

**Print (`@media print`):**
- All layout containers (`.cards-grid`, `.preview`, `.app`) are flattened to `display: block`
- Each `.card` gets explicit mm dimensions via CSS variables injected as inline styles: `--print-w`, `--print-h`
- Font size uses `--print-fs` (also in mm), calculated as `(cardSize / 2) * fontPct / 100`
- Container Queries are disabled (`container-type: unset`) so the mm font-size takes over
- Each card forces its own page via `break-before: page` / `page-break-before: always` (except first child)
- `@page { size: A4; margin: 0 }` eliminates browser-added margins

### Card structure

```
.card
  .half.half-top  (transform: rotate(180deg)) — inverted name for back-view when folded
    span.name
  .divider  (1px line)
  .half.half-bottom — upright name for front-view
    span.name
```

The top half is rotated 180deg as a whole container, which inverts both the text and any background image. When the square paper is folded along the divider into a tent shape, both sides read correctly.

### Font sizing logic

`fontPctEffective` is the percentage of half-card height the font should occupy:
- **Auto-fill mode**: `min(90, 170 / maxChars)` — derived from `(cardWidth * 0.85 / n) / (cardWidth / 2) * 100`
- **Manual mode**: user slider (10–95%)

### Background image pipeline

1. User selects an image via `<input type="file">`
2. `FileReader` loads it, `Image` object decodes it
3. `cropTo21()` draws the center-cropped 2:1 region onto an 800×400 canvas
4. Canvas exports as JPEG data URL → stored in `bgImage` ref
5. Applied as `background-image: url(...)` with `background-size: 100% 100%` on both `.half` elements
6. The `.half-top` rotation (180deg) also rotates the background image, so it appears correctly from the back

### Name input handling

- Names are split by newline, comma, or Chinese comma/顿号 (`[\n,，、]+`)
- Internal spaces are preserved by replacing `' '` with `'\u00A0'` (non-breaking space) after HTML-escaping the name
- Display uses `white-space: pre` and `v-html` to render the preserved spaces
- The escaping order matters: HTML entities first, then space replacement
