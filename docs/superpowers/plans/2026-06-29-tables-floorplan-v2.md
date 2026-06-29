# Table Booking Map v2 — Realistic Floor Plan Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the `#tables` map into a faithful zoned floor plan (Entrance / Bar-L / DJ / Stand zones / Table zone) with realistic top-down tables that have chairs and three statuses (Liber / Rezervat / Ocupat), keeping the existing booking interaction.

**Architecture:** All changes are inside `D:\Mirador-website\index.html` — the inline `<style>` block and the `initTableBooking` IIFE in the inline `<script>`. The map's CSS, the `TABLES` data array, the `tableEl` renderer, and `renderMap` are rewritten; the booking form, validation, confirmation, mobile sheet, and keyboard handling are reused with small status-guard edits.

**Tech Stack:** Plain HTML + CSS + vanilla JS, inline SVG. No build, no libraries.

## Global Constraints

- All code stays in `D:\Mirador-website\index.html`. No new files, no libraries. CSS classes stay `mt-` prefixed.
- Keep the dark-luxury palette: reuse `--bg`, `--crimson`, `--rose`, `--ivory`, `--ivory-dim`, `--gold`, `--lime`, `--violet`. Tables are realistic (chairs) but in the dark theme — NOT light/white.
- Three table statuses: `available` (Liber, green `--lime`), `reserved` (Rezervat, `--crimson`), `filled` (Ocupat, grey). No countdown timers.
- Selected table = **gold ring** (`.mt-halo` stroked gold). Both `reserved` and `filled` are non-selectable.
- Inventory exactly **24 tables / 88 seats**: Table Zone has 20 (8×2 round, 6×4 square, 4×6 pill6, 2×8 pill8 = 80 seats); Stand zone right margin has 4 standing 2-seat tables (= 8 seats). Reserved start: T03, T11, T22. Filled start: T07, T15, T20, T24.
- UI copy is Romanian. Confirmation summary format unchanged: `Masa T04 · 12 Iulie · 20:00`.
- Opening hours 17:00→02:00 (30-min steps), telephone required, date min = today — all unchanged from v1.
- No backend/persistence. Respect `prefers-reduced-motion` (existing `#tables *` rule already covers it).

**Testing reality:** Static single-file site, no test runner. Verify by serving (`python -m http.server 8000` from `D:\Mirador-website`, open `http://localhost:8000`, hard-refresh Ctrl+F5) and observing the browser, plus `node --check` on the extracted `<script>` for JS edits. Temporary `console.assert` checks are removed before their task's commit.

**Note on intermediate states:** This redesign touches CSS (Task 1) and the renderer (Task 2) in sequence. Between Task 1 and Task 2 the map briefly looks wrong (old renderer emits classes the new CSS dropped); between Task 2 and Task 3 `filled` tables are briefly still clickable. Both are expected and resolved by the final task. Each task's verification targets what that task delivers.

---

### Task 1: CSS + section markup for zones, realistic tables, 3-state legend, mobile scroll

**Files:**
- Modify: `index.html` — replace the map CSS block (lines 1122–1155, from `.mt-outline{` through `.mt-legend i.res{…}`); add a `≤620px` media block before the `prefers-reduced-motion` rule (~line 1213); update the section markup (legend + scroll wrapper + intro copy, ~lines 1627–1638).

**Interfaces:**
- Consumes: existing CSS vars and the `.mt-map-frame` / `#table-map` / `.mt-legend` markup.
- Produces: CSS classes the renderer (Task 2) emits — `.mt-zone`, `.mt-zone-outline/-entrance/-building/-dj/-bar/-stand/-table`, `.mt-zone-lbl` (+ `.sm/.bar/.zone`), `.mt-table.s-available/.s-reserved/.s-filled`, `.mt-halo`, `.mt-chair`, `.mt-top`, `.mt-disc`, `.mt-lbl`, `.mt-badge`; legend dot classes `.lg-av/.lg-rs/.lg-fl`; `.mt-scroll` + `.mt-scroll-hint`.

- [ ] **Step 1: Replace the map CSS block (lines 1122–1155)**

