# MARKAZ — smoothness pass (2026-08-22)

Everything stays a single `index.html`. No build step, no framework, same Vercel deploy.

---

## Why not Next.js

Next.js changes how HTML gets *delivered*. It does not change how the browser *paints*.
The jank on this site is a paint and compositing problem: ~70 live `backdrop-filter`
surfaces, two scroll handlers forcing synchronous layout on every tick, and 780 KB of
logo PNGs. Port it to Next.js untouched and it janks identically, except now there's a
build step in the way. The five fixes below are what actually moves the frame rate.

---

## What changed

### 1. Assets — 781 KB → 155 KB (−80%)

| file | before | after |
|---|---|---|
| `markaz-logo.png` | 508 KB (640×737) | `markaz-logo.webp` 36 KB (420×484) |
| `ouj-logo.png` | 273 KB (900×683) | `ouj-logo.webp` 39 KB (440×334) |
| favicon | 508 KB PNG | `favicon-180.png` 11 KB |

Optimised PNGs ship alongside as `onerror` fallbacks. The logos render at 40–60 px —
they were being sent at 10× the pixels they need.

**Also delete from the repo:** `markaz-logo-hires.png` (2.1 MB) and `ouj-logo-hires.png`
(2.4 MB). Nothing references them. They only bloat clones and deploys.

### 2. Backdrop blur clamped — the main scroll-jank source

Every `backdrop-filter:blur(Npx)` is now `blur(min(Npx, var(--bmax)))`:

```css
:root{ --bmax:14px }
@media (max-width:900px){ :root{ --bmax:6px } }
```

The compositor re-samples a backdrop blur on **every frame the backdrop moves** — i.e.
every frame of a scroll — and the cost scales with the radius. Blurs ran up to 38 px
across ~70 surfaces. Capping at 14 px roughly halves the per-frame work; on a white
background you cannot see the difference. Phones drop to 6 px, which is where mid-range
Android actually recovers its frame budget.

`saturate()` was also stripped from `backdrop-filter` declarations — it was a second
full filter pass on top of every blur.

One knob controls all of it. Raise `--bmax` if you want more frost, lower it for more speed.

### 3. Two scroll handlers were forcing layout every tick

The header progress bar and the menu-button ring both read
`document.documentElement.scrollHeight` inside an unthrottled `scroll` listener. That
forces the engine to run layout synchronously before it can answer — on every scroll
event, so several times per frame.

Both now read a cached value (`SCROLLMAX`, refreshed on resize/load/ResizeObserver) and
write inside `requestAnimationFrame`.

### 4. Parallax was thrashing reads against writes

`upd()` interleaved `getBoundingClientRect()` (read) with `style.setProperty()` (write)
in the same loop, invalidating layout before each next read. Reads are batched first now,
writes second: one layout per frame instead of N.

### 5. Off-screen sections stop painting

```css
body > section{ content-visibility:auto; contain-intrinsic-size:auto 1100px }
```

The browser skips layout and paint for sections outside the viewport. On a page this long
it is the single largest win. The hero, header, footer, nav and modals are excluded.

> **Kill switch:** if an anchor link ever lands short, delete block **7b** in the
> `[PERF]` CSS section and nothing else.

### 6. Lenis — the actual "glides down the page" feel

```html
<script src="https://unpkg.com/lenis@1.3.26/dist/lenis.min.js" defer></script>
```

3 KB. Replaces the browser's step-wise wheel scrolling with one interpolated rAF loop.
Runs on native scroll, so `position:sticky`, `scrollY`, IntersectionObserver, the
scrollspy and the anchor nav all keep working untouched.

Tuning is in the boot script at the bottom of the file:

```js
lerp: 0.085          // 0.05 = floatier, 0.12 = tighter. This is the feel knob.
anchors: {offset:-84} // clears the fixed header on #hash jumps
syncTouch: false      // phones keep native momentum — genuinely smoother than emulating it
```

It stops itself for Bootstrap modals and the mobile nav drawer (otherwise the page
scrolls behind them), and it disables entirely under `prefers-reduced-motion` or if the
CDN is blocked.

### 7. Render-blocking head trimmed

- Google Fonts: 8 families / ~20 weight files → 13. Loaded non-blocking (`media=print`
  + `onload`), with a `<noscript>` fallback.
- Bootstrap Icons (100 KB+ webfont) → non-blocking, same pattern.
- `markaz-logo.webp` preloaded with `fetchpriority="high"` for the preloader.

---

## Order matters

Lenis was added **last**, and that is deliberate. Momentum scrolling puts the page at a
new sub-pixel offset every single frame, which forces every blurred surface to
re-composite every frame. Adding Lenis *before* fixing the blur and the layout thrash
would have made the site feel measurably worse.

---

## Still on the table

- **Unsplash placeholders** — 14 remote images. Real academy photos, exported to WebP
  at ~900 px, would cut another ~1 MB and remove a third-party round trip.
- **Bootstrap CSS** is ~230 KB for maybe 15% usage. Not worth hand-pruning unless you
  want to add a build step later.
- **Self-host the fonts** if Karachi mobile latency to `fonts.gstatic.com` looks bad in
  the field — worth checking on a real 4G connection before bothering.
