# Contribution 1: Fix reflow of index on image load

**Contribution Number:** 1
**Student:** Nyan Lin Htet
**Issue:** [astral-sh/uv #6264 — Fix reflow of index on image load](https://github.com/astral-sh/uv/issues/6264)
**Status:** Phase IV — In Progress

---

## Why I Chose This Issue

On the uv documentation homepage (https://docs.astral.sh/uv/), the benchmark image has no reserved height, so when it finishes loading the surrounding content shifts down to make room for it. This causes a layout reflow (cumulative layout shift) where the page visibly "jumps" as the user is reading. It matters because layout shift is a jarring user-experience problem and a recognized Core Web Vitals metric (CLS) that hurts both readability and perceived quality of the docs. I chose it because it's a well-scoped, beginner-friendly frontend/docs fix where the root cause is clear (set a fixed height/aspect ratio for the image), and it lets me practice diagnosing layout shift and contributing to a real, widely used open-source project.

---

## Understanding the Issue

### Problem Description

The uv docs homepage embeds a benchmark bar-chart image without telling the browser how tall it will be. The browser therefore reserves **no** vertical space for the image during initial layout. When the image bytes finally arrive and the image is painted, the browser has to grow the image's box from 0px to its real height, which pushes every element below it (the caption, the "Highlights" heading, the feature list, etc.) downward. The result is a visible "jump" — a Cumulative Layout Shift (CLS).

### Expected Behavior

The page layout should be stable while the benchmark image loads. The space the image will occupy should be reserved up front (via a fixed height or an aspect ratio), so content below the image does not move when the image appears. CLS contribution from the image should be ~0.

### Current Behavior

While the benchmark image is still downloading, the "Highlights" section sits directly beneath the intro paragraph. The instant the image loads, the chart appears and shoves "Highlights" (and everything below it) downward by roughly the image's height — a visible reflow / layout shift.

### Affected Components

- **`docs/index.md`** — the homepage source. It embeds the benchmark chart twice (light + dark variants) as raw `<img>` tags with **no** `width`/`height`/`aspect-ratio`:
  ```html
  <p align="center">
    <img alt="Shows a bar chart with benchmark results." src="https://github.com/astral-sh/uv/assets/1309177/629e59c0-9c6e-4013-9ad4-adb2bcf5080d#only-light">
  </p>
  <p align="center">
    <img alt="Shows a bar chart with benchmark results." src="https://github.com/astral-sh/uv/assets/1309177/03aa9163-1c79-4a87-a31d-7a9311ed9310#only-dark">
  </p>
  ```
- **`docs/stylesheets/extra.css`** — the docs' custom stylesheet. It already contains theme-aware rules that target these exact images via `img[src$="#only-light"]` / `img[src$="#only-dark"]`, so it is the natural place to add a sizing rule if a CSS-based fix is preferred.
- Built/served with MkDocs Material (`uv run --only-group docs mkdocs serve -f mkdocs.yml`, per uv's `CONTRIBUTING.md`).

---

## Reproduction Process

### Environment Setup

The issue is a frontend/docs rendering problem, so reproducing it does not require building uv from Rust source. I confirmed the root cause by reading `docs/index.md` (the `<img>` tags carry no dimensions) and `docs/stylesheets/extra.css` (no height/aspect-ratio rule for them). I reproduced and **measured** the layout shift two ways: (1) directly against the live docs site in Chrome DevTools, and (2) with a small, deterministic Playwright harness that isolates the exact homepage markup and reports CLS via the W3C Layout Instability API (the same metric Chrome DevTools / Lighthouse use). The only setup needed for the harness is Node.js + a local Google Chrome.

### Steps to Reproduce

**Path A — observe it on the live docs (no build, ~1 minute):**

1. Open Google Chrome and navigate to https://docs.astral.sh/uv/.
2. Open DevTools (F12 / Cmd+Opt+I) → **Network** tab → set throttling to **"Slow 3G"** and tick **"Disable cache"**.
3. Hard-reload the page (Cmd+Shift+R / Ctrl+Shift+R) and watch the area beneath the large "uv" heading.
4. **Observed result:** while the benchmark chart is still downloading, "Highlights" sits right under the intro text; the moment the chart finishes loading it appears and pushes "Highlights" and everything below it downward — the page visibly jumps.
5. To quantify it: DevTools → **Performance** panel (or **Lighthouse**) → record a reload. The trace reports a non-zero **Cumulative Layout Shift** whose offending element is the benchmark `<img>`.

**Path B — deterministic, measured harness (on my fork branch):**

1. Check out my reproduction branch and enter the folder:
   ```bash
   git clone --branch repro/issue-6264-image-reflow https://github.com/alex-nyan/uv.git
   cd uv/repro-6264
   ```
2. Install Playwright (it drives your already-installed Google Chrome):
   ```bash
   npm init -y
   PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 npm i playwright
   ```
3. Run the measurement:
   ```bash
   node measure-cls.mjs
   ```
4. **Observed result** (written to `results.json` and printed): the buggy page (`homepage-repro.html`, dimensionless `<img>`) records a layout-shift event and **CLS 0.046** on desktop / **0.049** on mobile, with the image box growing from **0px → 221px** (desktop) on load. The fixed page (`homepage-fixed.html`, which reserves the aspect ratio) records **CLS 0** with **zero** shift events. Before/after screenshots are saved alongside.

   *(No Node? Just open `homepage-repro.html` in a browser with the Network tab throttled to "Slow 3G" and watch it jump; open `homepage-fixed.html` the same way and watch it stay still.)*

### Reproduction Evidence

- **Branch in my fork:** https://github.com/alex-nyan/uv/tree/repro/issue-6264-image-reflow
- **Reproduction folder:** https://github.com/alex-nyan/uv/tree/repro/issue-6264-image-reflow/repro-6264
- **Commit showing reproduction:** https://github.com/alex-nyan/uv/commit/020d2368f8602fee02bcbbb20cf73796e2d9bcb2
- **Screenshots (desktop, buggy page):**

  *Before the image loads — "Highlights" sits directly under the intro paragraph:*

  ![Before the benchmark image loads](assets/repro-6264/desktop-buggy-before.png)

  *After the image loads — the chart appears and shoves "Highlights" and the feature list downward:*

  ![After the benchmark image loads](assets/repro-6264/desktop-buggy-after.png)

- **Measured CLS (W3C Layout Instability API; Lighthouse rates CLS > 0.1 as "needs improvement"):**

  | Viewport | Page | Image height before → after | Shift events | **CLS** |
  | --- | --- | --- | --- | --- |
  | Desktop 1280×800 | buggy | 0px → 221px | 1 | **0.046** |
  | Desktop 1280×800 | fixed | 221px → 221px | 0 | **0** |
  | Mobile 390×844 | buggy | 0px → 107px | 1 | **0.049** |
  | Mobile 390×844 | fixed | 107px → 107px | 0 | **0** |

- **My findings:**
  - The root cause is confirmed: the benchmark `<img>` tags in `docs/index.md` have no `width`, `height`, or `aspect-ratio`, so the browser reserves zero space until the bytes arrive and then reflows.
  - The real benchmark image is itself an **SVG with intrinsic size `496 × 107`** and is shown at natural size (centered), so the real-world downward shift is **≈107px**. My harness used a 1000×300 stand-in scaled to the content width (hence the 221px desktop delta), but the mechanism and conclusion are identical — and the harness proves the fix drives CLS to exactly **0**.
  - Reserving the image's box up front (a fixed height / `aspect-ratio`) eliminates the shift entirely with no change to how the image looks once loaded.

---

## Solution Approach

### Analysis

The browser cannot lay out an image's box until it knows the image's dimensions. With no `width`/`height` attributes and no CSS `aspect-ratio`, an `<img>` has an intrinsic height of 0 until the resource loads; once it loads, the box snaps to the real size and everything after it in the flow moves — the definition of a layout shift. The fix is to give the browser the dimensions ahead of time so it can reserve the correct box during the very first layout pass.

### Proposed Solution

Reserve the benchmark image's space before it loads by declaring its intrinsic size. The maintainer's note on the issue ("We need to set a fixed height for the image") points the same direction. Concretely, set the image's aspect ratio / dimensions so the layout is stable, while keeping it responsive (no distortion on narrow screens).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The homepage reflows because the benchmark `<img>` reserves no vertical space until it loads. Goal: make the layout stable (image CLS ≈ 0) without changing the image's rendered appearance or its existing light/dark theming.

**Match:** uv's docs already special-case these images in `docs/stylesheets/extra.css` using `img[src$="#only-light"]` / `img[src$="#only-dark"]` selectors (for theme-aware show/hide). That is the established pattern for targeting exactly these images, and the natural hook for a sizing rule. The broader pattern is the standard web.dev/MDN guidance: always give media an intrinsic size (width/height attributes or `aspect-ratio`) to prevent CLS.

**Plan:**
1. Confirm the intrinsic dimensions of both benchmark images (light + dark). The light SVG is `496 × 107` (aspect ratio ≈ 4.64:1); verify the dark variant matches.
2. Reserve the box. Preferred approach — add intrinsic attributes to the two `<img>` tags in `docs/index.md`:
   ```html
   <img alt="Shows a bar chart with benchmark results." width="496" height="107" src="...#only-light">
   ```
   This lets the browser compute the aspect ratio and reserve space immediately. (Alternative, if a markup-free fix is preferred: add a scoped rule to `docs/stylesheets/extra.css`, e.g. `img[src$="#only-light"], img[src$="#only-dark"] { aspect-ratio: 496 / 107; height: auto; }`.)
3. Preserve responsiveness: ensure the image still scales down on narrow viewports without distortion (`max-width: 100%; height: auto;`), so the declared dimensions act as a reserved ratio rather than a hard size.
4. Run the docs formatter on touched files (`npx prettier --write docs/index.md` and/or the CSS), per uv's `CONTRIBUTING.md`.

**Implement:** ✅ Done (Phase III) — branched off `astral-sh/uv` `main` and added `width="496" height="107"` to both benchmark `<img>` tags in `docs/index.md`. No CSS change was needed (see Implementation Notes). Commit: [`f0d0b3c`](https://github.com/alex-nyan/uv/commit/f0d0b3c264d31615c47f9dfac3604aa7999d38fb).

**Review:** Self-review checklist — change is minimal and docs-only; respects uv's `CONTRIBUTING.md` (formatted with Prettier; no new features; a docs issue that's appropriate for community contribution per the `documentation` label); no change to the image's loaded appearance; light/dark theming still works.

**Evaluate:** ✅ Done — see Testing Strategy below. The patched markup measures **CLS 0** with zero layout-shift events at desktop and mobile, and the image stays proportional.

---

## Implementation Notes

### Phase III Progress

The fix is implemented and verified. Summary of the work:

- **Probed the live site to decide the minimal correct fix.** Using Playwright + Chrome against https://docs.astral.sh/uv/, I confirmed the live page has a real **CLS of ~0.023** whose dominant shift is sourced from the `<p>` / `<h2>` / `<ul>` elements directly below the benchmark image — i.e. the content being pushed down. I also confirmed the benchmark image is a **496×107** SVG with **no** `width`/`height` attributes.
- **Determined that an attribute-only fix is sufficient and safe.** I tested whether adding `width`/`height` attributes would distort the image on narrow screens (which happens if `height:auto` isn't applied). At a 360px viewport the image stayed perfectly proportional (ratio 4.636), proving MkDocs Material already applies `height: auto` to content images. So **no CSS change to `extra.css` was needed** — the smaller, more idiomatic change.
- **Applied the fix:** added `width="496" height="107"` to both the `#only-light` and `#only-dark` benchmark `<img>` tags in `docs/index.md`. The browser now reserves the image's box during first layout, so nothing below it moves when the image loads. Prettier reports the file unchanged (already conformant).
- **Verified** the patched markup drives CLS to 0 with no distortion (details below).

### Code Changes

- **Files modified:** `docs/index.md` (2 lines — added intrinsic `width`/`height` to the two benchmark `<img>` tags). No other files changed.
- **Active development branch:** [`fix/issue-6264-reserve-benchmark-image-height`](https://github.com/alex-nyan/uv/tree/fix/issue-6264-reserve-benchmark-image-height)
- **Fix commit:** [`f0d0b3c` — docs: reserve benchmark image height to prevent layout shift (#6264)](https://github.com/alex-nyan/uv/commit/f0d0b3c264d31615c47f9dfac3604aa7999d38fb)
- **The diff:**
  ```diff
  - <img alt="Shows a bar chart with benchmark results." src="...#only-light">
  + <img alt="Shows a bar chart with benchmark results." width="496" height="107" src="...#only-light">
  - <img alt="Shows a bar chart with benchmark results." src="...#only-dark">
  + <img alt="Shows a bar chart with benchmark results." width="496" height="107" src="...#only-dark">
  ```
- **Verification harness** (on the reproduction branch): [`repro-6264/verify-fix/`](https://github.com/alex-nyan/uv/tree/repro/issue-6264-image-reflow/repro-6264/verify-fix) — `verify-fix.mjs` (before/after CLS of the exact fix), `probe-live.mjs` (live-site CLS), `probe-fix.mjs` (distortion check).
- **Approach decision:** chose the HTML attribute fix over a CSS `aspect-ratio` rule because (a) it's what the issue asked for ("set a fixed height for the image"), (b) it's the standard web.dev/MDN recommendation, (c) it's a 2-line, single-file change, and (d) the live-site probe proved it stays responsive without any CSS.

---

## Testing Strategy

### Automated CLS measurement (performed)

Re-ran the measurement harness against the **exact** shipped markup (`width`/`height` attributes) using the real 496×107 dimensions and Material's content-image CSS (`max-width:100%; height:auto`), via the W3C Layout Instability API:

| Viewport | Markup | Layout-shift events | **CLS** | Image ratio (4.636 = correct) |
| --- | --- | --- | --- | --- |
| Desktop 1280×800 | current (bug) | 1 | 0.0104 | 4.636 |
| Desktop 1280×800 | **fixed** | **0** | **0** | 4.636 |
| Mobile 390×844 | current (bug) | 1 | 0.0187 | 4.636 |
| Mobile 390×844 | **fixed** | **0** | **0** | 4.636 |

**Result: PASS** — the fix eliminates the layout shift entirely (CLS → 0, 0 shift events) and the image stays proportional at every viewport. (Harness verdict captured in [`verify-fix/results.json`](https://github.com/alex-nyan/uv/tree/repro/issue-6264-image-reflow/repro-6264/verify-fix).)

### Live-site baseline (performed)

Measured the unpatched live docs at https://docs.astral.sh/uv/ with network throttled (~Slow 3G, cache disabled): **CLS ≈ 0.023**, with the offending shift attributed to the elements below the benchmark image — confirming the bug is real in production, not just in the isolated harness.

### Manual / visual checks (recommended before/after the PR is reviewed)

- `uv run --only-group docs mkdocs serve -f mkdocs.yml` and load the homepage with DevTools → Network throttled to "Slow 3G" + "Disable cache"; confirm the page no longer jumps when the benchmark loads.
- Confirm light and dark themes both still show the correct benchmark image (the `#only-light` / `#only-dark` switching is untouched).
- Spot-check the image isn't distorted on a narrow (mobile) viewport.

---

## Pull Request

**Status:** Phase IV — in progress. The fix branch is final and verified; the PR to `astral-sh/uv` is being opened, after which I'll iterate on any maintainer feedback.

- **Source branch:** [`alex-nyan/uv:fix/issue-6264-reserve-benchmark-image-height`](https://github.com/alex-nyan/uv/tree/fix/issue-6264-reserve-benchmark-image-height)
- **Open-PR link (compare view):** https://github.com/astral-sh/uv/compare/main...alex-nyan:uv:fix/issue-6264-reserve-benchmark-image-height
- **PR Link:** _(added once opened)_

**What the PR does (my explanation):** the benchmark chart on the docs homepage was embedded with no intrinsic dimensions, so the browser reserved no space for it and the content below jumped down when it loaded (a Cumulative Layout Shift). The PR adds `width="496" height="107"` to both the light and dark benchmark `<img>` tags in `docs/index.md` so the browser reserves the image's box during the first layout pass. MkDocs Material already applies `height: auto` to content images, so the image stays responsive with no CSS change. I verified the element's CLS drops to 0 with no distortion.

**AI Policy compliance:** per uv's `CONTRIBUTING.md` and Astral's [AI Policy](https://github.com/astral-sh/.github/blob/main/AI_POLICY.md), the PR description is written in my own words, any AI assistance is disclosed, and I can explain the change and respond to maintainers myself.

---

## Resources Used

- Issue: [astral-sh/uv #6264](https://github.com/astral-sh/uv/issues/6264)
- uv homepage source: [`docs/index.md`](https://github.com/astral-sh/uv/blob/main/docs/index.md) and [`docs/stylesheets/extra.css`](https://github.com/astral-sh/uv/blob/main/docs/stylesheets/extra.css)
- uv [`CONTRIBUTING.md`](https://github.com/astral-sh/uv/blob/main/CONTRIBUTING.md) (docs preview + formatting)
- [web.dev — Cumulative Layout Shift (CLS)](https://web.dev/articles/cls) and [Optimize CLS](https://web.dev/articles/optimize-cls) (set dimensions on images/media)
- [MDN — Layout Instability API](https://developer.mozilla.org/en-US/docs/Web/API/Layout_Instability_API)