Edit — find:
```css
.mt-outline{ fill:none; stroke:var(--gold); stroke-width:1.4; stroke-opacity:.5; }
.mt-bar{ fill:rgba(196,160,96,.06); stroke:var(--gold); stroke-width:1; stroke-opacity:.55; }
.mt-bar-lbl{ fill:var(--gold); font-family:'Montserrat',sans-serif; font-size:16px;
  letter-spacing:.4em; text-transform:uppercase; text-anchor:middle; }
.mt-view-lbl{ fill:var(--ivory-dim); font-family:'Montserrat',sans-serif; font-size:13px;
  letter-spacing:.5em; text-transform:uppercase; text-anchor:middle; }
.mt-sprig{ fill:none; stroke:rgba(156,204,74,.35); stroke-width:1; }

.mt-table .mt-disc{ fill:rgba(237,227,216,.03); stroke:var(--ivory); stroke-width:1.2;
  transition:fill .35s, stroke .35s, opacity .35s; }
.mt-table .mt-lbl{ fill:var(--ivory-dim); font-family:'Montserrat',sans-serif; font-size:15px;
  font-weight:300; text-anchor:middle; dominant-baseline:central; pointer-events:none;
  transition:fill .35s; }
.mt-table:not(.reserved):hover .mt-disc{ fill:rgba(155,31,53,.25); stroke:var(--rose); }
.mt-table.selected .mt-disc{ fill:var(--crimson); stroke:var(--rose); }
.mt-table.selected .mt-lbl{ fill:var(--ivory); }
.mt-table.reserved{ cursor:not-allowed; }
.mt-table.reserved .mt-disc{ fill:rgba(237,227,216,.02); stroke:rgba(237,227,216,.18);
  stroke-dasharray:3 4; }
.mt-table.reserved .mt-lbl{ fill:rgba(237,227,216,.22); }
.mt-map.has-selection .mt-table:not(.selected):not(.reserved) .mt-disc{ opacity:.4; }

.mt-tip{ position:fixed; z-index:90; pointer-events:none; background:rgba(5,3,10,.92);
  border:1px solid rgba(196,160,96,.4); padding:8px 12px; font-size:.56rem; letter-spacing:.18em;
  text-transform:uppercase; color:var(--ivory); white-space:nowrap; opacity:0;
  transform:translate(-50%,-150%); transition:opacity .2s; }
.mt-tip.show{ opacity:1; }

.mt-legend{ display:flex; flex-wrap:wrap; gap:18px; margin-top:18px; }
.mt-legend span{ font-size:.5rem; letter-spacing:.3em; text-transform:uppercase;
  color:var(--ivory-dim); display:flex; align-items:center; gap:8px; }
.mt-legend i{ width:12px; height:12px; border-radius:50%; border:1px solid var(--ivory);
  display:inline-block; }
.mt-legend i.res{ border-style:dashed; border-color:rgba(237,227,216,.3); }
```
Replace with:
```css
/* ── floor-plan zones ── */
.mt-zone{ stroke-width:1.2; }
.mt-zone-outline{ fill:none; stroke:var(--gold); stroke-width:1.4; stroke-opacity:.55; }
.mt-zone-entrance{ fill:rgba(156,204,74,.10); stroke:rgba(156,204,74,.5); }
.mt-zone-building{ fill:rgba(237,227,216,.05); stroke:rgba(237,227,216,.22); }
.mt-zone-dj{ fill:rgba(156,204,74,.10); stroke:rgba(156,204,74,.5); }
.mt-zone-bar{ fill:rgba(8,5,12,.9); stroke:var(--gold); stroke-opacity:.5; }
.mt-zone-stand{ fill:rgba(138,106,191,.10); stroke:rgba(138,106,191,.4); }
.mt-zone-table{ fill:rgba(138,106,191,.06); stroke:rgba(196,160,96,.4); stroke-width:1.4; }
.mt-zone-lbl{ fill:var(--ivory-dim); font-family:'Montserrat',sans-serif; font-size:13px;
  letter-spacing:.32em; text-transform:uppercase; text-anchor:middle; dominant-baseline:central; }
.mt-zone-lbl.sm{ font-size:10px; letter-spacing:.2em; }
.mt-zone-lbl.bar{ fill:var(--gold); }
.mt-zone-lbl.zone{ fill:rgba(237,227,216,.28); font-size:15px; letter-spacing:.4em; }

/* ── realistic tables ── */
.mt-table{ cursor:none; }
.mt-halo{ fill:none; stroke:none; transition:stroke .3s; }
.mt-table.selected .mt-halo{ stroke:var(--gold); stroke-width:2; }
.mt-chair{ fill:rgba(237,227,216,.08); stroke:rgba(237,227,216,.25); stroke-width:1;
  transition:opacity .35s; }
.mt-top{ fill:rgba(237,227,216,.05); stroke:var(--ivory); stroke-width:1.2; stroke-opacity:.5;
  transition:stroke .35s, stroke-opacity .35s; }
.mt-disc{ transition:fill .35s, filter .35s; }
.mt-lbl{ fill:#0a0610; font-family:'Montserrat',sans-serif; font-size:13px; font-weight:600;
  text-anchor:middle; dominant-baseline:central; pointer-events:none; }
.mt-table.s-available .mt-disc{ fill:var(--lime); }
.mt-table.s-reserved  .mt-disc{ fill:var(--crimson); }
.mt-table.s-reserved  .mt-lbl{ fill:var(--ivory); }
.mt-table.s-filled    .mt-disc{ fill:rgba(237,227,216,.30); }
.mt-table.s-available:hover .mt-top{ stroke:var(--rose); stroke-opacity:1; }
.mt-table.s-available:hover .mt-disc{ filter:brightness(1.12); }
.mt-table.selected .mt-top{ stroke:var(--gold); stroke-opacity:1; }
.mt-table.s-reserved, .mt-table.s-filled{ cursor:not-allowed; }
.mt-table.s-reserved .mt-top, .mt-table.s-filled .mt-top{ stroke-opacity:.28; }
.mt-table.s-reserved .mt-chair, .mt-table.s-filled .mt-chair{ opacity:.4; }
.mt-map.has-selection .mt-table.s-available:not(.selected){ opacity:.45; }
.mt-badge{ font-family:'Montserrat',sans-serif; font-size:10px; letter-spacing:.2em;
  text-transform:uppercase; text-anchor:middle; fill:var(--rose); }

.mt-tip{ position:fixed; z-index:90; pointer-events:none; background:rgba(5,3,10,.92);
  border:1px solid rgba(196,160,96,.4); padding:8px 12px; font-size:.56rem; letter-spacing:.18em;
  text-transform:uppercase; color:var(--ivory); white-space:nowrap; opacity:0;
  transform:translate(-50%,-150%); transition:opacity .2s; }
.mt-tip.show{ opacity:1; }

/* ── scroll frame (mobile) ── */
.mt-scroll{ overflow-x:auto; -webkit-overflow-scrolling:touch; }
.mt-scroll-hint{ display:none; font-size:.5rem; letter-spacing:.3em; text-transform:uppercase;
  color:var(--ivory-dim); margin-top:10px; text-align:center; }

/* ── legend ── */
.mt-legend{ display:flex; flex-wrap:wrap; gap:18px; margin-top:18px; }
.mt-legend span{ font-size:.5rem; letter-spacing:.3em; text-transform:uppercase;
  color:var(--ivory-dim); display:flex; align-items:center; gap:8px; }
.mt-legend i{ width:11px; height:11px; border-radius:50%; display:inline-block; }
.mt-legend i.lg-av{ background:var(--lime); }
.mt-legend i.lg-rs{ background:var(--crimson); }
.mt-legend i.lg-fl{ background:rgba(237,227,216,.4); }
```

