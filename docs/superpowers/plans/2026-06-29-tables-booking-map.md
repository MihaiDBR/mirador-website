# Interactive Table Booking Map — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a top-down, interactive SVG map of the Mirador rooftop where a guest picks a table and fills a styled, validated reservation form — a frontend-only preview with no backend.

**Architecture:** Everything lives in the existing single `index.html` (its `<style>` block, its `<body>`, and the existing inline `<script>`). A new `#tables` section is inserted before `#reservation`. A self-contained IIFE renders the SVG map from one `TABLES` data array (single source of truth), handles selection, builds + validates the form, and shows an animated confirmation. Desktop = two-column (map + sticky form); mobile reuses the site's existing bottom-sheet pattern.

**Tech Stack:** Plain HTML + CSS + vanilla JS (no build, no framework, no dependencies). Inline SVG for the map.

## Global Constraints

- All code stays inside `D:\Mirador-website\index.html`. No new files, no libraries.
- Reuse existing CSS variables (`--bg`, `--bg-2`, `--crimson`, `--rose`, `--ivory`, `--ivory-dim`, `--gold`, `--lime`) and fonts (`Cormorant Garamond` italic for headings, `Montserrat` for UI). New CSS classes are prefixed `mt-` to avoid collisions.
- Table inventory is exactly **20 tables / 80 seats**: eight 2-seat, six 4-seat, four 6-seat, two 8-seat. Four tables start `reserved`.
- All form fields are **required, including phone**. Opening hours for the time select are **17:00 → 02:00** in 30-minute steps. Date min = today.
- UI copy is in **Romanian**. Confirmation summary format: `Masa 04 · 12 Iulie · 20:00`.
- No backend, persistence, payment, email, or SMS. Refresh resets all state.
- Respect `prefers-reduced-motion`. Custom cursor (`cursor:none`) already applies site-wide; new interactive elements must keep that consistent (they inherit it).

**Testing reality:** This is a static single-file site with no test framework. Each task is verified by serving locally and observing in the browser. To serve: from `D:\Mirador-website` run `python -m http.server 8000` (background) and open `http://localhost:8000`. Hard-refresh (Ctrl+F5) after each change. Temporary `console.assert` checks are used where noted and removed before that task's commit.

---

### Task 1: Section scaffold + navigation links

**Files:**
- Modify: `index.html` (nav `<ul class="nav-links">` ~line 1127; `.mobile-menu` ~line 1137; insert new `<section id="tables">` immediately before `<section id="reservation">` ~line 1505)

**Interfaces:**
- Produces: DOM ids `#tables`, `#table-map` (empty `<svg>`), `#mt-aside`, `#mt-panel`; nav anchors to `#tables`. Later tasks render into `#table-map` and `#mt-panel`.

- [ ] **Step 1: Add the desktop nav link**

Edit — find:
```html
    <li><a href="#events">Events</a></li>
    <li><a href="#reservation">Contact</a></li>
```
Replace with:
```html
    <li><a href="#events">Events</a></li>
    <li><a href="#tables">Tables</a></li>
    <li><a href="#reservation">Contact</a></li>
```

- [ ] **Step 2: Add the mobile-menu link**

Edit — find:
```html
  <a href="#events">Events</a>
  <a href="#reservation">Contact</a>
```
Replace with:
```html
  <a href="#events">Events</a>
  <a href="#tables">Tables</a>
  <a href="#reservation">Contact</a>
```

- [ ] **Step 3: Insert the section before `#reservation`**

Edit — find:
```html
<section id="reservation">
  <div class="res-ambient"></div>
```
Replace with:
```html
<!-- ═════════ TABLES / BOOKING MAP ═════════ -->
<section id="tables">
  <div class="mt-head reveal">
    <p class="tag">The Floor</p>
    <h2>Alege-ți <em>masa</em></h2>
    <p>O hartă a terasei, văzută de sus. Atinge o masă liberă și rezervă-ți locul cu vedere spre oraș. Aceasta este o previzualizare — momentan fără sistem de rezervări activ.</p>
  </div>
  <div class="mt-wrap">
    <div class="mt-map-col reveal rd-1">
      <div class="mt-map-frame">
        <svg id="table-map" role="group" aria-label="Hartă mese Mirador"></svg>
      </div>
      <div class="mt-legend">
        <span><i></i> Disponibil</span>
        <span><i class="res"></i> Rezervat</span>
        <span>2 · 4 · 6 · 8 locuri</span>
      </div>
    </div>
    <div class="mt-aside" id="mt-aside">
      <div class="mt-aside-backdrop"></div>
      <div class="mt-panel" id="mt-panel">
        <div class="mt-panel-empty">Alege o masă de pe hartă pentru a rezerva.</div>
      </div>
    </div>
  </div>
</section>

<section id="reservation">
  <div class="res-ambient"></div>
```

