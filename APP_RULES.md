# Widget Build Rules (GitHub prototype → Quickbase production)

**Audience:** anyone (human or AI assistant) creating or changing an app in this repo.
**Goal:** every app that works on GitHub Pages must also work, unchanged, as a Quickbase **Code Page**.
If a rule here conflicts with a request, follow the rule and flag the conflict. Don't quietly break it.

---

## 0. Why these rules exist

| | GitHub Pages (prototype) | Quickbase Code Page (production) |
|---|---|---|
| URL | `https://lyondsgn.github.io/Test-apps/<folder>/<file>.html` | `https://<realm>.quickbase.com/db/<appDbid>?a=dbpage&pagename=<file>.html` |
| Folders | Yes | **No.** Pages live in one flat list, addressed by name |
| Other files (css/js/img/json) | Served next to the page | **Not served.** Each code page is a single text blob |
| Origin / storage | `lyondsgn.github.io`, shared by all your Pages sites | `<realm>.quickbase.com`, **shared by every app and page in the realm** |
| Who can open it | Anyone with the URL (repo is public) | Only signed-in Quickbase users with access to the app |
| Saving data | Nowhere (download to device only) | Quickbase tables via the REST API |
| Secrets | **Everything is public** | Still visible to any user who can open the page |

Anything that relies on the left column only (relative files, folders, a backend, a stored token) breaks on the right.

---

## 1. File and structure rules (MUST)

1. **One app = one self-contained `.html` file.** All CSS in `<style>`, all JS in `<script>`. No `<link>`/`<script src>` to files in this repo.
2. **No build step.** No npm, bundlers, TypeScript, JSX, or frameworks that need compiling. Plain HTML/CSS/JS (ES2020) only.
3. **File names:** lowercase kebab-case, `.html`, **unique across the whole repo** (Quickbase has no folders, so `a/report.html` and `b/report.html` would collide). Example: `site-photo-log.html`.
4. Folders are allowed in GitHub for organization (`DrumPhysics/`), but **nothing may depend on the folder path.**
5. **Images/icons:** inline SVG or `data:` URIs only. Keep the page under ~1 MB.
6. **No `fetch()` of local files** (`./data.json`, etc.). Put reference data in a `const` inside the page.
7. **External libraries:** avoid them. If one is truly needed, it must be:
   - loaded from `https://cdnjs.cloudflare.com` with an exact pinned version (e.g. `.../jspdf/2.5.1/jspdf.umd.min.js`),
   - listed in the app header comment (rule 12),
   - optional where possible (the app should still load if the CDN is blocked).
   Confirm with the Quickbase admin that the realm allows external scripts before relying on one in prod.
   Google Fonts stylesheets are OK as long as a system-font fallback is in the CSS stack.
8. **No** service workers, web app manifests, cookies, `iframe`s of other sites, or `window.open` popups as a required step.
9. **Every page** includes, at the top of `<head>`:
   ```html
   <meta name="robots" content="noindex, nofollow, noarchive, nosnippet, noimageindex">
   <meta name="referrer" content="no-referrer">
   ```

## 2. Navigation (MUST)

10. **Never hardcode a link to another app.** Use the `appUrl()` helper (copy from `index.html`) and set `HOME` to the path from your page up to the repo root (`''` at the root, `'../'` one folder down). Pass repo-relative paths (`appUrl('index.html')`). It produces:
    - GitHub: the relative path, e.g. `DrumPhysics/mixer-drum-torque-calculator.html`
    - Quickbase: `?a=dbpage&pagename=mixer-drum-torque-calculator.html`
11. **Register every new app** in the `APPS` array in `index.html` (`id`, `name`, `file`, `desc`, `status`, `features`).

## 3. App header comment (MUST)

12. The first thing inside `<body>` (or right after `<head>`) is a comment block:
    ```html
    <!--
      app-id:    site-photo-log            (kebab-case, matches file name)
      version:   0.1.0
      owner:     <name or team>
      purpose:   One sentence.
      uses:      camera, pdf, csv          (any of: camera, pdf, csv, qb-read, qb-write, storage)
      externals: none                      (or pinned CDN URLs)
      qb-tables: none                      (or table name + field ids it reads/writes in prod)
    -->
    ```

