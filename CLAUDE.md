# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**WorkLoad — Proyección de Pedidos** is a Spanish-language order-forecasting single-page web application. It loads historical order data from Excel files, projects expected order volumes by ISO week, and shows a traffic-light workload indicator. The entire application lives in one file: `index.html`.

## Running the App

No build step or package manager. Open `index.html` directly in a browser, or serve it with any static server:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

There are no tests, no linter, and no CI configuration.

## Architecture

Everything — HTML structure, CSS, and JavaScript — is in `index.html` (~1300 lines). The file is organized with clear comment banners:

| Section | Lines (approx.) | Purpose |
|---|---|---|
| `<style>` | 11–453 | All CSS with CSS custom properties (dark theme) |
| HTML views | 456–609 | Three `<div class="view">` panels: `#view-dashboard`, `#view-history`, `#view-settings` |
| Bottom nav | 612–635 | Fixed tab bar that calls `switchView()` |
| `// STATE` | 640–647 | Four global variables that hold all runtime state |
| `// ISO WEEK HELPERS` | 649–687 | ISO 8601 date math (no UTC shortcuts — see below) |
| `// INIT` | 690–717 | `window.onload`: restores localStorage, kicks render |
| `// FILE HANDLING` | 720–800 | `handleFile()` — SheetJS Excel parse → `allData` |
| `// BUILD WEEKLY STATS` | 803–815 | `buildWeeklyStats()` — `allData` → `weeklyStats` |
| `// GET PROJECTION` | 830–956 | Core algorithm (see below) |
| `// RENDER *` | 978–1118 | DOM-update functions for each view |
| `// SETTINGS / FESTIVOS` | 1120–1251 | Threshold sliders, holiday file handler |
| `// UI HELPERS` | 1253–1297 | `switchView()`, `showToast()`, `showConfirm()` |

## Global State

```js
let allData = [];         // {fecha: Date, codigo: string, nombre: string}[]
let weeklyStats = {};     // {isoKey: {week, year, count, clients: {}}}
let threshRed = 100;      // absolute order count → red semaphore
let threshGreen = 50;     // absolute order count → green semaphore
let festivosData = [];    // Date[] — public holidays (Mon–Fri only affect capacity)
let currentWeekOffset = 0;
```

`weeklyStats` keys follow the pattern `"YYYY-WNN"` (e.g. `"2025-W03"`). Each entry's `clients` is a plain object keyed by `codigo` with `{nombre, count}`.

## Persistence (localStorage)

| Key | Format | Content |
|---|---|---|
| `wl_data` | JSON array | `allData` with `fecha` stored as ISO string |
| `wl_festivos` | Comma-separated `YYYY-MM-DD` | Holiday dates |

Thresholds are **not** persisted; they reset to defaults (100/50) on reload.

## Core Algorithm: `getProjection(targetWeek, targetYear)`

1. Collect all `weeklyStats` entries for `week === targetWeek` across all years.
2. Split into `historical` (years < targetYear) and `currentYearData` (year === targetYear).
3. **Holiday adjustment on historical data**: each fixed holiday (using `festivosEnSemanaHistorica`) falling Mon–Fri in a past occurrence adds +20 to that week's count, normalizing artificially low readings.
4. **Recency weighting**: most recent historical year gets highest weight. Schemes: 1 yr→100%, 2 yrs→70/30%, 3 yrs→50/30/20%, 4+→40/30/20/10%.
5. **Blend with current-year actuals**: if `currentYearData` exists, it contributes 50% and the weighted historical average contributes 50%.
6. **Holiday reduction on target week**: if `festivosEnSemana(targetYear, targetWeek)` returns holidays, multiply projected count by `(5 - holidays) / 5`.
7. **Client probability**: each client's probability is its recency-weighted appearance ratio across all source years.

The semaphore compares `proj.avgCount` to `capacidadAjustada()` (threshRed minus 20 per holiday) for red, and to `threshGreen` for green.

## ISO 8601 Date Handling

A deliberate pattern throughout the codebase: **never use `new Date(string)` or UTC methods** for dates from Excel, because `new Date("2024-01-15")` parses as UTC midnight, which shifts to the previous day in negative-offset timezones.

The solution used everywhere:
```js
// Parse parts directly into local constructor
const [y, m, d] = s.split('-').map(Number);
new Date(y, m - 1, d);  // local midnight, no UTC shift
```

`isoCalendar(date)` uses Thursday-anchored arithmetic (ISO 8601 rule: the week containing the Thursday belongs to that ISO year). Do not replace this with library calls that might use UTC internally.

## Excel Input Format

The order Excel file must have columns matching (case-insensitive, partial match):
- **Date**: `fecha`, `date`, or `fec`
- **Client code**: `código`, `codigo`, `cod`, `code`, or `id`
- **Client name**: `nombre`, `cliente`, `name`, `client`, or `nom`

The holidays Excel can be any layout — `handleFestivos()` scans every cell for numeric Excel date serials (40000–60000 range) or date-like strings.

## External Dependencies (CDN)

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
```

SheetJS is the only runtime dependency. Google Fonts (Syne, JetBrains Mono) are loaded from CDN for display purposes and are non-critical.

## UI Conventions

- CSS custom properties defined in `:root` — use `var(--accent)`, `var(--red)`, `var(--green)`, `var(--yellow)`, etc. for any new styled elements.
- `var(--font-mono)` (JetBrains Mono) for numeric values, labels, codes. `var(--font-display)` (Syne) for headings and UI text.
- Views are toggled by adding/removing the `active` class — only one `.view.active` is shown at a time.
- `showConfirm()` is used instead of `window.confirm()` because the browser's native confirm is blocked in iframe contexts.
- `showToast(msg)` for non-blocking user feedback (3-second auto-dismiss).