- [ ] **Step 4: Verify in the browser**

Serve and hard-refresh. Expected: clicking "Tables" in the nav smooth-scrolls to the new section; the eyebrow "The Floor", italic title "Alege-ți masa", and intro paragraph are visible and reveal on scroll; the empty `#table-map` area shows the framed box with the "Alege o masă…" placeholder beside it (unstyled is fine — styling is Task 2). No console errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(tables): add booking section scaffold and nav links"
```

---

### Task 2: Styles for the section, map frame, legend, and form panel

**Files:**
- Modify: `index.html` (insert a CSS block immediately before `</style>` ~line 1101)

**Interfaces:**
- Consumes: class names emitted by later tasks (`.mt-table`, `.mt-disc`, `.mt-lbl`, `.mt-outline`, `.mt-bar`, `.mt-map.has-selection`, `.mt-form`, `.mt-field`, `.mt-confirm`, `.mt-tip`, `.mt-aside.open`).
- Produces: full visual styling + responsive behaviour (desktop two-column / mobile bottom-sheet) for the feature.

- [ ] **Step 1: Insert the CSS block before `</style>`**

Edit — find:
```css
}
</style>
```
Replace with (note: this matches the final `}` of the mobile menu-row media query, then `</style>`):
```css
}

/* ═════════ TABLE BOOKING MAP ═════════ */
#tables{ padding:160px 48px; position:relative; overflow:hidden; }
.mt-head{ max-width:1400px; margin:0 auto 56px; }
.mt-head .tag{ font-size:.58rem; font-weight:300; letter-spacing:.5em; text-transform:uppercase;
  color:var(--rose); display:flex; align-items:center; gap:14px; margin-bottom:20px; }
.mt-head .tag::before{ content:''; width:50px; height:1px; background:var(--rose); }
.mt-head h2{ font-family:'Cormorant Garamond',serif; font-style:italic;
  font-size:clamp(2.6rem,5vw,4.4rem); font-weight:300; line-height:1.05; color:var(--ivory); }
.mt-head h2 em{ color:var(--crimson); }
.mt-head p{ font-size:.7rem; font-weight:200; letter-spacing:.12em; color:var(--ivory-dim);
  line-height:2; max-width:560px; margin-top:18px; }

.mt-wrap{ max-width:1400px; margin:0 auto; display:grid; grid-template-columns:1.5fr 1fr;
  gap:40px; align-items:start; }
