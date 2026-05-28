# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Static web portal for **SESI SENAI Goiás** (`sesisenai.inf.br`). No build tools, no package manager, no framework — pure HTML, CSS, and vanilla JavaScript. All assets are served directly from the filesystem or a static HTTP server.

## Running Locally

Since there is no build step, serve files with any static HTTP server from the repo root:

```bash
python3 -m http.server 8080
# or
npx serve .
```

Then open `http://localhost:8080` for the landing page and `http://localhost:8080/acoes-diretores/` for the directors panel.

There are no tests, no linters, and no CI configuration in this repository.

## Architecture

### Entry point — `index.html`

Landing page for the domain. Light-themed, single self-contained file (CSS is inlined, no external stylesheet). The only JavaScript is a one-liner that writes the current year into the footer. Links out to sub-applications via anchor cards.

Brand design tokens are defined as CSS custom properties at the top of the `<style>` block:
- `--azul-sesi-senai: #15499B` (primary blue)
- `--laranja: #F04B16`, `--verde: #58B031` (accent colours)

### Sub-application — `acoes-diretores/`

A full client-side SPA for SESI directors to register strategic actions, view metrics, and export reports. Three files:

| File | Role |
|---|---|
| `acoes-diretores/index.html` | HTML shell: layout, all modals, and all DOM anchor points |
| `acoes-diretores/app.js` | All application logic (≈900 lines, vanilla JS) |
| `acoes-diretores/styles.css` | Dark-theme design system (≈1500 lines) |

**Important path quirk:** `acoes-diretores/index.html` references its own CSS and JS with the paths `acoes-diretores/styles.css` and `acoes-diretores/app.js`. These paths are relative to the file's directory, so they resolve to `acoes-diretores/acoes-diretores/…`, which is incorrect for direct file access. The app is intended to be served from the repo root (e.g. `http://localhost:8080/acoes-diretores/`) where those paths would be served correctly by a server rewriting the base, or accessed via the root `index.html` link.

### State and data model

All state lives in a single global `state` object:

```js
const state = {
  actions: [],        // array persisted to localStorage
  viewMode: 'weekly' | 'monthly',
  activeWeek: '',     // YYYY-Www  (e.g. "2026-W22")
  activeMonth: '',    // YYYY-MM
  directorName: '',
  schoolUnit: '',
  photoBase64: '',    // active image being edited
  photoMeta: { ... }
};
```

**localStorage keys:**
- `acoes_estrategicas_db` — JSON array of all action records
- `diretor_nome` — director's display name
- `diretor_escola` — school unit name

**Action record shape:**
```js
{ id, titulo, assunto, data, status, sumario, escola, diretor, evidencia }
```
`evidencia` stores the photo as a base64 data URL (compressed client-side via Canvas API to max 900 px / JPEG 70%).

### Render pattern

`render()` is the single function that rebuilds the entire actions grid. It is called after every state change (filter, period change, save, delete). It works by setting `grid.innerHTML` wholesale — there is no virtual DOM or diffing.

KPI counters (`kpi-total`, `kpi-sent`, `kpi-photos`, `kpi-rate`) are recalculated inside `render()` from the period-filtered slice of `state.actions` before secondary filters are applied.

### Print / PDF system

CSS classes `.no-print` / `.show-print` toggle visibility between screen and `@media print`. The consolidated report hides the sidebar and controls and shows a structured print header and signature block. Individual action print works by appending a temporary `div.temp-print-area` to `<body>`, calling `window.print()`, then removing it.

CSV export uses `;` as the delimiter and prepends a UTF-8 BOM (`﻿`) for correct rendering in Brazilian Windows Excel.

### Third-party dependencies (CDN, no local install)

- **Lucide icons** — `https://unpkg.com/lucide@latest` — initialised via `lucide.createIcons()` after every `render()` call
- **Google Fonts** — Outfit (headings) + Inter (body) — loaded in `acoes-diretores/index.html` only; the root `index.html` uses system `Arial`

### Two separate design systems

The root landing page and the `acoes-diretores` app have completely independent CSS. Do not mix their tokens. The directors app uses a dark theme (`--bg-app: #080711`) with Indigo as the primary accent (`--primary: #6366f1`).
