# Mirador — Table Booking Map v2: Realistic Floor Plan (Design)

**Date:** 2026-06-29
**Status:** Approved design, frontend-only preview (no backend)
**Supersedes:** the v1 map in `2026-06-29-tables-booking-map-design.md` (this redesigns the
visual map; the booking interaction is largely reused)

## Why this redesign

The v1 map read as "a few vector shapes." The client wants two things, from two reference
images they provided:

1. **Realistic tables with chairs** (reference img 1) — top-down tables where each table has
   chairs drawn around it, a status dot with the table label in the center, and a clear
   selected state.
2. **A faithful venue floor plan** (reference img 2) — the real terrace laid out in labelled
   zones (Entrance, Part of Building / bar, DJ, an L-shaped Bar, Stand zones, and a large
   Table Zone), with bookable tables placed only inside the Table Zone (plus standing tables
   along the right-margin stand zone).

## Decisions (locked)

- **Theme:** keep the site's dark-luxury palette (dark background, gold/ivory strokes,
  crimson accents). Tables are rendered realistically with chairs, but in the dark theme —
  NOT the light/white look of img 1.
- **Table states:** three — **Liber** (available), **Rezervat** (reserved), **Ocupat**
  (filled). No countdown/"available soon" timers (we have no live backend).
- **Layout fidelity:** full zoned floor plan as in img 2, with labelled zones; bookable
  tables only inside the Table Zone, plus 4 standing tables in the right-margin stand zone.
- **Table inventory:** Table Zone has ~20 tables (8×2 + 6×4 + 4×6 + 2×8 = 80 seats); stand
  zone adds 4 standing bar tables (2 seats each = 8 seats). **Total: 24 tables, 88 seats.**

## Placement & Scope

- This replaces the **rendering** of the existing `#tables` section (added in the v1 work).
  The section element, nav links, the reservation form, validation, confirmation, mobile
  bottom-sheet, and keyboard handling are **reused with minimal change**.
- All code remains inside the single `index.html` (inline `<style>` + the `initTableBooking`
  IIFE in the inline `<script>`).

## The Floor Plan (inline SVG, dark-luxury styling)

A wider, landscape `viewBox` holds the whole terrace. Rendered with the site's idiom: thin
gold/ivory strokes, dark fills, small uppercase letter-spaced labels (Montserrat), Cormorant
only where a display feel is wanted.

Zones (positions tuned in the implementation plan; relative arrangement fixed here to match
img 2):

- **Terrace outline** — rounded rectangle, thin gold stroke, framing everything.
- **ENTRANCE** — small block, top-left. Subtle green-tinted edge (echoes img 2 green) but
  muted for the dark theme. Label "ENTRANCE".
- **PART OF BUILDING** — dark charcoal block next to the entrance, label
  "PART OF BUILDING" with a smaller "cu barul / with the bar" sub-label.
- **DJ** — small block, label "DJ".
- **BAR (L-shaped)** — a near-black block forming an L: a horizontal run across the
  top-center and a vertical run up the top-right. Gold "BAR" label (the vertical arm's label
  rotated). This is the visual anchor that matches the photo's bar-in-the-corner.
- **STAND ZONE (main)** — the band directly below the bar/blocks; faint tint, label
  "STAND ZONE". Non-bookable (standing area) except the right margin (below).
- **STAND ZONE (right margin)** — the vertical strip in the top-right under the bar's
  vertical arm; label "STAND ZONE". Holds the 4 standing bar tables.
- **TABLE ZONE** — the large bordered region across the bottom; label "TABLE ZONE". Holds
  the 20 seated tables.

Zone labels are decorative (not interactive). Only tables are interactive.

## Realistic Tables with Chairs

Each table is one interactive `<g class="mt-table" data-id data-cap data-status …>` containing:

- **Chairs** — rounded-rectangle "seats" positioned around the table top, count = capacity,
  drawn first (behind the table top). Dark fill, ivory-dim stroke. Chairs are purely
  decorative (not individually interactive).
- **Table top** — the surface shape, by capacity:
  - **2 seats** → small circle, 1 chair on each of two opposite sides.
  - **4 seats** → rounded square, 1 chair per side.
  - **6 seats** → long "stadium"/pill (rounded-end rectangle), 3 chairs per long side.
  - **8 seats** → longer pill, 4 chairs per long side.
  - **standing (2 seats)** → small circle table with 2 small "stool" dots instead of chairs,
    to read as a high bar table.