.mt-map-col{ position:relative; }
.mt-map-frame{ border:1px solid rgba(196,160,96,.18); padding:18px; background:
  radial-gradient(ellipse 70% 60% at 50% 30%, rgba(120,60,180,.08) 0%, transparent 60%),
  linear-gradient(160deg, #0b0712 0%, #07040d 60%, #100618 100%); }
#table-map{ width:100%; height:auto; display:block; }

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

.mt-aside{ position:relative; }
.mt-aside-backdrop{ display:none; }
.mt-panel{ border:1px solid rgba(196,160,96,.2); background:linear-gradient(180deg,#0d0712,#08040d);
  padding:30px; position:sticky; top:100px; }
.mt-panel-empty{ font-size:.66rem; letter-spacing:.18em; color:var(--ivory-dim);
  text-align:center; padding:46px 10px; line-height:2; }
.mt-chip{ display:inline-block; font-family:'Cormorant Garamond',serif; font-style:italic;
  font-size:1.2rem; color:var(--ivory); border:1px solid var(--crimson); padding:8px 16px;
  margin-bottom:22px; }
.mt-form{ display:flex; flex-direction:column; gap:16px; }
.mt-field{ display:flex; flex-direction:column; gap:7px; }
.mt-field span{ font-size:.5rem; letter-spacing:.35em; text-transform:uppercase; color:var(--gold); }
.mt-field input, .mt-field select{ background:rgba(237,227,216,.04);
  border:1px solid rgba(237,227,216,.12); padding:13px 14px; color:var(--ivory);
  font-family:'Montserrat',sans-serif; font-size:.7rem; font-weight:200; letter-spacing:.08em;
  outline:none; transition:border-color .3s, background .3s; }
.mt-field input::placeholder{ color:rgba(237,227,216,.3); }
.mt-field select option{ background:#0d0712; color:var(--ivory); }
.mt-field input:focus, .mt-field select:focus{ border-color:var(--crimson);
  background:rgba(155,31,53,.08); }
.mt-field.invalid input, .mt-field.invalid select{ border-color:var(--crimson); }
.mt-err{ font-size:.5rem; letter-spacing:.1em; color:var(--rose); min-height:.7em; }
.mt-submit{ margin-top:6px; background:var(--crimson); border:1px solid var(--crimson);
  color:var(--ivory); font-family:'Montserrat',sans-serif; font-size:.58rem; font-weight:300;
  letter-spacing:.4em; text-transform:uppercase; padding:16px; transition:background .4s, border-color .4s; }
.mt-submit:hover{ background:var(--rose); border-color:var(--rose); }

.mt-confirm{ text-align:center; padding:10px 0; animation:mtFade .6s cubic-bezier(.16,1,.3,1); }
@keyframes mtFade{ from{ opacity:0; transform:translateY(16px); } to{ opacity:1; transform:translateY(0); } }
.mt-check{ width:54px; height:54px; margin:0 auto 18px; border-radius:50%; border:1px solid var(--gold);
  display:flex; align-items:center; justify-content:center; color:var(--gold); font-size:1.4rem; }
.mt-confirm h3{ font-family:'Cormorant Garamond',serif; font-style:italic; font-weight:400;
  font-size:1.7rem; color:var(--ivory); margin-bottom:10px; }
.mt-summary{ font-size:.62rem; letter-spacing:.2em; text-transform:uppercase; color:var(--rose); margin-bottom:16px; }
.mt-note{ font-size:.54rem; letter-spacing:.12em; color:var(--ivory-dim); line-height:1.8; margin-bottom:24px; }
.mt-reset{ font-family:'Montserrat',sans-serif; font-size:.54rem; letter-spacing:.32em;
  text-transform:uppercase; color:var(--ivory-dim); border:1px solid rgba(237,227,216,.2);
  padding:13px 26px; transition:color .3s, border-color .3s; }
.mt-reset:hover{ color:var(--ivory); border-color:var(--rose); }

@media (max-width:1024px){ #tables{ padding:110px 28px; } }
@media (max-width:820px){
  #tables{ padding:90px 22px; }
  .mt-wrap{ grid-template-columns:1fr; gap:0; }
  .mt-aside{ position:fixed; inset:0; z-index:300; visibility:hidden; }
  .mt-aside.open{ visibility:visible; }
  .mt-aside-backdrop{ display:block; position:absolute; inset:0; background:rgba(5,3,10,.72);
    backdrop-filter:blur(6px); opacity:0; transition:opacity .45s; }
  .mt-aside.open .mt-aside-backdrop{ opacity:1; }
  .mt-panel{ position:absolute; left:0; right:0; bottom:0; top:auto; border-radius:24px 24px 0 0;
    border-bottom:none; transform:translateY(100%); transition:transform .5s cubic-bezier(.16,1,.3,1);
    max-height:88vh; overflow-y:auto; padding:24px 22px calc(28px + env(safe-area-inset-bottom)); }
  .mt-aside.open .mt-panel{ transform:translateY(0); }
  .mt-panel-empty{ display:none; }
}
@media (prefers-reduced-motion:reduce){
  .mt-table .mt-disc, .mt-panel, .mt-confirm, .mt-aside .mt-panel{ transition:none; animation:none; }
}
</style>
```

- [ ] **Step 2: Verify in the browser**

Hard-refresh. Expected: the section now has proper padding/rhythm matching other sections; the map frame is a dark bordered box; the legend row shows two dots (solid + dashed) and "2 · 4 · 6 · 8 locuri"; on desktop the layout is two columns (frame left ~60%, panel right ~40%) with the panel showing the centered "Alege o masă…" placeholder. Resize below 820px: layout stacks to one column and the panel is hidden (it becomes a bottom-sheet, driven later). No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "style(tables): add booking map, legend, and form panel styles"
```

---

### Task 3: Render the SVG map from the TABLES data

**Files:**
- Modify: `index.html` (insert a new IIFE in the inline `<script>`, immediately before `</script>` ~line 1815, after the `updateParallax();` line)

**Interfaces:**
- Consumes: global `isFinePointer` (declared earlier in the same script).
- Produces: inside the IIFE closure — `TABLES` (array of `{id,label,cap,x,y,status}`), `RADIUS` map, `VIEWBOX`, `renderMap()`, and module refs `root` (`#table-map`), `mapGroup` (`.mt-map`), `aside` (`#mt-aside`), `panel` (`#mt-panel`). After render, the SVG contains one `<g class="mt-table" data-id data-cap role="button" tabindex aria-label>` per table with a `.mt-disc` circle + `.mt-lbl` text. Later tasks attach behaviour by querying `.mt-table` and reading `data-id`.

- [ ] **Step 1: Insert the IIFE before `</script>`**

Edit — find:
```javascript
addEventListener('scroll', () => { if(!pTick){ requestAnimationFrame(updateParallax); pTick = true; } }, { passive:true });
updateParallax();
</script>
```
Replace with:
```javascript
addEventListener('scroll', () => { if(!pTick){ requestAnimationFrame(updateParallax); pTick = true; } }, { passive:true });
updateParallax();

/* ═════════ TABLE BOOKING MAP (frontend-only preview) ═════════ */
(function initTableBooking(){
  const root = document.getElementById('table-map');
  const aside = document.getElementById('mt-aside');
  const panel = document.getElementById('mt-panel');
  if (!root || !aside || !panel) return;

  const VIEWBOX = { w:1000, h:700 };
  const RADIUS = { 2:18, 4:24, 6:30, 8:38 };

  // Single source of truth — 20 tables / 80 seats (8×2, 6×4, 4×6, 2×8); 4 start reserved.
  const TABLES = [
    { id:'t1',  label:'01', cap:4, x:360, y:130, status:'available' },
    { id:'t2',  label:'02', cap:2, x:500, y:120, status:'available' },
    { id:'t3',  label:'03', cap:2, x:620, y:120, status:'reserved'  },
    { id:'t4',  label:'04', cap:4, x:740, y:125, status:'available' },
    { id:'t5',  label:'05', cap:2, x:880, y:150, status:'available' },
    { id:'t6',  label:'06', cap:6, x:150, y:300, status:'available' },
    { id:'t7',  label:'07', cap:2, x:330, y:290, status:'available' },
    { id:'t8',  label:'08', cap:4, x:480, y:280, status:'reserved'  },
    { id:'t9',  label:'09', cap:6, x:640, y:285, status:'available' },
    { id:'t10', label:'10', cap:4, x:820, y:300, status:'available' },
    { id:'t11', label:'11', cap:4, x:150, y:470, status:'available' },
    { id:'t12', label:'12', cap:6, x:320, y:460, status:'available' },
    { id:'t13', label:'13', cap:8, x:500, y:450, status:'available' },
    { id:'t14', label:'14', cap:6, x:680, y:455, status:'reserved'  },
    { id:'t15', label:'15', cap:2, x:860, y:470, status:'available' },
    { id:'t16', label:'16', cap:2, x:250, y:600, status:'available' },
    { id:'t17', label:'17', cap:4, x:420, y:605, status:'available' },
    { id:'t18', label:'18', cap:8, x:590, y:600, status:'available' },
    { id:'t19', label:'19', cap:2, x:760, y:605, status:'available' },
    { id:'t20', label:'20', cap:2, x:900, y:600, status:'reserved'  }
  ];

  const state = { selected:null };
  let mapGroup = null;

  function tableEl(t){
    const r = RADIUS[t.cap];
    const reserved = t.status === 'reserved';
    return '<g class="mt-table' + (reserved ? ' reserved' : '') + '"' +
      ' data-id="' + t.id + '" data-cap="' + t.cap + '"' +
      ' role="button" tabindex="' + (reserved ? '-1' : '0') + '"' +
      ' aria-label="Masa ' + t.label + ', ' + t.cap + ' locuri, ' +
        (reserved ? 'rezervat' : 'disponibil') + '"' +
      (reserved ? ' aria-disabled="true"' : '') + '>' +
      '<circle class="mt-disc" cx="' + t.x + '" cy="' + t.y + '" r="' + r + '"></circle>' +
      '<text class="mt-lbl" x="' + t.x + '" y="' + t.y + '">' + t.label + '</text>' +
      '</g>';
  }

  function renderMap(){
    root.setAttribute('viewBox', '0 0 ' + VIEWBOX.w + ' ' + VIEWBOX.h);
    root.innerHTML =
      '<rect class="mt-outline" x="40" y="40" width="920" height="620" rx="26"></rect>' +
      '<rect class="mt-bar" x="60" y="60" width="240" height="140" rx="10"></rect>' +
      '<text class="mt-bar-lbl" x="180" y="138">Bar</text>' +
      '<text class="mt-view-lbl" x="520" y="30">← City View →</text>' +
      '<text class="mt-view-lbl" x="986" y="350" transform="rotate(90 986 350)">← City View →</text>' +
      '<path class="mt-sprig" d="M70 230 q14 -20 34 -12 M104 218 q-6 -22 -34 -12"></path>' +
      '<path class="mt-sprig" d="M930 560 q-14 -20 -34 -12 M896 548 q6 -22 34 -12"></path>' +
      '<path class="mt-sprig" d="M70 640 q18 -16 36 -6 M106 634 q-2 -24 -36 -6"></path>' +
      '<g class="mt-map">' + TABLES.map(tableEl).join('') + '</g>';
    mapGroup = root.querySelector('.mt-map');
  }

  renderMap();
})();
</script>
```

- [ ] **Step 2: Temporary self-check for inventory**

Inside the IIFE, immediately after `renderMap();`, add temporarily:
```javascript
  console.assert(TABLES.length === 20, 'expected 20 tables, got ' + TABLES.length);
  console.assert(TABLES.reduce((s,t)=>s+t.cap,0) === 80, 'expected 80 seats');
  console.assert(TABLES.filter(t=>t.status==='reserved').length === 4, 'expected 4 reserved');
```

- [ ] **Step 3: Verify in the browser**

Hard-refresh, open devtools console. Expected: the map draws a rounded terrace outline (gold), a "Bar" block top-left, "City View" labels on the top and right edges, faint green sprigs at corners, and 20 numbered table circles sized by capacity (8-seat clearly largest). Four tables (03, 08, 14, 20) appear greyed with a dashed ring. No `console.assert` failures appear (assertions are silent when passing — verify there are NO assertion error lines).

- [ ] **Step 4: Remove the temporary self-check**

Delete the three `console.assert` lines added in Step 2.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(tables): render SVG floor map from table data"
```

---

### Task 4: Selection, hover tooltip, and form build

**Files:**
- Modify: `index.html` (extend the `initTableBooking` IIFE: add helper/builder functions and a `wireSelection()` call before the IIFE closes)

**Interfaces:**
- Consumes: `root`, `panel`, `aside`, `mapGroup`, `state`, `TABLES`, `RADIUS`, global `isFinePointer`.
- Produces: `pad2()`, `todayISO()`, `RO_MONTHS`, `formatRoDate(iso)`, `timeSlots()`, `buildFormHTML(table)`, `selectTable(table)`, `isMobile()`. After selecting an available table, `#mt-panel` contains `<form id="mt-form">` with fields named `nume`, `telefon`, `data`, `ora`, `persoane`, an `#mt-err` div, and a `.mt-submit` button. Submit handling is Task 5.

- [ ] **Step 1: Add helpers, form builder, and selection — before the IIFE's closing `})();`**

Edit — find (the end of the IIFE from Task 3):
```javascript
  renderMap();
})();
</script>
```
Replace with:
```javascript
  function pad2(n){ return String(n).padStart(2,'0'); }
  function todayISO(){ const d = new Date();
    return d.getFullYear() + '-' + pad2(d.getMonth()+1) + '-' + pad2(d.getDate()); }
  const RO_MONTHS = ['Ianuarie','Februarie','Martie','Aprilie','Mai','Iunie',
    'Iulie','August','Septembrie','Octombrie','Noiembrie','Decembrie'];
  function formatRoDate(iso){ const p = iso.split('-').map(Number);
    return p[2] + ' ' + RO_MONTHS[p[1]-1]; }
  function timeSlots(){ const out = []; for (let m = 17*60; m <= 26*60; m += 30){
    out.push(pad2(Math.floor(m/60)%24) + ':' + pad2(m%60)); } return out; }
  function isMobile(){ return window.matchMedia('(max-width:820px)').matches; }

  function buildFormHTML(t){
    const slots = timeSlots().map(s => '<option value="' + s + '">' + s + '</option>').join('');
    let people = '';
    for (let i = 1; i <= t.cap; i++){
      people += '<option value="' + i + '">' + i + (i === 1 ? ' persoană' : ' persoane') + '</option>';
    }
    return '' +
      '<form class="mt-form" id="mt-form" novalidate>' +
        '<div class="mt-chip">Masa ' + t.label + ' · ' + t.cap + ' locuri</div>' +
        '<label class="mt-field"><span>Nume</span>' +
          '<input type="text" name="nume" required autocomplete="name" placeholder="Numele tău"></label>' +
        '<label class="mt-field"><span>Telefon</span>' +
          '<input type="tel" name="telefon" required autocomplete="tel" placeholder="07xx xxx xxx"></label>' +
        '<label class="mt-field"><span>Data</span>' +
          '<input type="date" name="data" required min="' + todayISO() + '"></label>' +
        '<label class="mt-field"><span>Ora</span>' +
          '<select name="ora" required><option value="" disabled selected>Alege ora</option>' + slots + '</select></label>' +
        '<label class="mt-field"><span>Persoane</span>' +
          '<select name="persoane" required><option value="" disabled selected>Câte persoane</option>' + people + '</select></label>' +
        '<div class="mt-err" id="mt-err" role="alert"></div>' +
        '<button type="submit" class="mt-submit">Confirmă rezervarea</button>' +
      '</form>';
  }

  function selectTable(t){
    if (!t || t.status === 'reserved') return;
    state.selected = t;
    mapGroup.classList.add('has-selection');
    root.querySelectorAll('.mt-table').forEach(el =>
      el.classList.toggle('selected', el.dataset.id === t.id));
    panel.innerHTML = buildFormHTML(t);
    const first = panel.querySelector('input[name="nume"]');
    if (first) first.focus();
  }

  function wireSelection(){
    root.addEventListener('click', e => {
      const g = e.target.closest('.mt-table'); if (!g) return;
      const t = TABLES.find(x => x.id === g.dataset.id);
      if (t && t.status !== 'reserved') selectTable(t);
    });

    if (isFinePointer){
      const tip = document.createElement('div');
      tip.className = 'mt-tip';
      document.body.appendChild(tip);
      root.addEventListener('mouseover', e => {
        const g = e.target.closest('.mt-table'); if (!g) return;
        const t = TABLES.find(x => x.id === g.dataset.id); if (!t) return;
        tip.textContent = t.status === 'reserved'
          ? ('Masa ' + t.label + ' · Rezervat')
          : ('Masa ' + t.label + ' · ' + t.cap + ' locuri');
        tip.classList.add('show');
      });
      root.addEventListener('mousemove', e => {
        tip.style.left = e.clientX + 'px';
        tip.style.top = e.clientY + 'px';
      });
      root.addEventListener('mouseout', e => {
        if (!e.relatedTarget || !e.relatedTarget.closest('.mt-table')) tip.classList.remove('show');
      });
    }
  }

  renderMap();
  wireSelection();
})();
</script>
```

- [ ] **Step 2: Verify in the browser (desktop width)**

Hard-refresh at desktop width. Expected: hovering an available table glows it crimson and shows a tooltip near the cursor ("Masa 05 · 2 locuri"); hovering a reserved table shows "Masa 03 · Rezervat" and does not glow. Clicking an available table fills it crimson, dims the other available tables, and the right panel shows the form with chip "Masa NN · C locuri". The **Persoane** dropdown lists exactly 1..capacity (e.g. table 13 → up to 8; a 2-seat → up to 2). The **Ora** dropdown runs 17:00 → 02:00 in 30-min steps (19 options). The **Data** field cannot pick a past date. Clicking a reserved table does nothing. (Submit does nothing yet — that's Task 5.)

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(tables): table selection, hover tooltip, and reservation form"
```

---

### Task 5: Validation, confirmation, and reset

**Files:**
- Modify: `index.html` (extend the IIFE: add submit/reset handling and a `wireForm()` call)

**Interfaces:**
- Consumes: `panel`, `root`, `mapGroup`, `state`, `TABLES`, `todayISO()`, `formatRoDate()`, `selectTable` context.
- Produces: `wireForm()`, `showConfirmation(form)`, `resetPanel()`. On valid submit, the panel is replaced by `.mt-confirm` and the booked table flips to `reserved`. `resetPanel()` clears selection back to the empty placeholder.

- [ ] **Step 1: Add validation/confirmation/reset — before `renderMap();`**

Edit — find:
```javascript
  renderMap();
  wireSelection();
})();
```
Replace with:
```javascript
  function resetPanel(){
    state.selected = null;
    mapGroup.classList.remove('has-selection');
    root.querySelectorAll('.mt-table.selected').forEach(el => el.classList.remove('selected'));
    panel.innerHTML = '<div class="mt-panel-empty">Alege o masă de pe hartă pentru a rezerva.</div>';
  }

  function showConfirmation(f){
    const t = state.selected;
    const summary = 'Masa ' + t.label + ' · ' +
      formatRoDate(f.elements['data'].value) + ' · ' + f.elements['ora'].value;
    panel.innerHTML =
      '<div class="mt-confirm" role="status" aria-live="polite">' +
        '<div class="mt-check">✓</div>' +
        '<h3>Rezervare trimisă</h3>' +
        '<p class="mt-summary">' + summary + '</p>' +
        '<p class="mt-note">Aceasta este o previzualizare demo — momentan nu există un sistem de ' +
          'rezervări activ în spate. Într-o versiune finală, masa ar fi confirmată automat.</p>' +
        '<button class="mt-reset" type="button">Alege altă masă</button>' +
      '</div>';
    // Flip the booked table to reserved for realism (session only).
    t.status = 'reserved';
    const el = root.querySelector('.mt-table[data-id="' + t.id + '"]');
    if (el){
      el.classList.remove('selected');
      el.classList.add('reserved');
      el.setAttribute('aria-disabled', 'true');
      el.setAttribute('tabindex', '-1');
    }
    mapGroup.classList.remove('has-selection');
  }

  function wireForm(){
    panel.addEventListener('submit', e => {
      if (!e.target || e.target.id !== 'mt-form') return;
      e.preventDefault();
      const f = e.target;
      const err = f.querySelector('#mt-err');
      f.querySelectorAll('.mt-field').forEach(x => x.classList.remove('invalid'));
      const order = ['nume','telefon','data','ora','persoane'];
      for (const name of order){
        const ctrl = f.elements[name];
        const val = (ctrl.value || '').trim();
        if (!val){
          ctrl.closest('.mt-field').classList.add('invalid');
          err.textContent = 'Te rugăm completează toate câmpurile.';
          ctrl.focus();
          return;
        }
      }
      if (f.elements['data'].value < todayISO()){
        f.elements['data'].closest('.mt-field').classList.add('invalid');
        err.textContent = 'Alege o dată din viitor.';
        f.elements['data'].focus();
        return;
      }
      err.textContent = '';
      showConfirmation(f);
    });

    panel.addEventListener('click', e => {
      if (e.target.classList && e.target.classList.contains('mt-reset')) resetPanel();
    });
  }

  renderMap();
  wireSelection();
  wireForm();
})();
```

- [ ] **Step 2: Verify in the browser**

Hard-refresh. Select an available table. Click **Confirmă rezervarea** with empty fields → the first empty field (Nume) gets a crimson border and focus, and `#mt-err` reads "Te rugăm completează toate câmpurile." Leave **Telefon** blank but fill the rest → submit blocks on Telefon (confirms phone is required). Fill all fields validly → the panel animates to the confirmation card showing e.g. "Masa 04 · 12 Iulie · 20:00", a demo note, and an "Alege altă masă" button; the table you booked is now greyed/dashed on the map. Click "Alege altă masă" → returns to the empty placeholder, selection cleared. No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(tables): form validation, confirmation, and reset"
```

---

### Task 6: Mobile bottom-sheet behaviour

**Files:**
- Modify: `index.html` (extend the IIFE: open/close the aside on mobile, wire backdrop + Escape; close on reset)

**Interfaces:**
- Consumes: `aside`, `panel`, `state`, `isMobile()`, `selectTable`, `resetPanel`.
- Produces: `openAside()`, `closeAside()`; `selectTable` opens the sheet on mobile; `resetPanel` closes it on mobile; backdrop click and Escape close it.

- [ ] **Step 1: Add open/close helpers and hook them in**

Edit — find:
```javascript
    panel.innerHTML = buildFormHTML(t);
    const first = panel.querySelector('input[name="nume"]');
    if (first) first.focus();
  }
```
Replace with:
```javascript
    panel.innerHTML = buildFormHTML(t);
    if (isMobile()) openAside();
    const first = panel.querySelector('input[name="nume"]');
    if (first) first.focus();
  }

  function openAside(){ aside.classList.add('open'); document.body.style.overflow = 'hidden'; }
  function closeAside(){ aside.classList.remove('open'); document.body.style.overflow = ''; }