- [ ] **Step 2: Add the ≤620px scroll media block**

Edit — find:
```css
@media (prefers-reduced-motion:reduce){
  #tables *{ transition:none !important; animation:none !important; }
}
```
Replace with:
```css
@media (max-width:620px){
  #table-map{ min-width:640px; }
  .mt-scroll-hint{ display:block; }
}
@media (prefers-reduced-motion:reduce){
  #tables *{ transition:none !important; animation:none !important; }
}
```

- [ ] **Step 3: Update the section markup (scroll wrapper, legend, intro copy)**

Edit — find:
```html
    <h2>Alege-ți <em>masa</em></h2>
    <p>O hartă a terasei, văzută de sus. Atinge o masă liberă și rezervă-ți locul cu vedere spre oraș. Aceasta este o previzualizare — momentan fără sistem de rezervări activ.</p>
```
Replace with:
```html
    <h2>Alege-ți <em>masa</em></h2>
    <p>Harta reală a terasei, văzută de sus — cu zonele locației (bar, DJ, intrare) și mesele cu locurile lor. Atinge o masă liberă și rezervă-ți locul. Previzualizare — momentan fără sistem de rezervări activ.</p>
```

Edit — find:
```html
      <div class="mt-map-frame">
        <svg id="table-map" role="group" aria-label="Hartă mese Mirador"></svg>
      </div>
      <div class="mt-legend">
        <span><i></i> Disponibil</span>
        <span><i class="res"></i> Rezervat</span>
        <span>2 · 4 · 6 · 8 locuri</span>
      </div>
```
Replace with:
```html
      <div class="mt-map-frame">
        <div class="mt-scroll">
          <svg id="table-map" role="group" aria-label="Hartă mese Mirador"></svg>
        </div>
      </div>
      <p class="mt-scroll-hint">← glisează harta →</p>
      <div class="mt-legend">
        <span><i class="lg-av"></i> Liber</span>
        <span><i class="lg-rs"></i> Rezervat</span>
        <span><i class="lg-fl"></i> Ocupat</span>
        <span>2 · 4 · 6 · 8 locuri</span>
      </div>
```