## 4. Storage and data (MUST)

13. **`localStorage` is for conveniences only** (last inputs, a remembered tab, a draft). Never the record of anything.
    - Keys **must** be prefixed `qbw.<app-id>.` (e.g. `qbw.mixer-drum-torque.inputs.v3`). The Quickbase origin is shared with every other page in the realm.
    - Wrap every read/write in `try/catch`; the app must work when storage is empty or blocked.
14. **All saving goes through one adapter** in the page, so the prototype and prod differ in exactly one place:
    ```js
    // Returns a Promise. In the prototype it downloads; on Quickbase it can also write a record.
    async function saveReport({ blob, fileName, fields }) { ... }
    ```
    App code calls `saveReport()` and nothing else. No scattered `fetch` calls to Quickbase.
15. **Environment check** (one line, used by the adapter and `appUrl()`):
    ```js
    const ON_QB = /\.quickbase\.com$/i.test(location.hostname);
    ```
16. **Quickbase writes** (prod only) use the REST API with a **temporary token from the user's own session**. Never a stored token:
    ```js
    // Sketch: verify field ids and the table dbid with the QB admin before shipping.
    async function qbToken(dbid) {
      const r = await fetch(`https://api.quickbase.com/v1/auth/temporary/${dbid}`, {
        credentials: 'include',
        headers: { 'QB-Realm-Hostname': location.hostname }
      });
      if (!r.ok) throw new Error('QB auth ' + r.status);
      return (await r.json()).temporaryAuthorization;
    }
    // Then: POST https://api.quickbase.com/v1/records
    //   headers: { 'QB-Realm-Hostname': location.hostname, 'Authorization': 'QB-TEMP-TOKEN ' + token }
    //   body: { to: tableDbid, data: [{ 6: {value: 'text'}, 7: {value: {fileName, data: base64}} }] }
    ```
    In the GitHub prototype, `ON_QB` is false and this code path never runs. It must fail gracefully (fall back to download) if it does.

## 5. Security and privacy (MUST, because the repo is public)

17. **No secrets, ever:** no Quickbase user tokens, app tokens, API keys, passwords, realm-specific dbids for sensitive apps, or internal URLs in the code or commit history.
18. **No real data** in the repo: no customer names, employee names, job/site addresses, real photos, or pricing. Use obviously fake sample data.
19. A client-side password prompt or link key is **not** security. Don't add one and don't describe it as protection.
20. Photos can carry GPS in EXIF. When photos are re-encoded through a canvas (rule 22), the metadata is stripped. Keep it that way unless location is an explicit requirement.

## 6. Feature recipes (use these patterns)

### Camera / site photos
21. **Primary:** a file input. It works on iOS/Android, on both hosts, and inside Quickbase without extra permissions:
    ```html
    <input type="file" accept="image/*" capture="environment" multiple>
    ```
    `getUserMedia` (live viewfinder) is an optional enhancement only, always with the file-input fallback.
22. **Always downscale before saving:** draw to a `<canvas>`, longest edge ≤ 1600 px, `canvas.toBlob(cb, 'image/jpeg', 0.8)`. Phone originals are 3–8 MB each.
23. Keep photos in memory (array of Blobs + object URLs) until saved; revoke object URLs when removed.

### CSV reports
24. Build CSV in JS: quote every field, double inner quotes, `\r\n` line endings, and prefix `﻿` (BOM) so Excel reads UTF-8 correctly. Neutralise formula injection: prefix cells that start with `= + - @` with `'`.

### PDF reports
25. **Default:** a print stylesheet plus `window.print()` (user chooses "Save as PDF"). Zero dependencies, works everywhere.
    ```css
    @media print { .no-print{display:none!important} body{background:#fff;color:#000} @page{margin:12mm} }
    ```
26. **When a real file is required** (attach to a QB record, auto-named): jsPDF from cdnjs, pinned version, per rule 7.

### Downloads
27. One shared helper:
    ```js
    function download(blob, fileName) {
      const a = document.createElement('a');
      a.href = URL.createObjectURL(blob); a.download = fileName;
      document.body.appendChild(a); a.click(); a.remove();
      setTimeout(() => URL.revokeObjectURL(a.href), 1000);
    }
    ```
    File names: `<app-id>_<YYYY-MM-DD>_<short-label>.<ext>`, no spaces.