```

- [ ] **Step 2: Close the sheet on reset (mobile)**

Edit — find:
```javascript
    panel.innerHTML = '<div class="mt-panel-empty">Alege o masă de pe hartă pentru a rezerva.</div>';
  }
```
Replace with:
```javascript
    panel.innerHTML = '<div class="mt-panel-empty">Alege o masă de pe hartă pentru a rezerva.</div>';
    if (isMobile()) closeAside();
  }
```

- [ ] **Step 3: Wire backdrop + Escape — inside `wireForm()`, after the click handler**

Edit — find:
```javascript
    panel.addEventListener('click', e => {
      if (e.target.classList && e.target.classList.contains('mt-reset')) resetPanel();
    });
  }
```
Replace with:
```javascript
    panel.addEventListener('click', e => {
      if (e.target.classList && e.target.classList.contains('mt-reset')) resetPanel();
    });

    const backdrop = aside.querySelector('.mt-aside-backdrop');
    if (backdrop) backdrop.addEventListener('click', closeAside);
    addEventListener('keydown', e => {
      if (e.key === 'Escape' && aside.classList.contains('open')) closeAside();
    });
  }
```

- [ ] **Step 4: Verify in the browser (mobile width)**

Resize to ≤820px (or device toolbar). Expected: the map is full-width; tapping an available table slides a bottom-sheet up from the bottom with the form; the page behind is scroll-locked; tapping the dimmed backdrop or pressing Escape slides it away; submitting a valid form shows the confirmation inside the sheet, and "Alege altă masă" closes the sheet and clears selection. Resize back to desktop: panel returns to the sticky right column. No console errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(tables): mobile bottom-sheet for the booking form"
```