- [ ] **Step 4: Verify in the browser**

Hard-refresh. The map area is still driven by the old renderer, so the SVG itself will look broken/unstyled (expected — fixed in Task 2). Verify the parts THIS task owns: the legend now shows three filled dots (green / crimson / grey) labelled Liber · Rezervat · Ocupat plus "2 · 4 · 6 · 8 locuri"; the intro copy is updated; no CSS parse errors (other sections unaffected). Resize ≤620px: the "← glisează harta →" hint appears. Confirm exactly one `</style>` remains.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "style(tables): v2 floor-plan zones, realistic tables, 3-state legend"
```

---

### Task 2: Data model + geometry config + realistic table renderer + zoned map

**Files:**
- Modify: `index.html` — replace the IIFE block from `const VIEWBOX = { w:1000, h:700 };` through the end of `renderMap` (lines 1967–2024).

**Interfaces:**
- Consumes: `root` (#table-map), `mapGroup` (assigned in `renderMap`), the CSS classes from Task 1.
- Produces (in closure): `VIEWBOX` (1120×1000), `SHAPES` (per-type geometry), `RO_STATUS` (status→Romanian word), `TABLES` (24 entries with `type`/`status`), `chairRect`, `topShape`, `zoneRect`, `zoneLabel`, `tableEl`, `renderMap`. Each table renders as `<g class="mt-table s-<status>" data-id data-cap data-status role="button" tabindex aria-label>` containing a `.mt-halo`, `.mt-chair`s, a `.mt-top`, a `.mt-disc`, a `.mt-lbl`, and (reserved only) a `.mt-badge`.

- [ ] **Step 1: Replace VIEWBOX/RADIUS/TABLES/tableEl/renderMap**

Edit — find the block starting at `const VIEWBOX = { w:1000, h:700 };` and ending with the closing brace of `renderMap` (the line `  }` immediately before `  function pad2(n)`). Replace the whole block with:
```javascript
  const VIEWBOX = { w:1120, h:1000 };

  // Per-type geometry: table-top shape, chair offsets (from center), status-disc radius,
  // and the selection-halo half-dimensions (covers chairs).
  const SHAPES = {
    round:  { top:{ kind:'circle', r:20 }, disc:15, halo:{ hw:48, hh:26 },
      chairs:[ {dx:-34,dy:0,w:12,h:22}, {dx:34,dy:0,w:12,h:22} ] },
    square: { top:{ kind:'rect', w:56, h:56, rx:10 }, disc:17, halo:{ hw:54, hh:54 },
      chairs:[ {dx:0,dy:-40,w:24,h:12}, {dx:0,dy:40,w:24,h:12},
               {dx:-40,dy:0,w:12,h:24}, {dx:40,dy:0,w:12,h:24} ] },
    pill6:  { top:{ kind:'rect', w:150, h:52, rx:26 }, disc:18, halo:{ hw:86, hh:50 },
      chairs:[ {dx:-45,dy:-40,w:24,h:12}, {dx:0,dy:-40,w:24,h:12}, {dx:45,dy:-40,w:24,h:12},
               {dx:-45,dy:40,w:24,h:12}, {dx:0,dy:40,w:24,h:12}, {dx:45,dy:40,w:24,h:12} ] },
    pill8:  { top:{ kind:'rect', w:200, h:52, rx:26 }, disc:18, halo:{ hw:110, hh:50 },
      chairs:[ {dx:-67,dy:-40,w:24,h:12}, {dx:-22,dy:-40,w:24,h:12}, {dx:22,dy:-40,w:24,h:12}, {dx:67,dy:-40,w:24,h:12},
               {dx:-67,dy:40,w:24,h:12}, {dx:-22,dy:40,w:24,h:12}, {dx:22,dy:40,w:24,h:12}, {dx:67,dy:40,w:24,h:12} ] },
    stand:  { top:{ kind:'circle', r:16 }, disc:13, halo:{ hw:36, hh:22 },
      chairs:[ {dx:-26,dy:0,w:11,h:11}, {dx:26,dy:0,w:11,h:11} ] }
  };

  const RO_STATUS = { available:'Liber', reserved:'Rezervat', filled:'Ocupat' };

  // Single source of truth — 24 tables / 88 seats.
  // Table Zone (20): 8×round(2), 6×square(4), 4×pill6(6), 2×pill8(8) = 80 seats.
  // Stand zone right margin (4): standing 2-seat = 8 seats.
  const TABLES = [
    { id:'t1',  label:'T01', cap:8, type:'pill8',  x:190, y:485, status:'available' },
    { id:'t2',  label:'T02', cap:4, type:'square', x:400, y:485, status:'available' },
    { id:'t3',  label:'T03', cap:6, type:'pill6',  x:610, y:485, status:'reserved'  },
    { id:'t4',  label:'T04', cap:4, type:'square', x:820, y:485, status:'available' },
    { id:'t5',  label:'T05', cap:2, type:'round',  x:190, y:588, status:'available' },
    { id:'t6',  label:'T06', cap:6, type:'pill6',  x:400, y:588, status:'available' },
    { id:'t7',  label:'T07', cap:2, type:'round',  x:610, y:588, status:'filled'    },
    { id:'t8',  label:'T08', cap:4, type:'square', x:820, y:588, status:'available' },
    { id:'t9',  label:'T09', cap:4, type:'square', x:190, y:691, status:'available' },
    { id:'t10', label:'T10', cap:2, type:'round',  x:400, y:691, status:'available' },
    { id:'t11', label:'T11', cap:8, type:'pill8',  x:610, y:691, status:'reserved'  },
    { id:'t12', label:'T12', cap:2, type:'round',  x:820, y:691, status:'available' },
    { id:'t13', label:'T13', cap:6, type:'pill6',  x:190, y:794, status:'available' },
    { id:'t14', label:'T14', cap:2, type:'round',  x:400, y:794, status:'available' },
    { id:'t15', label:'T15', cap:4, type:'square', x:610, y:794, status:'filled'    },
    { id:'t16', label:'T16', cap:2, type:'round',  x:820, y:794, status:'available' },
    { id:'t17', label:'T17', cap:2, type:'round',  x:190, y:897, status:'available' },
    { id:'t18', label:'T18', cap:6, type:'pill6',  x:400, y:897, status:'available' },
    { id:'t19', label:'T19', cap:2, type:'round',  x:610, y:897, status:'available' },
    { id:'t20', label:'T20', cap:4, type:'square', x:820, y:897, status:'filled'    },
    { id:'t21', label:'T21', cap:2, type:'stand',  x:965, y:100, status:'available' },
    { id:'t22', label:'T22', cap:2, type:'stand',  x:965, y:180, status:'reserved'  },
    { id:'t23', label:'T23', cap:2, type:'stand',  x:965, y:260, status:'available' },
    { id:'t24', label:'T24', cap:2, type:'stand',  x:965, y:340, status:'filled'    }
  ];

  function chairRect(cx, cy, c){
    return '<rect class="mt-chair" x="' + (cx + c.dx - c.w/2) + '" y="' + (cy + c.dy - c.h/2) +
      '" width="' + c.w + '" height="' + c.h + '" rx="4"></rect>';
  }
  function topShape(cx, cy, top){
    if (top.kind === 'circle')
      return '<circle class="mt-top" cx="' + cx + '" cy="' + cy + '" r="' + top.r + '"></circle>';
    return '<rect class="mt-top" x="' + (cx - top.w/2) + '" y="' + (cy - top.h/2) +
      '" width="' + top.w + '" height="' + top.h + '" rx="' + top.rx + '"></rect>';
  }
  function tableEl(t){
    const s = SHAPES[t.type], cx = t.x, cy = t.y;
    const interactive = t.status === 'available';
    const halo = '<rect class="mt-halo" x="' + (cx - s.halo.hw) + '" y="' + (cy - s.halo.hh) +
      '" width="' + (s.halo.hw*2) + '" height="' + (s.halo.hh*2) + '" rx="14"></rect>';
    const chairs = s.chairs.map(c => chairRect(cx, cy, c)).join('');
    const disc = '<circle class="mt-disc" cx="' + cx + '" cy="' + cy + '" r="' + s.disc + '"></circle>';
    const lbl = '<text class="mt-lbl" x="' + cx + '" y="' + cy + '">' + t.label + '</text>';
    const badge = t.status === 'reserved'
      ? '<text class="mt-badge" x="' + cx + '" y="' + (cy + s.halo.hh + 13) + '">Rezervat</text>' : '';
    return '<g class="mt-table s-' + t.status + '"' +
      ' data-id="' + t.id + '" data-cap="' + t.cap + '" data-status="' + t.status + '"' +
      ' role="button" tabindex="' + (interactive ? '0' : '-1') + '"' +
      ' aria-label="Masa ' + t.label + ', ' + t.cap + ' locuri, ' + RO_STATUS[t.status].toLowerCase() + '"' +
      (interactive ? '' : ' aria-disabled="true"') + '>' +
      halo + chairs + topShape(cx, cy, s.top) + disc + lbl + badge +
      '</g>';
  }

  function zoneRect(cls, x, y, w, h, rx){
    return '<rect class="mt-zone ' + cls + '" x="' + x + '" y="' + y + '" width="' + w +
      '" height="' + h + '" rx="' + (rx || 8) + '"></rect>';
  }
  function zoneLabel(cls, x, y, text, rot){
    const tr = rot ? ' transform="rotate(' + rot + ' ' + x + ' ' + y + ')"' : '';
    return '<text class="mt-zone-lbl ' + cls + '" x="' + x + '" y="' + y + '"' + tr + '>' + text + '</text>';
  }
  function renderMap(){
    root.setAttribute('viewBox', '0 0 ' + VIEWBOX.w + ' ' + VIEWBOX.h);
    root.setAttribute('preserveAspectRatio', 'xMidYMid meet');
    const zones =
      zoneRect('mt-zone-outline', 14, 14, 1092, 972, 10) +
      zoneRect('mt-zone-entrance', 40, 150, 80, 210, 8) +
        zoneLabel('sm', 80, 255, 'Entrance', -90) +
      zoneRect('mt-zone-building', 130, 150, 215, 210, 8) +
        zoneLabel('', 237, 246, 'Part of building') +
        zoneLabel('sm', 237, 268, 'cu barul') +
      zoneRect('mt-zone-dj', 360, 150, 110, 80, 8) +
        zoneLabel('', 415, 190, 'DJ') +
      zoneRect('mt-zone-bar', 485, 150, 345, 80, 8) +
        zoneLabel('bar', 657, 192, 'Bar') +
      zoneRect('mt-zone-bar', 830, 55, 75, 175, 8) +
        zoneLabel('bar', 867, 142, 'Bar', -90) +
      zoneRect('mt-zone-stand', 905, 55, 120, 305, 8) +
        zoneLabel('sm', 1008, 207, 'Stand', 90) +
      zoneRect('mt-zone-stand', 360, 235, 545, 125, 8) +
        zoneLabel('', 632, 297, 'Stand zone') +
      zoneRect('mt-zone-table', 40, 395, 1045, 575, 8) +
        zoneLabel('zone', 560, 418, 'Table zone');
    root.innerHTML = zones + '<g class="mt-map">' + TABLES.map(tableEl).join('') + '</g>';
    mapGroup = root.querySelector('.mt-map');
  }
