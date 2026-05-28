# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands (no necessary to run after editing code)

```bash
npm install          # install dependencies
npm run dev          # start dev server (Vite HMR)
npm run build        # production build → dist/
npm run preview      # preview production build locally
```

No tests, linter, or CI are configured locally. GitHub Actions (`.github/workflows/deploy.yml`) builds and deploys to GitHub Pages on push to `main`/`master`.

## Architecture

Single-page Vue 3 (Composition API, `<script setup>`) app built with Vite. The entire application lives in one file: `src/App.vue` — no router, no state management, no child components.

### Layout (screen only)

```
┌──────────────┬──────────────────────────┐
│  sidebar     │  .right-panel            │
│  (312px)     │  ┌────────────────────┐  │
│              │  │ .col-bar (1/2/3列) │  │
│  名字输入    │  ├────────────────────┤  │
│  纸张边长    │  │                    │  │
│  背景设置    │  │  .preview          │  │
│  字体设置    │  │  .cards-grid       │  │
│              │  │    .card .card …   │  │
│  [打印按钮]  │  │                    │  │
│  (sticky)    │  │                    │  │
└──────────────┴──────────────────────────┘
```

- `.sidebar` — all settings panels + sticky print button. `overflow-y: auto` with `position: sticky; bottom: 0` on print button wrapper so it's always visible.
- `.col-bar` — the lone setting moved to the right panel top (1/2/3 column toggle). Only this one control lives outside the sidebar.
- `.right-panel` — `flex column` container wrapping column bar + preview. On print, uses `display: contents` to dissolve into the flat document flow.

### Screen vs Print dual rendering

The same DOM produces two visual outputs:

**Screen (preview):**
- Cards in CSS Grid with `repeat(N, minmax(0, 1fr))` — responsive columns, max-width 960px
- Each `.card` has `aspect-ratio: 1` and `width: 100%` — squares that fill their grid cell
- Font size via CSS Container Queries: `.half` has `container-type: size`; `.name` uses `font-size: calc(1cqh * var(--font-pct))`
- `--font-pct` is set as an inline style on `.card` (from `cardStyle` computed), inherited by `.name`
- `cardSize` (mm) does not affect screen card size — it's stored in CSS vars for print only
- `.divider` renders as `border-top: 1px dashed #d0d0d0` (light gray dashed)

**Print (`@media print`):**
- `@page { size: A4; margin: 0 }` — full-bleed pages
- All `.no-print` elements hidden (sidebar, col-bar)
- `.right-panel` dissolved via `display: contents`
- `.cards-grid` flattened to `display: block`
- Each `.card` gets explicit mm dimensions via `--print-w` / `--print-h` CSS vars
- Font size overridden with `--print-fs` (mm), calculated as `(cardSize / 2) * fontPctEffective / 100`
- Container Queries disabled (`container-type: unset`) so mm font-size takes over
- `.divider` hidden via `display: none`
- Each card starts a new page: `break-before: page` + `page-break-before: always` (except first child)

### Card structure

```
.card
  .half.half-top  ← transform: rotate(180deg) inverts text + bg image
    span.name
  .divider         ← dashed fold guide (screen only)
  .half.half-bottom
    span.name
```

### Font sizing logic

`fontPctEffective` = percentage of half-card height the font should occupy:
- **Auto-fill**: `min(90, 170 / maxChars)` — derived from `(cardWidth * 0.85 / n) / (cardWidth / 2) * 100`
- **Manual**: user slider (10–95%)

On screen, `1cqh` = 1% of `.half` height, so `calc(1cqh * 70)` = 70% of half height. This ratio is preserved regardless of card size.

### Background image pipeline

1. File input → `FileReader` → `Image` decode
2. `cropTo21()` canvas-crops to 2:1 aspect ratio (center-crop), renders at 800×400
3. Canvas exports JPEG data URL → stored in `bgImage` ref
4. Applied as `background-image` with `background-size: 100% 100%` on both `.half` elements
5. `.half-top` rotation (180deg) also rotates the bg image — correct for folded back-view

### Name input

- Split by `[\n,，、]+` (newline, comma, Chinese comma/顿号)
- HTML-escaped, then spaces replaced with `\u00A0` (non-breaking) for spacing control
- Rendered via `v-html` with `white-space: pre`