- **Status disc** — a filled circle centered on the table top, colored by status, with the
  table label ("T01"…"T24") in the center.

### Table states (visual)

| State | Disc color | Table top | Interactive | Extra |
|-------|------------|-----------|-------------|-------|
| Liber (available) | green (`--lime`) | normal stroke | yes | hover glow + tooltip |
| Rezervat (reserved) | crimson (`--crimson`) | dimmed | no | small "Rezervat" badge under the table; `cursor:not-allowed` |
| Ocupat (filled) | grey (`--ivory-dim`) | dimmed | no | `cursor:not-allowed` |
| Selected | green disc + **gold ring** around the whole table (Mirador signature) | highlighted | (the chosen one) | other available tables dim slightly |

A handful of tables start `reserved` and a handful start `filled` for realism (exact set in
the plan).

### Legend

Three chips: "● Liber" (green), "● Rezervat" (crimson), "● Ocupat" (grey), plus a small note
"2 · 4 · 6 · 8 locuri".

## Data Model

The single source of truth is the `TABLES` array. Each entry:

```
{ id, label, cap, type, x, y, status, orient }
```

- `id` — stable id (e.g. `'t1'`).
- `label` — display label (`'T01'`…`'T24'`).
- `cap` — 2 | 4 | 6 | 8 (standing tables use cap 2 with `type:'stand'`).
- `type` — `'round'` | `'square'` | `'pill'` | `'stand'` (drives the table-top shape + chair
  layout; chosen from `cap` if omitted, but stored explicitly for clarity).
- `x`, `y` — center in viewBox units.
- `status` — `'available' | 'reserved' | 'filled'`.
- `orient` — `'h'` | `'v'` for pill tables (long axis horizontal/vertical), so the Table Zone
  can be packed without overlap.

Rendering, chair placement, status disc, tooltip, and the form are all derived from this
array. Counts are verified during implementation (24 tables, 88 seats, the 8/6/4/2 + 4-stand
mix).

## Interaction (reused from v1, with one change)

Unchanged: click an available table → reservation card (desktop) / bottom-sheet (mobile);
form with Masă chip, Nume, **Telefon (required)**, Dată (min today), Oră (17:00–02:00 in
30-min steps), Nr. persoane (1..capacity); validation blocks submit and focuses the first
invalid field; animated confirmation `Masa T04 · 12 Iulie · 20:00` with the demo note; booked
table flips to reserved; "Alege altă masă" resets; keyboard activation (Enter/Space), aria
labels, aria-live confirmation; `prefers-reduced-motion` respected.

**Change:** selection is blocked for BOTH `reserved` and `filled` tables (v1 only had
reserved). The click guard, keyboard guard, hover tooltip text, and `tableEl()` markup all
account for the third status.

## Responsive

- The floor plan is landscape. It scales to container width via `viewBox` +
  `preserveAspectRatio`, so on mobile it becomes shorter; tap targets are the whole table
  `<g>` (chairs + top + disc), which keeps them tappable.
- On narrow screens (≤620px) the map sits in a horizontally scrollable frame: the SVG keeps
  a `min-width` (≈ 640px) so tables never shrink below a usable tap size, and the frame
  scrolls sideways. A small hint ("← glisează →") indicates the map is scrollable.
- Desktop: two-column (map left, sticky form right). Mobile: full-width map + bottom-sheet,
  as in v1.

## Out of Scope (preview)

- No backend, availability API, persistence, payment, or notifications.
- No countdown timers / "available soon" state.
- Reserved/filled sets are hardcoded for realism.

## Reuse / Change Summary

- **Rewritten:** the SVG render (`renderMap` + a new `tableEl` that draws chairs + top +
  disc), the zones markup, the CSS for the map/tables/legend, and the `TABLES` data array
  (now 24 entries with `type`/`orient`/3 statuses).
- **Reused, lightly touched:** `selectTable`, `buildFormHTML`, `wireSelection`,
  `wireForm`, `resetPanel`, `showConfirmation`, `openAside`/`closeAside`, helpers
  (`pad2`, `todayISO`, `formatRoDate`, `timeSlots`, `isMobile`) — only the status guards and
  tooltip/legend strings change.
- **Unchanged:** section `#tables` position, nav links, the fallback `#reservation` form.