```

- [ ] **Step 2: Temporary inventory self-check**

Inside the IIFE, immediately after `renderMap();` near the bottom, add temporarily:
```javascript
  console.assert(TABLES.length === 24, 'expected 24 tables, got ' + TABLES.length);
  console.assert(TABLES.reduce((s,t)=>s+t.cap,0) === 88, 'expected 88 seats');
  console.assert(TABLES.filter(t=>t.status==='reserved').length === 3, 'expected 3 reserved');
  console.assert(TABLES.filter(t=>t.status==='filled').length === 4, 'expected 4 filled');
```

- [ ] **Step 3: Verify in the browser**

Hard-refresh, open the console. Expected: the map now draws the zoned floor plan — terrace outline, an L-shaped dark **Bar** (gold "Bar" labels), **Entrance**, **Part of building / cu barul**, **DJ**, two **Stand zone** areas, and a large **Table zone** holding the tables. Tables look realistic: chairs (rounded rects) around each top, a colored status disc with "T##" centered — green (available), crimson (reserved, with a "Rezervat" badge under it), grey (filled). The 4 standing tables sit in the right-margin stand strip. No tables overlap. No `console.assert` errors in the console (silent = pass). Note: clicking a `filled` table may still (incorrectly) select it — that guard lands in Task 3.

- [ ] **Step 4: Remove the temporary self-check**

Delete the four `console.assert` lines from Step 2.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(tables): v2 zoned floor plan with realistic chaired tables"
```