---

### Task 7: Keyboard accessibility + focus management

**Files:**
- Modify: `index.html` (extend the IIFE: keyboard activation of tables + focus into the panel)

**Interfaces:**
- Consumes: `root`, `panel`, `TABLES`, `selectTable`.
- Produces: keydown handler on `root` so Enter/Space on a focused table selects it. (ARIA roles/labels were emitted in Task 3; the confirmation already uses `aria-live`.)

- [ ] **Step 1: Add keyboard activation — inside `wireSelection()`, after the click handler**

Edit — find:
```javascript
    root.addEventListener('click', e => {
      const g = e.target.closest('.mt-table'); if (!g) return;
      const t = TABLES.find(x => x.id === g.dataset.id);
      if (t && t.status !== 'reserved') selectTable(t);
    });
```
Replace with:
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

- [ ] **Step 2: Verify in the browser**

Hard-refresh at desktop width. Tab into the map: each available table is focusable (visible focus ring from the site's `:focus-visible` rule) and reserved tables are skipped. Press Enter or Space on a focused table → it selects and focus moves to the **Nume** input. Use a screen reader or inspect: each table's `aria-label` reads "Masa NN, C locuri, disponibil/rezervat"; the confirmation card is announced (it has `role="status" aria-live="polite"`). Toggle OS "reduce motion" → table/panel transitions and the confirm animation are disabled. No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(tables): keyboard activation and focus management"
```

---

## Self-Review

**Spec coverage:**
- Placement before `#reservation` + nav link → Task 1. ✓
- SVG line-art map (outline, bar, city-view labels, sprigs) → Task 3. ✓
- 20 tables / 80 seats with the exact 8/6/4/2 mix + 4 reserved → Task 3 data + temporary assert. ✓
- Table states available/hover/selected/reserved + dim-others → Task 2 CSS + Task 4. ✓
- Click → form card (desktop) / bottom-sheet (mobile) → Tasks 4 + 6. ✓
- Form fields: Masă chip, Nume, **Telefon required**, Dată (min today), Oră 17:00–02:00, Nr. persoane 1..capacity → Task 4. ✓
- Validation blocks submit + focuses first invalid; phone required → Task 5. ✓
- Animated confirmation `Masa 04 · 12 Iulie · 20:00` + demo note; booked table flips reserved; reset → Task 5. ✓
- Responsive desktop two-column / mobile sheet → Tasks 2 + 6. ✓
- Accessibility (focusable tables, Enter/Space, aria-label, aria-live, focus move) → Tasks 3 + 7. ✓
- `prefers-reduced-motion` → Task 2. ✓
- No backend / no persistence → inherent (no network code). ✓

**Placeholder scan:** No TBD/TODO; every code step contains complete code.

**Type/name consistency:** `renderMap`, `wireSelection`, `wireForm`, `selectTable`, `resetPanel`, `showConfirmation`, `openAside`, `closeAside`, `isMobile`, `buildFormHTML`, `timeSlots`, `formatRoDate`, `todayISO`, `pad2` — all defined once and referenced consistently. Field names `nume/telefon/data/ora/persoane` consistent between Task 4 (build) and Task 5 (validate). DOM ids `#table-map/#mt-aside/#mt-panel` consistent across Tasks 1–6. CSS classes match emitted markup.