## 7. Desktop + mobile (MUST)

Every app is used on **both** an office desktop and a phone on site. Neither is an afterthought.

28. **Widths to support:** 360 px (small phone) up to 1440 px+ (desktop). No horizontal page scroll at any width. Test at **360, 390, 768, 1280**, and in phone landscape.
29. **One responsive layout, not two apps.** Use CSS grid/flex with `minmax(0,1fr)` and one main breakpoint around **860 px**: side-by-side above it, single column below. Cap content width on desktop (`max-width` ~1180 px) so lines don't stretch.
30. **Keep results in view on phones.** If inputs and results sit in separate areas, the phone layout must show the key results while editing, using a sticky summary bar (see the calculator's `.mbar`) or by placing results directly beside each input.
31. **Touch rules** (apply under `@media (pointer:coarse)`):
    - tap targets ≥ 44 × 44 px (inputs, buttons, checkboxes, `<summary>`), with ≥ 8 px between them;
    - **inputs, selects and textareas at `font-size:16px` or larger**, or iOS Safari zooms the page on focus;
    - numeric fields use `type="number" inputmode="decimal"` so the number pad opens.
32. **Nothing hover-only.** Anything shown on hover (tooltips, chart readouts) must also work by tap: handle `pointerdown`/`click`, not just `pointermove`/`mouseover`. Wording follows the device: "Hover" on desktop, "Tap" on touch (`@media (hover:none)`).
33. **Charts and SVGs:** size from the container (`ResizeObserver`), `width:100%; height:auto`, with fewer ticks and labels when narrow. Use `touch-action:pan-y` on interactive charts so the page still scrolls. Floating readouts are clamped inside their box.
34. **Tables:** wrap in `overflow-x:auto` *and* hide non-essential columns under the breakpoint. Better still, for short tables, switch to stacked label/value rows on phones.
35. **Safe areas:** `viewport-fit=cover` plus `env(safe-area-inset-*)` padding so nothing sits under the notch or home bar; sticky bars offset by `env(safe-area-inset-top)`.
36. **Field conditions:** strong contrast (readable in sunlight), large primary numbers, no tiny controls (usable with gloves), no essential info in color alone.
37. **Desktop niceties still count:** keyboard focus rings (`:focus-visible`), Enter/Tab flow through inputs, readable at 100% zoom without a giant phone layout stretched across the screen.

## 8. Other UI rules (SHOULD)

38. Colors as CSS custom properties on `:root`, with a dark-mode override under `@media (prefers-color-scheme: dark)`. Give `body` an explicit background.
39. Include a "← All tools" link built with `appUrl('index.html')`.
40. Show which environment you're on (small "Prototype" badge when `!ON_QB`) so nobody mistakes the prototype for the system of record.

## 9. Definition of done (checklist before merging)

- [ ] Single `.html`, kebab-case unique name, header comment filled in
- [ ] `noindex` + `no-referrer` meta present
- [ ] Opens correctly by **double-clicking the file locally** (`file://`). If it works there, it has no hidden file dependencies
- [ ] Opens from the GitHub Pages URL **on a phone**; camera/download features tested on iOS Safari and Android Chrome
- [ ] Desktop + mobile (§7): no sideways scroll at 360 px; key results visible while editing on a phone; tapping an input doesn't zoom; every hover readout also works by tap; looks right at 1280 px
- [ ] No console errors; works with `localStorage` cleared
- [ ] All saves go through `saveReport()`; no tokens or real data in the diff
- [ ] Registered in `index.html` `APPS`

## 10. Moving an app to Quickbase (for later)

1. Quickbase app → **Settings → Pages → New Page → Code page**.
2. Page name = the exact file name (e.g. `mixer-drum-torque-calculator.html`). Paste the whole file. Save.
3. Do the same for `index.html` so the launcher's `appUrl()` links resolve.
4. Open `https://<realm>.quickbase.com/db/<appDbid>?a=dbpage&pagename=index.html` and run the checklist again.
5. Wire the `ON_QB` branch of `saveReport()` to the real table/field ids.