---

### Task 3: Three-state selection guards, tooltip, and confirmation class swap

**Files:**
- Modify: `index.html` — `selectTable` guard; `wireSelection` click + keydown guards and tooltip text; `showConfirmation` table-class swap (all inside the IIFE, ~lines 2061–2146 in the current file, shifted by Task 2).

**Interfaces:**
- Consumes: `RO_STATUS` (from Task 2), `state`, `root`, `mapGroup`, `TABLES`, `selectTable`.
- Produces: `available`-only selection (both `reserved` and `filled` are non-selectable); tooltip shows the Romanian status word for non-available tables; on booking, the table's class flips `s-available`→`s-reserved` and `data-status` updates so the new CSS greys/locks it.

- [ ] **Step 1: Guard `selectTable` to available-only**

Edit — find:
```javascript
  function selectTable(t){
    if (!t || t.status === 'reserved') return;
```
Replace with:
```javascript
  function selectTable(t){
    if (!t || t.status !== 'available') return;
```

- [ ] **Step 2: Guard the click and keydown handlers**

Edit — find:
```javascript
    root.addEventListener('click', e => {
      const g = e.target.closest('.mt-table'); if (!g) return;
      const t = TABLES.find(x => x.id === g.dataset.id);
      if (t && t.status !== 'reserved') selectTable(t);
    });

    root.addEventListener('keydown', e => {
      if (e.key !== 'Enter' && e.key !== ' ' && e.key !== 'Spacebar') return;
      const g = e.target.closest('.mt-table'); if (!g) return;
      e.preventDefault();
      const t = TABLES.find(x => x.id === g.dataset.id);
      if (t && t.status !== 'reserved') selectTable(t);
    });
```
Replace with:
```javascript
    root.addEventListener('click', e => {
      const g = e.target.closest('.mt-table'); if (!g) return;
      const t = TABLES.find(x => x.id === g.dataset.id);
      if (t && t.status === 'available') selectTable(t);
    });

    root.addEventListener('keydown', e => {
      if (e.key !== 'Enter' && e.key !== ' ' && e.key !== 'Spacebar') return;
      const g = e.target.closest('.mt-table'); if (!g) return;
      e.preventDefault();
      const t = TABLES.find(x => x.id === g.dataset.id);
      if (t && t.status === 'available') selectTable(t);
    });
```

