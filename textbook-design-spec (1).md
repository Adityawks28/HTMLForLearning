# The Interactive Textbook System — Complete Design & Architecture Specification

*A reusable blueprint for building single-file, college-depth, visual, interactive HTML textbooks in the "Signal & Field" style. Hand this whole document to an author (human or model) and they should be able to produce a new layer that is visually and pedagogically indistinguishable from the existing ones.*

---

## 0. How to use this document

This spec has two jobs. **As a reference**, sections 1–10 describe every design token, component, and convention precisely enough to reproduce them. **As a prompt**, section 12 is a fill-in-the-blanks template you can paste directly into a new build session along with this file.

The non-negotiables — the things that make all the layers feel like one book — are: the **Signal & Field** color/type system (§2), the **box taxonomy** (§5), the **canvas demo engine** (§8), the **KaTeX setup including the load-race fix** (§7), and the **intuition-first writing voice** (§6). Everything else is adjustable per topic.

---

## 1. Mission & philosophy

The goal is a textbook that a motivated beginner can read end to end and come away with both **mechanical fluency** (able to do the math) and **load-bearing intuition** (able to feel why the math had to be that way). Three principles drive every decision:

**Derive, don't assert.** Every important formula is built up from something simpler in a numbered, pen-and-paper derivation. The reader should never be asked to accept a result on faith when a five-line argument is available.

**Intuition is a first-class citizen, not a garnish.** Each formal idea is paired with a plain-language picture — an analogy, a thought experiment, a physical metaphor. A reader who skips every equation should still finish each section understanding what happened and why it matters.

**See it move.** Wherever a concept has spatial, dynamic, or statistical structure, there is an interactive canvas demo the reader can drag, toggle, and run. Static prose explains; the demo convinces.

The artifact is always a **single self-contained HTML file** — one file, no build step, no local assets, openable by double-click. Only KaTeX is loaded from a CDN. This portability is a hard requirement.

---

## 2. The "Signal & Field" design system

The name encodes the core visual rule: the **field** is a calm, paper-toned reading surface in indigo-on-off-white; the **signal** is a single saturated coral reserved *exclusively* for things the reader can interact with. Coral never appears as decoration. If it's coral, you can touch it. This one discipline is what makes the interactivity legible.

### 2.1 Color tokens (CSS custom properties on `:root`)

```css
:root{
  --paper:#F4F5F8;        /* page background — soft, not white */
  --surface:#FFFFFF;      /* cards, boxes, demos */
  --ink:#14162A;          /* primary text — near-black indigo */
  --ink-soft:#474B66;     /* secondary text, intros */
  --ink-faint:#878BA6;    /* captions, axis labels */
  --indigo:#4338CA;       /* primary accent — structure, links, curves */
  --indigo-deep:#2A2090;  /* emphasis, defined terms, readout numbers */
  --indigo-wash:#EEEEF8;  /* definition-box fill, hover states */
  --coral:#FF5C49;        /* THE INTERACTIVE SIGNAL — controls only */
  --coral-deep:#E8412E;   /* coral hover/active, "danger"/divergence */
  --teal:#0E9C92;         /* intuition boxes, secondary data series */
  --teal-wash:#E4F3F1;
  --amber:#C98314;        /* forward-reference / "where you'll meet this" notes */
  --amber-wash:#FBF1DC;
  --tint:#EDEEF7;         /* readout backgrounds, inline code, chips */
  --line:#DBDDE8;         /* borders, grid lines */
  --line-soft:#E7E8F0;    /* lighter internal dividers */
  --shadow:0 1px 2px rgba(20,22,42,.04),0 8px 30px rgba(20,22,42,.06);
  --shadow-lg:0 4px 12px rgba(20,22,42,.08),0 20px 50px rgba(20,22,42,.10);
}
```

