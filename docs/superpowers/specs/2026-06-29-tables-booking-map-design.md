# Mirador — Interactive Table Booking Map (Design)

**Date:** 2026-06-29
**Status:** Approved design, frontend-only preview (no backend)

## Goal

Add an interactive, top-down map of the Mirador rooftop terrace where a guest can pick a
table and fill a styled reservation form. This is a **preview / mockup** — no booking
backend, no persistence, no real availability. State lives entirely in client-side JS.

The feature is a visual "step 1" placed **before** the existing `#reservation` section.
The current simple email form stays underneath as a fallback.

## Placement & Navigation

- New section `<section id="tables">` inserted immediately **before** `<section id="reservation">`.
- Section heading in Mirador style: eyebrow tag ("The Floor" / "Alege-ți masa"),
  italic Cormorant title, short intro line.
- Add a nav link (`#tables`) in both the desktop `.nav-links` and the `.mobile-menu`,
  e.g. label "Tables", positioned between "Events" and "Contact".

## The Map (SVG line-art)

Rendered as inline SVG to match the existing line-art aesthetic (logo, decorative strokes).

- Rectangular terrace outline, thin gold/ivory stroke (`--gold` / `--ivory`), rounded corners.
- **BAR** block in the back corner (mirrors the real photo: bar against the building structure).
- Discreet **entrance / stairs** marker near the bar corner.
- Subtle **"← CITY VIEW →"** labels along the two open edges (the balustrade sides).
- Fine vegetal accents (small dots / sprigs) along the perimeter to echo the floral decor.
- Tables drawn as circles, radius scaled by capacity (2 < 4 < 6 < 8).
- Each table has a small numeric label.

### Table inventory (~80 seats, 20 tables)

| Capacity | Tables | Seats | Placement |
|----------|--------|-------|-----------|
| 2 people | 8      | 16    | perimeter, by the view/rail |
| 4 people | 6      | 24    | central zone |
| 6 people | 4      | 24    | larger tables |
| 8 people | 2      | 16    | lounge, in corners |

Tables are defined as a JS data array — single source of truth — each with:
`{ id, label, capacity, x, y, status }` where `status` ∈ `available | reserved`.
A handful of tables are pre-set to `reserved` for realism.

### Table visual states

- **available** — line-art circle, ivory stroke.
- **hover** — crimson glow + tooltip "Masă 04 · 4 locuri" (capacity-aware text).
- **selected** — crimson fill; all other available tables dim slightly to focus attention.
- **reserved** — greyed, reduced opacity, not interactive, cursor not-allowed; tooltip
  "Rezervat".

## Interaction Flow

1. Guest clicks an available table → it becomes **selected**.
2. The reservation card appears:
   - **Desktop:** card to the right of the map, sticky (map ~60% / card ~40%).
   - **Mobile:** a bottom-sheet slides up (reuse the existing `.cocktail-sheet`
     bottom-sheet pattern already in the codebase).
3. Form fields (all fields required, including phone):
   - **Masă** — read-only chip showing number + capacity (e.g. "Masa 04 · 4 locuri").
   - **Nume** — text, required.
   - **Telefon** — tel, **required**.
   - **Dată** — date picker, required; min = today.
   - **Oră** — select, required; options only within opening hours **17:00 → 02:00**
     (e.g. 17:00, 17:30 … 01:30, 02:00).
   - **Nr. persoane** — select, required; options `1 … capacity` (max = selected table's
     capacity).
4. **"Confirmă rezervarea"** button:
   - Validates all fields (native HTML5 `required` + JS guard). Phone must be non-empty.
   - On success → animated confirmation state replacing the form body:
     *"Rezervare trimisă · Masa 04 · 12 Iulie · 20:00"* plus a discreet note that this is a
     demo preview without a live booking system.
   - Selected table can optionally flip to a temporary "reserved" look in this session.
5. A reset / "Alege altă masă" action clears selection and returns to the map.

## Validation Rules

- Name: required, non-empty after trim.
- Phone: required, non-empty after trim (light format hint only; no strict regex for the demo).
- Date: required, not in the past.
- Time: required, must be one of the generated 17:00–02:00 slots.
- Party size: required, between 1 and the selected table's capacity.
- Submit is blocked (and the first invalid field focused) until all rules pass.

## Responsive Behaviour

- **Desktop (≥ 821px):** two-column — SVG map left, sticky form card right. Tooltip on hover.
- **Mobile (≤ 820px):** map full-width; tapping a table opens the bottom-sheet form
  (mirrors `.cocktail-sheet`). No hover tooltip; tap = select.
- Map SVG uses a `viewBox` so it scales cleanly; touch targets enlarged on coarse pointers.

## Styling

- Reuse existing CSS variables (`--bg`, `--crimson`, `--rose`, `--ivory`, `--gold`, …),
  fonts (Cormorant Garamond italic for headings, Montserrat for UI), and section rhythm
  (padding, eyebrow tags, reveal-on-scroll classes).
- Confirmation animation consistent with the site's cubic-bezier easing and crimson accents.
- Respect `prefers-reduced-motion`.

## Accessibility

- Tables are keyboard-focusable (`tabindex`, `role="button"`, `aria-label` with table
  number + capacity + status). Enter/Space selects.
- Tooltip content also exposed via `aria-label` so it is not hover-only.
- Form fields have associated labels; confirmation state announced via `aria-live`.
- Focus moves into the form/sheet when a table is selected.

## Out of Scope (preview)

- No backend, API, database, or real availability checks.
- No payment, email, or SMS dispatch.
- No persistence across reloads (refresh resets all state).
- Reserved-table set is hardcoded for realism, not driven by real data.

## Implementation Notes

- Everything stays inside the single `index.html` (style block + body + the existing
  inline `<script>`), consistent with the current project structure.
- Table data array is the single source of truth; the SVG and form are rendered/driven
  from it.