- [ ] **Step 3: Update tooltip text for three states**

Edit — find:
```javascript
        tip.textContent = t.status === 'reserved'
          ? ('Masa ' + t.label + ' · Rezervat')
          : ('Masa ' + t.label + ' · ' + t.cap + ' locuri');
```
Replace with:
```javascript
        tip.textContent = t.status === 'available'
          ? ('Masa ' + t.label + ' · ' + t.cap + ' locuri')
          : ('Masa ' + t.label + ' · ' + RO_STATUS[t.status]);
```

- [ ] **Step 4: Swap the booked table to the reserved class scheme**

Edit — find:
```javascript
    t.status = 'reserved';
    const el = root.querySelector('.mt-table[data-id="' + t.id + '"]');
    if (el){
      el.classList.remove('selected');
      el.classList.add('reserved');
      el.setAttribute('aria-disabled', 'true');
      el.setAttribute('tabindex', '-1');
    }
```
Replace with:
```javascript
    t.status = 'reserved';
    const el = root.querySelector('.mt-table[data-id="' + t.id + '"]');
    if (el){
      el.classList.remove('selected', 's-available');
      el.classList.add('s-reserved');
      el.setAttribute('aria-disabled', 'true');
      el.setAttribute('tabindex', '-1');
      el.setAttribute('data-status', 'reserved');
    }
```