The JS demos use a parallel `C = {...}` object holding the same hex values (canvas can't read CSS variables conveniently), so keep the two in sync.

### 2.2 Typography

Three families, each with one job. Load from Google Fonts in `<head>`.

```css
--serif:"Newsreader",Georgia,serif;          /* body text — long-form reading */
--display:"Space Grotesk",system-ui,sans-serif; /* headings, labels, UI, nav */
--mono:"JetBrains Mono",ui-monospace,monospace; /* code, readouts, axis numbers, kickers */
```

Body copy is set in the serif at a generous size (≈19px / line-height 1.7) for sustained reading. Everything structural or numeric — headings, control labels, the chapter "kicker," readout panels, axis ticks — is sans or mono. The serif/sans contrast is a big part of the "real textbook" feel; don't collapse it to one font.

### 2.3 Sizing tokens

```css
--reading:760px;   /* max width of the reading column — optimal line length */
--wide:1000px;     /* hero + any full-bleed wide content */
--sidebar:280px;   /* fixed TOC width (becomes 0 on mobile) */
```

### 2.4 Misc surface rules

Page background is `--paper`, never pure white; cards are `--surface` white so they lift off the page. Border radius is generous (14–16px on boxes/demos, 8–10px on small elements). Shadows are soft and double-layered (see tokens). `::selection` is coral on white. Inline `code` sits on `--tint` with `--indigo-deep` text.

---

## 3. Document skeleton

```
<!DOCTYPE html><html lang="en"><head>
  meta charset + viewport
  <title>…</title>
  Google Fonts preconnect + stylesheet (Newsreader, Space Grotesk, JetBrains Mono)
  KaTeX CSS + two deferred KaTeX scripts (see §7)
  <style> … entire design system, ~300 lines … </style>
</head><body>
  <div id="progress"></div>                  <!-- scroll progress bar -->
  <button class="menu-btn" id="menuBtn">☰</button>  <!-- mobile TOC toggle -->
  <aside id="sidebar"> brand + nav.toc + side-foot </aside>
  <div id="scrim"></div>                      <!-- mobile dim behind drawer -->
  <div class="content">
    <header class="hero"> canvas#heroCanvas + hero-inner </header>
    <section class="wrap" id="preface"> … </section>
    <section class="wrap chapter" id="ch1"> … </section>
    … one <section> per chapter …
  </div>
  <script> … one DOMContentLoaded handler: KaTeX init, nav, hero, demos … </script>
</body></html>
```

Every chapter is a `<section class="wrap chapter" id="chN">`. The `.wrap` class centers content in the 760px reading column. Sub-sections within a chapter are plain `<h3 id="…">` anchors that the scrollspy tracks.

---

## 4. Layout & navigation

**Fixed sidebar (left, 280px).** Contains: a brand block (`L#` mark + title + "Layer N of 6"), the table of contents (`ul.toc` with `.chap` chapter links each holding a two-digit `.num`, and nested `ul.subs` sub-section links), and a `side-foot` note linking to the adjacent layer.

**Content column.** Offset by `margin-left:var(--sidebar)`. Reading content is capped at 760px and centered; the hero and any wide visuals may use the 1000px `--wide` width.

**Scroll progress bar.** A 3px fixed bar at the very top (`#progress`) with an indigo→coral gradient; its width tracks `scrollY / scrollHeight`.

**Scrollspy.** On scroll, the chapter whose section top has passed a threshold gets `.active` (coral left-border + indigo text + wash background); likewise the current sub-section link. This is the reader's "you are here."

**Responsive.** At `max-width:1080px` the sidebar collapses to `--sidebar:0`, becomes a slide-in drawer toggled by the hamburger `#menuBtn` over a dimming `#scrim`. At `640px`, font sizes step down and control min-widths shrink. A `prefers-reduced-motion` query disables smooth scroll.

---

## 5. The component library

These classes are the vocabulary of the book. Use them consistently; do not invent new box styles per chapter.

### 5.1 The box taxonomy (the heart of the system)

Four semantic boxes, each color-coded with a labeled dot. The label is uppercase mono with a small filled circle (`::before`) in the box's accent color.

| Class | Color | Purpose | Label convention |
|---|---|---|---|
| `.box.def` | indigo wash | **Definition** — the precise statement of an idea | "Definition — <name>" |
| `.box.intuition` | teal wash | **Intuition** — analogy/picture; readable without the math | "Intuition — <hook>" |
| `.box.derivation` | white, thick black left-border | **Step-by-step derivation** | "Derivation — <what>" |
| `.box.note` | amber wash | **Forward reference** — "Where you'll meet this again" | "Where you'll meet this again" |

```html
<div class="box def">
  <div class="box-label">Definition — dot product</div>
  <p>…prose with $inline$ and $$display$$ math…</p>
</div>
```

**Derivations** use numbered steps:

```html
<div class="box derivation">
  <div class="box-label">Derivation — why arithmetic equals geometry</div>
  <div class="step"><div class="s-num">1</div><div class="s-body">
    <p>…</p>$$…$$
  </div></div>
  <div class="step"><div class="s-num">2</div><div class="s-body">…</div></div>
</div>
```

The `.s-num` is a small indigo circle; `.s-body` holds prose + equations. Aim for 2–4 steps, each a genuine logical move.

### 5.2 Defined terms

Inline, wrap the first occurrence of a key term in `<span class="term">…</span>` (semibold, indigo-deep). Use sparingly — only for terms the reader should latch onto.

### 5.3 The demo card

The interactive unit. See §8 for the JS; the HTML shell is:

```html
<div class="demo">
  <div class="demo-head">
    <div class="tag">Interactive</div>
    <h4>Human-readable demo title</h4>
  </div>
  <div class="demo-body">
    <canvas id="fooCanvas" width="640" height="360"></canvas>
    <div class="demo-controls">
      <div class="ctrl">
        <label>angle <span class="val" id="fooAngV">45°</span></label>
        <input type="range" id="fooAng" min="0" max="360" step="1" value="45">
      </div>
      <div class="ctrl" style="flex:0;justify-content:flex-end">
        <button class="btn" id="fooRun">▸ Run</button>
        <button class="btn ghost" id="fooReset">Reset</button>
      </div>
    </div>
    <div class="readout" id="fooReadout"></div>
  </div>
</div>
```

Control conventions: sliders live in `.ctrl` with a `<label>` whose right-aligned `.val` mirrors the live value **in coral**; primary action buttons are solid coral `.btn`; secondary actions are `.btn.ghost`; mutually-exclusive presets use a `.seg` segmented control whose active button carries `.on`. The `.readout` is a mono panel that narrates what's happening in plain language and updates every frame.

### 5.4 Other components

`.endcard` — a dark indigo "Where this goes next" card closing each layer. `table` — styled with indigo-deep headers and soft row dividers, for comparisons. `.chip` — small mono pill for tags. `.svg-fig` — a framed inline SVG for static diagrams (see §10). `footer.page` — closing line in the footer.

---

## 6. Pedagogical structure & writing voice

**Per-chapter rhythm.** Open with a `.chapter-head` (mono kicker "Chapter 0N" + serif `h2`) and a large `p.intro` that frames why the chapter matters. Then 2–4 sub-sections, each following the loop: **intuition → definition → derivation → demo → forward-link**. Not every sub-section needs all five, but the *intuition usually comes first* — set the picture before the formalism.

**Voice.** Warm, precise, second-person ("you"), confident but never breezy. Short declarative sentences for key claims. Analogies are concrete and reused across the book so they compound. The reference layers established these — reuse them when the concept recurs:

- vector = a recipe (parts east, parts north)
- dot product = an agreement meter
- matrix = a machine that moves space; its columns = where the basis lands
- derivative = a speedometer / a zoom lens (smooth = locally straight)
- chain rule = gear ratios multiplying
- gradient = the steepest-uphill arrow
- probability = area; conditioning = zooming in
- MLE = picking the least-surprising world
- overfitting = memorizing the answer key vs. understanding
- entropy = average surprise; information = resolution of uncertainty

**Forward links.** Every chapter ends key ideas with an amber `.note` titled "Where you'll meet this again," explicitly naming where the idea returns in later layers. This is what turns a set of chapters into a *curriculum*. (Layer 1 → Layer 2 → … → VLA.)

**Closing.** Each layer ends with a one-paragraph "the whole layer in a paragraph" intuition box, then a dark `.endcard` pointing to the next layer.

**Depth target.** College intro-course depth: full derivations, but every symbol introduced; no "it can be shown that." When a result needs heavier machinery than the layer assumes, state it, give the intuition, and cite where it's proven.

---

## 7. Math rendering — KaTeX (read this carefully)

Math is authored in LaTeX between `$…$` (inline) and `$$…$$` (display), rendered client-side by KaTeX auto-render.

### 7.1 Head tags — NO integrity attributes

```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/KaTeX/0.16.9/katex.min.css">
<script defer src="https://cdnjs.cloudflare.com/ajax/libs/KaTeX/0.16.9/katex.min.js"></script>
<script defer src="https://cdnjs.cloudflare.com/ajax/libs/KaTeX/0.16.9/contrib/auto-render.min.js"
  onload="window.__katexReady=true;if(window.__doRender)window.__doRender();"></script>
```

**Hard-won lesson #1:** never add `integrity`/`crossorigin` SRI hashes to these tags unless you have the exact correct hash. A wrong/fabricated hash silently blocks the script and *all* math fails. Omit them.

### 7.2 The render call — must retry, not fire once

**Hard-won lesson #2 (the bug that shipped):** the deferred KaTeX script may not have finished executing at `DOMContentLoaded`. If you call the renderer exactly once and `renderMathInElement` isn't defined yet, every formula silently fails to render while the rest of the page (canvas demos, etc.) works — which is a confusing failure because it looks topic-specific but is really a load race. **Fix: poll until KaTeX is available.**

```js
window.__doRender=function(el){
  if(!window.renderMathInElement) return;
  renderMathInElement(el||document.body,{
    delimiters:[
      {left:'$$',right:'$$',display:true},
      {left:'\\[',right:'\\]',display:true},
      {left:'$',right:'$',display:false},
      {left:'\\(',right:'\\)',display:false}
    ],
    throwOnError:false,
    macros:{"\\R":"\\mathbb{R}","\\E":"\\mathbb{E}"}
  });
};
// Render as soon as KaTeX is available; the CDN script may not be ready at
// DOMContentLoaded, so poll briefly instead of giving up after one try.
(function(){
  if(window.renderMathInElement){ window.__doRender(); return; }
  var tries=0, iv=setInterval(function(){
    if(window.renderMathInElement){ clearInterval(iv); window.__doRender(); }
    else if(++tries>200){ clearInterval(iv); } // ~10s ceiling
  },50);
})();
```

`throwOnError:false` means a single malformed expression renders red instead of blanking the page. Keep the macro set tiny (`\R`, `\E`) and consistent across layers.

**Authoring caveat:** avoid a bare `<` followed by a space or digit *inside inline math* in HTML prose (e.g. `$|\lambda|<1$`) — it can confuse the HTML parser. Write `\lt` / `\gt` instead (`$|\lambda|\lt 1$`). The validation battery (§11) greps for this.

---

## 8. The interactive demo engine

All demos share one infrastructure block defined once at the top of the `DOMContentLoaded` handler, then each demo is a self-contained IIFE.

### 8.1 Shared infrastructure

```js
var C = { ink:'#14162A', inkSoft:'#474B66', faint:'#878BA6',
          indigo:'#4338CA', indigoDeep:'#2A2090', coral:'#FF5C49',
          coralDeep:'#E8412E', teal:'#0E9C92', line:'#DBDDE8',
          wash:'#EEEEF8', amber:'#C98314' };
var REDUCE = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
var DISP='600 13px "Space Grotesk",sans-serif', MONO='12px "JetBrains Mono",monospace';

var redraws=[];
// DPR-aware sizing: makes canvases crisp on retina AND responsive on resize.
function fit(cv,aspect){
  var dpr=window.devicePixelRatio||1;
  cv.style.width='100%';
  var w=cv.clientWidth||cv.parentElement.clientWidth;
  var h=Math.round(w/aspect);
  cv.style.height=h+'px';
  cv.width=Math.round(w*dpr); cv.height=Math.round(h*dpr);
  var ctx=cv.getContext('2d'); ctx.setTransform(dpr,0,0,dpr,0,0);
  return {w:w,h:h,ctx:ctx};
}
function register(fn){ redraws.push(fn); fn(); }       // draw once now, redraw on resize
window.addEventListener('resize',function(){
  clearTimeout(window._rt);
  window._rt=setTimeout(function(){ redraws.forEach(function(f){f();}); },160);
});
function gauss(){ var u1=Math.random()||1e-9,u2=Math.random();
  return Math.sqrt(-2*Math.log(u1))*Math.cos(2*Math.PI*u2); }  // Box–Muller normal
```

### 8.2 The canonical demo pattern

Every demo is an IIFE guarded by an existence check so a missing canvas can never throw:

```js
(function(){
  var cv=document.getElementById('fooCanvas'); if(!cv) return;   // guard
  var ang=document.getElementById('fooAng'),
      angV=document.getElementById('fooAngV'),
      rd=document.getElementById('fooReadout');
  var W,H,ctx,S;
  function draw(){
    S=fit(cv,640/360); W=S.w; H=S.h; ctx=S.ctx; ctx.clearRect(0,0,W,H);
    angV.textContent = ang.value+'°';
    // …compute from control values, draw with ctx using C.* colors…
    rd.innerHTML = 'Plain-language narration with the live <b>'+result+'</b>.';
  }
  ang.addEventListener('input',draw);          // re-draw on every control change
  register(draw);                              // initial draw + resize hookup
})();
```

For **animated** demos, drive a `requestAnimationFrame` loop, and provide a static fallback when `REDUCE` is true (e.g. run the simulation forward a fixed number of steps once, then stop). The drifting vector-field hero is the reference example.

### 8.3 Demo design principles

- **One idea per demo.** The demo should make exactly one concept undeniable.
- **Always narrate.** The `.readout` must translate the current state into words and adapt its message to regimes ("Underfit", "Diverged!", "Nearly parallel — maximum agreement").
- **Reward poking.** Build in the instructive extremes: the learning-rate that diverges, the matrix that collapses space, the |x| kink with no derivative, the prior so rare a positive test still means ~9%. The "aha" lives at the edges.
- **Direct manipulation where possible.** Click-to-add-points, click-to-place-the-ball, drag-the-vector beat slider-only demos.
- **Coral = interactive, everywhere.** Controls, draggable handles, and the live `.val` numbers are coral; computed structure (curves, grids, fits) is indigo; secondary data/series are teal.

### 8.4 Demo archetype catalog (proven patterns to reuse)

1. **Vector / geometry manipulator** — rotate/scale an arrow; show projection shadow + a live meter. (dot product)
2. **Space transformer** — sliders for a matrix's entries; warp a grid + unit square; overlay eigen-directions; presets via `.seg`. (linear maps)
3. **Secant→tangent zoom** — a curve with an amber secant that melts into a coral tangent as `h→0`; function presets. (derivative)
4. **1-D descent** — click to place a ball on a multi-well curve; run gradient descent; expose divergence and local minima. (optimization)
5. **Population dot-grid** — N people as dots colored by class, filled by test result; live Bayesian posterior. (conditional probability)
6. **Sampling histogram** — repeatedly average draws from an ugly source; watch a Gaussian emerge; fit overlay. (CLT / sampling)
7. **Likelihood curve** — accumulate observations; show the normalized likelihood narrowing around the truth. (MLE)
8. **Click-to-fit regression** — add/remove data points; live closed-form line + residual segments. (least squares)
9. **Hand-fit-then-solve** — sliders to manually fit an S-curve + a "Fit by MLE" button that animates gradient descent. (logistic regression)
10. **Complexity dial** — a polynomial-degree slider showing under/overfit on train vs. held-out points, with a side U-shaped error chart. (generalization)
11. **Distribution shaper** — a slider morphing a distribution from uniform to peaked; live entropy meter + per-outcome surprise. (information theory)

Each layer should aim for **~8–12 demos plus one animated hero**, roughly one interactive per major concept.

---

## 9. The hero

A full-width `header.hero` (dark indigo, `--ink` background) with a `<canvas id="heroCanvas">` running a topic-flavored generative animation behind the title. Each layer gets a *different* animation so layers are visually distinguishable (Layer 1 = drifting vector field; Layer 2 = neural-net constellation). Pattern: size to the header, ~90 particles, trail-fade by painting a translucent background rect each frame, highlight a few particles in coral, and provide a `REDUCE`-motion static render. The `hero-inner` holds a mono eyebrow, a serif `h1` with an `<em>` accent, a `lead` paragraph, and a "Scroll to begin · N chapters" cue.

---

## 10. Static visual aids (SVG)

Not everything should be a canvas. For fixed conceptual diagrams (pipelines, anatomy, labeled structures), author inline `<svg class="svg-fig" viewBox="…">` using the same color tokens. Use SVG when the picture is **explanatory and static**; use canvas when it must **respond to input or animate**. Both are framed identically (`--surface` fill, `--line` border, radius 14, soft shadow) so they read as one family. Tables (styled per §5.4) are the third visual aid — ideal for side-by-side comparisons.

---

## 11. Build workflow & validation

**Construction.** Reuse the proven head+CSS from the previous layer verbatim (it's the design system), change only the `<title>`. Append all subsequent content with quoted heredocs (`cat >> file << 'EOF'`) so `$math$` and JS `${...}` template literals pass through literally without shell interpolation. Build in `/home/claude`, then copy the finished file to `/mnt/user-data/outputs/` and present it.

**Validation battery — run before every delivery.** This catches the failure modes that have actually shipped:

```bash
# 1. JS syntax: extract the inline script and node --check it
awk '/^<script>$/{f=1;next} /^<\/script>$/{f=0} f' file.html > /tmp/x.js
node --check /tmp/x.js

# 2. Tag balance (expect equal open/close for html, body, script, section)
for t in html body script section; do
  echo "$t: open=$(grep -o "<$t[ >]" file.html|wc -l) close=$(grep -c "</$t>" file.html)"; done

# 3. No fabricated SRI hashes (must be 0)
grep -c 'integrity=' file.html

# 4. Math delimiter parity ($ count must be even)
grep -o '\$' file.html | wc -l

# 5. No bare '<' + space/digit in PROSE (strip script first; must be 0)
sed '/^<script>$/,/^<\/script>$/d' file.html | grep -cE '<[[:space:]0-9]'

# 6. Every canvas/control id present exactly once
#    (loop your id list; each grep -c must equal 1)

# 7. Every getElementById / querySelector target exists in the DOM
```

A stronger optional check: stub a minimal DOM + canvas in Node, `eval` the script, fire `DOMContentLoaded`, and confirm every demo's initial `draw()` runs without throwing. This catches runtime errors that `node --check` cannot.

---

## 12. Reusable build prompt (fill in and paste)

> Build **Layer N — \<TITLE\>**, the next single-file interactive HTML textbook in the **Signal & Field** system, exactly matching the attached design spec and the existing layers.
>
> **Topic & scope:** \<one-paragraph description of what this layer teaches and where it sits in the curriculum\>.
>
> **Chapters (\<count\>):** \<list chapter titles and their 2–4 sub-sections each\>.
>
> **Requirements (all mandatory):**
> - Single self-contained HTML file; only KaTeX from CDN; opens by double-click.
> - Reuse the head + full CSS design system verbatim from the previous layer; change only the title and the brand "Layer N of 6."
> - College-intro depth: every key formula gets a **numbered step-by-step derivation**; introduce every symbol; no "it can be shown."
> - **Intuition-first**: each concept paired with a plain-language analogy/picture in a teal `.intuition` box; reuse the established analogies where concepts recur.
> - Box taxonomy used consistently: `.def` (indigo), `.intuition` (teal), `.derivation` (numbered steps), `.note` (amber "Where you'll meet this again" forward-links to later layers).
> - **\<8–12\> interactive canvas demos plus one animated hero** unique to this layer; coral reserved exclusively for interactive elements; every demo has a live narrating `.readout` and instructive extremes to discover. Prefer direct manipulation (drag/click) over sliders alone.
> - KaTeX set up with the **polling render** (no SRI hashes, retry until loaded) per the spec.
> - Fixed sidebar TOC with scrollspy + progress bar; responsive drawer on mobile; `prefers-reduced-motion` fallbacks for all animation.
> - End with a "whole layer in a paragraph" intuition box and a dark `.endcard` pointing to Layer N+1.
>
> **Before delivering**, run the full validation battery from the spec (JS syntax, tag balance, zero SRI, even `$` parity, no bare `<` in prose, all ids unique and resolvable) and fix anything it flags.

---

## 13. Quick checklist (tape to your monitor)

- [ ] Head + CSS reused verbatim; only title/brand changed
- [ ] Coral appears **only** on interactive things
- [ ] Serif body / sans headings / mono numerics — all three present
- [ ] Every key formula has a numbered derivation
- [ ] Every concept has an intuition box + a reused analogy
- [ ] Amber forward-links connect this layer to the next
- [ ] 8–12 demos + 1 unique animated hero, each with a narrating readout
- [ ] Instructive extremes built into demos (divergence, collapse, kinks…)
- [ ] KaTeX: no SRI hashes, polling render, `throwOnError:false`
- [ ] Reduced-motion fallback on every animation
- [ ] Validation battery passes; demos initialize without throwing
- [ ] File copied to outputs and presented