- [ ] **Step 5: Verify (browser + node syntax)**

Extract the `<script>` to a temp `.js` and run `node --check` — expect no syntax errors. Then hard-refresh and verify: clicking or keyboard-activating a **filled** (grey) table does nothing; same for **reserved**; only **green** tables select (gold ring appears, others dim). Hover tooltip reads "Masa T07 · Ocupat" on a filled table, "Masa T03 · Rezervat" on reserved, and "Masa T05 · 2 locuri" on available. Complete a booking on an available table → confirmation shows `Masa T## · …`, and that table turns grey/locked (now reserved). No console errors.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(tables): three-state selection guards, tooltip, confirm swap"
```

---

## Self-Review

**Spec coverage:**
- Dark theme + realistic tables with chairs → Task 1 CSS (`.mt-chair/.mt-top/.mt-disc`) + Task 2 `SHAPES`/`tableEl`. ✓
- Zoned floor plan (Entrance, Part of building, DJ, L-shaped Bar, Stand zones, Table zone) → Task 2 `renderMap` + Task 1 zone CSS. ✓
- Three states Liber/Rezervat/Ocupat, no countdown → Task 1 `.s-*` CSS, Task 2 statuses, Task 3 guards. ✓
- Selected = gold ring; reserved+filled non-selectable → Task 1 `.mt-halo`, Task 3 guards. ✓
- 24 tables / 88 seats, exact mix, reserved T03/T11/T22, filled T07/T15/T20/T24 → Task 2 `TABLES` + temp asserts. ✓
- 4 standing tables in right-margin stand zone → Task 2 `t21..t24` type `stand`. ✓
- Reused interaction (form, validation, confirmation, mobile sheet, keyboard, aria-live) → untouched except Task 3 guards/swap. ✓
- Confirmation format `Masa T04 · …`; telefon required; 17:00–02:00; date min today → unchanged from v1. ✓
- Mobile ≤620px horizontal scroll + hint → Task 1 Step 2 + markup. ✓
- prefers-reduced-motion → existing `#tables *` rule retained. ✓

**Placeholder scan:** none — every step has complete code.

**Type/name consistency:** `SHAPES` keys (`round/square/pill6/pill8/stand`) match every `TABLES.type`. `RO_STATUS` defined in Task 2, used in Task 2 `tableEl` and Task 3 tooltip. Class names `s-available/s-reserved/s-filled`, `mt-halo/mt-chair/mt-top/mt-disc/mt-lbl/mt-badge`, zone classes, and legend `lg-av/lg-rs/lg-fl` are identical between Task 1 (CSS) and Task 2 (emitted markup). Guards switched consistently to `=== 'available'` across `selectTable`, click, and keydown in Task 3.
