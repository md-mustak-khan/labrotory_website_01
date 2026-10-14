# Website Audit & Recommendations — md-mustak-khan.github.io

**Prepared:** 21 August 2026
**Scope:** Full static-site audit — all 10 HTML pages, both CSS files, `script.js`, images, and repo hygiene.
**Site:** Academic research portfolio for Md. Mustak Khan (University of Chittagong / BioPC).

---

## How to read this document

Findings are grouped by **priority**, highest impact first. Each item shows a severity tag, the affected file(s) with line numbers, the problem, and a concrete fix.

| Tag | Meaning |
|-----|---------|
| 🔴 **P0** | Broken / incorrect — fix first |
| 🟠 **P1** | Performance — big user-visible & SEO impact |
| 🟡 **P2** | SEO / discoverability |
| 🔵 **P3** | Accessibility (WCAG) |
| ⚪ **P4** | Code quality / maintainability |
| 🟣 **P5** | Repo hygiene / cleanup |

### Scorecard

| Area | Grade | One-line verdict |
|------|:-----:|------------------|
| Visual design & content | **A−** | Rich, polished, professional; strong academic content |
| Functionality | **C** | Contact form is effectively broken; one broken anchor |
| Performance | **D** | ~91 MB repo, ~85 MB images, full-size photos used as thumbnails |
| SEO | **B−** | Good meta/OG, but zero canonical tags, no JSON-LD, no sitemap |
| Accessibility | **C+** | Good alt text & skip link, but keyboard access & focus are weak |
| Code maintainability | **C−** | 10.5 k-line CSS pile, 505 `!important`, heavy inline styles |

---

## 🔴 P0 — Broken things (fix first)

### 1. Contact form is misleading / effectively non-functional
**File:** `contact.html:147`, `contact.html:262`
The form posts to a real Formspree endpoint (`action="https://formspree.io/f/mzzdwywn"`), **but**:
- The JS submit handler (`contact.html:262`) **never calls `e.preventDefault()`** and uses no `fetch()`. The browser does a full-page POST and navigates away to Formspree's page — so the "Sending…" spinner, `form.reset()`, and the success toast ("Message sent! I'll reply within 1-3 days.") are a **simulated** success the user never actually sees.
- The handler sets `btn.disabled = true` synchronously during submit, which in some browsers **cancels the submission entirely** — meaning the message may never send.
- `novalidate` (`contact.html:147`) disables native validation and there is **no JS validation**, so the `required` attributes on Name/Email/Subject/Message are **not enforced**.
- **Dead code:** the script targets `.field select` (`contact.html:253`, `269`) but the form has no `<select>`.

**Fix (recommended — AJAX submit, stays on page):**
```javascript
form.addEventListener('submit', async function (e) {
    e.preventDefault();
    if (!form.checkValidity()) { form.reportValidity(); return; }   // enforce required
    btn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> Sending...';
    btn.disabled = true;
    try {
        const res = await fetch(form.action, {
            method: 'POST',
            body: new FormData(form),
            headers: { 'Accept': 'application/json' }
        });
        if (res.ok) { form.reset(); toast.classList.add('show'); setTimeout(() => toast.classList.remove('show'), 4500); }
        else { alert('Something went wrong. Please email me directly.'); }
    } catch { alert('Network error. Please email me directly.'); }
    btn.innerHTML = '<span class="btxt">Send Message</span><i class="fas fa-paper-plane bico"></i>';
    btn.disabled = false;
});
```
Also **remove `novalidate`** (or keep it and rely on `checkValidity()` above) and delete the dead `.field select` code.

### 2. `protocols.html` is missing its `</html>` closing tag
**File:** `protocols.html` (ends at `</body>` on line 1308, no `</html>`)
Every other page closes correctly. Add `</html>` on the final line.

### 3. Broken in-page anchor
**File:** `research.html:108` → `href="about.html#academic"`
There is **no `id="academic"`** anywhere in `about.html`, so the link lands at the top of the page. It also uses `target="_blank"` to open an *internal* page in a new tab (unusual UX).
**Fix:** add `id="academic"` to the relevant section in `about.html`, or change the link target. Drop `target="_blank"` for internal links.

---

## 🟠 P1 — Performance (the ~85 MB image problem)

The repository is **~91 MB, of which images are ~85 MB.** This is the single biggest issue for real visitors (especially on mobile in Bangladesh where bandwidth matters) and for Google's Core Web Vitals.

### 4. Gallery uses full-resolution originals as thumbnails
**File:** `gallery.html` — 21 grid images, ~55 MB total on one page.
Worst offenders (full-size, no resized thumbnail):
- `C elegans under Fluorescent Microscope.jpg` **5.2 MB** — used twice (`gallery.html:77` + `:231`)
- `Lab Event 1.jpg` **4.8 MB** (`:297`), `Lab 3.jpg` **4.5 MB** (`:191`), `Lab visit.jpg` 3.8 MB (`:217`), `Poster Pre.jpg` 3.7 MB (`:257`), `Lab Experiment.JPG` 3.4 MB (`:165`)
- `TG-100 _Day5.gif` **4.1 MB animated GIF** autoplaying in-grid (`:244`)

**Fix:**
- Generate resized thumbnails (~400–600 px wide, WebP) for the grid; load the full image only in the lightbox.
- Use `srcset`/`sizes` so browsers pick the right size.
- Convert the autoplaying GIF to an MP4/WebM `<video>` (often 10–20× smaller).
- Target: get this page from ~55 MB to **under ~3 MB**.

### 5. Multi-MB images above the fold on `research.html`
**File:** `research.html`
- `images/gallery/TG-100 _Day5.gif` **4.1 MB** in the flagship section (`:115`)
- `images/gallery/Thesis Presentation.jpg` **2.5 MB** (`:124`)
- `images/gallery/C elegans...jpg` **5.2 MB**, displayed at only 350×400 (`:168`, inline size `:170`)

Compress/resize these; a 350×400 display box should not ship a 5 MB source.

### 6. Favicon is a 1.46 MB PNG — downloaded on every page load
**File:** every page (`logo_main.png`, e.g. `index.html:25-26`)
`logo_main.png` is **1.43–1.46 MB** and is used as the favicon site-wide (and as the 40 px nav logo, and as several `og:image`s).
**Fix:** export a proper small favicon (`favicon.ico` + 32×32/48×48 PNG, a few KB). Use a separate optimized logo (~20–40 KB) for the navbar. Create a dedicated 1200×630 `og-image.png` for social sharing.

### 7. Other large *used* images to compress
- `images/Bioinformatics_Research.png` **2.2 MB** (course banner, `teaching.html:148`)
- `images/In silico Innovation...png` **1.5 MB**
Re-export as WebP/optimized PNG (typically 70–90 % smaller).

### 8. No `width`/`height` on any `<img>` → layout shift (CLS)
Confirmed across all content pages and the gallery. Add explicit `width`/`height` (or CSS `aspect-ratio`) to every `<img>` so the browser reserves space before the image loads.

### 9. Hero / above-the-fold images use `loading="lazy"` (anti-pattern)
**Files:** `about.html:82-83` (hero avatar), `gallery.html:77/86/95` (featured images)
Lazy-loading the largest visible image delays the Largest Contentful Paint. **Remove `loading="lazy"` on above-the-fold images**; keep it for below-the-fold ones.

---

## 🟡 P2 — SEO / discoverability

### 10. Zero canonical tags site-wide
**All 10 pages.** No `<link rel="canonical">` anywhere (confirmed). Add one per page, e.g. on `about.html`:
```html
<link rel="canonical" href="https://md-mustak-khan.github.io/about.html">
```

### 11. No structured data (JSON-LD)
An academic site is the ideal candidate. Add:
- `Person` schema on `about.html` / `index.html` (name, affiliation, `sameAs` links to ORCID/Scholar/LinkedIn).
- `ScholarlyArticle` schema on `publications.html` for the Wiley paper (DOI, authors, journal) — materially helps scholarly indexing.

### 12. Open Graph gaps & inconsistencies
- **`research.html` and `contact.html` have no Open Graph tags at all** — add the standard block.
- **`resources.html:16`** — `og:image` is a **relative** URL (`logo_main.png`); OG requires absolute `https://…`, so the preview image won't resolve for scrapers.
- **`gallery.html:8`** — `og:title` ("Research Gallery | …") doesn't match `<title>` ("Gallery | …").
- No `twitter:card` and no `og:site_name` on any page — add for richer link previews.

### 13. No `sitemap.xml`, `robots.txt`, or `404.html`
None exist (confirmed). Add:
- `sitemap.xml` listing all 10 pages (helps indexing).
- `robots.txt` pointing to the sitemap.
- A branded `404.html` (GitHub Pages serves it automatically).

### 14. Minor meta inconsistencies
- `keywords`/`author` meta exist only on `resources.html` (`:8-9`) — either add site-wide or drop (keywords are ignored by Google anyway).
- The "Active March 2026" / stat lines on `resources.html:124` are becoming stale — consider auto-dating or softening.

---

## 🔵 P3 — Accessibility (WCAG)

### 15. Mobile hamburger is not a real button
**All pages** (e.g. `index.html:75-79`) — `<div class="nav-toggle">` with empty `<span>` bars: no `role`, no `aria-label`, not keyboard-focusable. Screen-reader and keyboard users cannot open the mobile menu.
**Fix:** make it `<button class="nav-toggle" aria-label="Open menu" aria-expanded="false">…</button>` and toggle `aria-expanded` in JS.

### 16. Gallery thumbnails are not keyboard-accessible
**File:** `gallery.html:76/85/94` and JS-attached `.g-item` handlers.
Thumbnails are `<div onclick=...>` with no `role="button"`, `tabindex`, or key handler — the lightbox **cannot be opened without a mouse/touch.**
**Fix:** use `<button>` wrappers (or add `role="button"` + `tabindex="0"` + Enter/Space handlers).

### 17. `prefers-reduced-motion` is completely absent
The site has heavy animation (scroll-driven timeline fills, glowing orbs, custom cursor lerp, typing effect, many transitions) but **zero** `@media (prefers-reduced-motion: reduce)` rules. This is a real barrier for users with vestibular disorders.
**Fix:** add a global reduce-motion block that disables non-essential animation/transition.

### 18. Focus visibility is near-unusable for keyboard users
Only 3 `:focus-visible` rules exist (`styles-extended.css:1721/1747/1753`), all on `.interactive-card`. Nav links, buttons, and general links have no custom focus style; the one ring color is a hardcoded faint navy (`rgba(15,42,74,0.12)`), **not theme-aware**, effectively invisible on the dark background. Combined with `cursor:none` (below), keyboard navigation is very hard to follow.
**Fix:** add a visible, theme-aware `:focus-visible` outline for all interactive elements.

### 19. `cursor: none` with a JS-only replacement
**File:** `styles.css:56` sets `body { cursor: none; }`; the replacement dot/ring are generated by `script.js` only on `(pointer: fine)`. **If JS fails or is blocked, desktop users get no visible cursor at all.** Touch is handled (`styles.css:629`).
**Fix:** only hide the native cursor after the custom cursor is successfully initialized (e.g. add a `js-cursor-ready` class to `<body>` from JS, and scope `cursor:none` to that class).

### 20. Icon-only links without accessible names
- **`research.html:432-445`** and **`advisor.html:234-253`** — footer social links are icon-only with **no text, no `title`, no `aria-label`**; screen readers announce nothing.
- Other pages use `title="…"` which is acceptable but `aria-label` is more reliable.
- Decorative Font Awesome `<i>` icons across the site should have `aria-hidden="true"`.
**Fix:** add `aria-label` to every icon-only control; `aria-hidden="true"` on decorative icons.

### 21. Heading-hierarchy skips
- `contact.html` — `<h4>`/`<h3>` cards appear *before* the section `<h2>` at `:143`.
- `advisor.html:199` `<h2>` → `:205/:211` `<h4>` (skips `<h3>`).
- `protocols.html` — protocol titles `<h3>` jump to `<h5>`/`<h6>` for visual weight, not semantics.
**Fix:** use headings for structure (h2→h3→h4 in order); style with CSS, not by picking a smaller heading tag.

---

## ⚪ P4 — Code quality & maintainability

### 22. Duplicated nav/footer markup on all 10 pages
The 10-item nav and the footer are copy-pasted into every page, so any change means editing 10 files. This already caused drift (see #26, #27).
**Fix:** since this is plain static HTML, inject the nav/footer with a small shared JS include, or adopt a lightweight static-site generator / build step (e.g. 11ty) to keep one source of truth.

### 23. `resources.html` — 193 near-identical tool cards + 193 repeated inline styles
**File:** `resources.html`
- The inline style `style="color: var(--secondary-color); font-size: 0.9rem;"` is repeated **193 times** — should be a single CSS class.
- 193 near-identical 11-line `.tool-item` blocks make the file 3,116 lines / ~243 KB.
**Fix:** move the styling to a class; render the tool list from a JSON array via JS (or a build step).

### 24. `resources.html` — 20 empty description paragraphs
Every "course" card has a blank `<p></p>` (e.g. `:365, :378, :391, :704, :904, :1832…`, 20 total). Either fill them in or remove them.

### 25. CSS is a 10.5 k-line append-only pile
**Files:** `styles.css` (1,766) + `styles-extended.css` (10,501).
- **505 `!important`** (73 + 432) — ~1 every 24 lines in the extended file; specificity is being brute-forced.
- **878 hardcoded hex colors** coexisting with a real `:root` variable system (`var()` used 710×) — the dark theme is built from ~880 hardcoded `[data-theme='dark']` overrides instead of variable swaps.
- Duplicated blocks: "Research Timeline (global copy)" appears twice (`styles-extended.css:243` & `:438`); "Shared section separator" twice (`:227` & `:422`).
- 63 `@media` queries with no consistent breakpoint scale (480/500/560/600/640/720/768/820/860/900/960/999/1000/1024/1100 all appear).
**Fix (incremental):** consolidate the dark theme into variable overrides, delete duplicated blocks, replace `!important` where cascade can be fixed, standardize on ~4 breakpoints. This is a gradual refactor, not urgent, but it's what makes future edits risky.

### 26. Footer label drift: "Tools & Resources" vs nav "Resources & Tools"
Nav says **"Resources & Tools"** but the footer says **"Tools & Resources"** on multiple pages (`protocols.html:1008/:1053`, `resources.html`). Pick one wording.

### 27. `contact.html` footer nav has an extra item
`contact.html:221-225` Navigation column lists 5 items (adds "Teaching") vs the 4-item column on `teaching.html`/`gallery.html`. Standardize.

### 28. Two different footer implementations
`research.html` and `advisor.html` footers use inline styles (`research.html:425-430`, `advisor.html:228-232`) instead of the `footer-brand-*` CSS classes used by `about.html`/`publications.html`. Unify.

### 29. Leftover debug + fragile JS
- `assets/js/script.js:90` — `console.log('Research Portfolio JavaScript loaded successfully');` should be removed.
- `script.js:136` — `navToggle.addEventListener(...)` with no null check; if a page lacks `.nav-toggle` this throws and halts the rest of the script.
- `script.js:347` — uses deprecated `pageYOffset` (use `window.scrollY`).
- The mousemove parallax (`:231`) and multiple `scroll` listeners are unthrottled — add `requestAnimationFrame`/throttle for smoothness.

### 30. `target="_blank"` without `rel="noopener noreferrer"`
Inconsistent site-wide (security/tabnabbing + perf):
- **`protocols.html`** — 15 external links miss `rel` (`:231, :315, :349, :404, :443, :444, :535, :583, :641, :758, :759, :813, :814, :868, :869`).
- **`research.html:434`**, **`advisor.html:237`**, **`contact.html:204`** — Scholar footer link has `target="_blank"` but no `rel`.
- `resources.html` gets this right on all 193 links — use it as the template.

---

## 🟣 P5 — Repo hygiene / cleanup (~30 MB removable)

### 31. Orphaned / duplicate images never referenced anywhere
Confirmed not referenced in any HTML/CSS/JS:

| File | Size | Note |
|------|-----:|------|
| `images/C elegans under Fluorescent Microscope.gif` | **22 MB** | unused GIF; page uses the 5.2 MB `.jpg` |
| `images/gallery/MS Thesis 2.JPG` | **2.9 MB** | unused |
| `Thesis Presentation.jpg` (repo root) | **2.5 MB** | duplicate of `images/gallery/` copy |
| `images/Md Mustak khan.png` | **1.6 MB** | duplicate of the used 156 KB `Md. Mustak khan.png` (note the period) |
| `images/profile-placeholder.png` | 532 KB | unused |
| `logo.png` (root) | 516 KB | unused |
| `images/1logo.png` | 516 KB | unused |
| `images/Untitled.png` | 508 KB | unused |
| `images/biopc_logo.png` | 288 KB | unused |
| `images/University_logo.svg` | 88 KB | unused |
| `images/department_logo.jfif` | 36 KB | unused |

**~31 MB** total. Verify each is truly unwanted, then delete. **Before deleting, confirm you have originals backed up** — some (e.g. the 22 MB microscopy GIF) may be source material you want to keep outside the web repo.

### 32. Stray junk file
`documents/dh` — a 1 KB text file containing `jjkjkjd` (a stray test file). Delete.

### 33. Unused "Design Element" folder
`Design Element/` contains 4 large HTML design-library files (animation/box/dark-mode/timeline "copy-ready specs"). **Not referenced by any page.** These look like design scratchpads — move them out of the published repo (or into a `_design/` folder that isn't deployed).

### 34. No `README.md` or `.gitignore`
Add a short `README.md` (what the site is, how to run/deploy) and a `.gitignore`. A `.nojekyll` file is also worth adding for GitHub Pages if you ever use folders starting with `_`.

---

## Content accuracy fixes

| Issue | Location | Fix |
|-------|----------|-----|
| Typo "Antihelmentic" | `research.html:278` | → **"Anthelmintic"** |
| Lab name inconsistency: "Applied Bioscience" (3×) vs "Applied Biosciences" (24×) | `research.html:155/156/231` | Standardize on **"Applied Biosciences"** (plural) |
| "Bioicon" | `resources.html:2779/2782` | → **"Bioicons"** (bioicons.com) |
| Truncated sentence: "Data and performs statistical tests efficiently." | `resources.html:843` (researchpy) | Rewrite the description |
| "ADMETsar 2.0" casing | `resources.html:1636` | Standardize to **"admetSAR"** |
| Likely-dead external links: `pfam.xfam.org` (retired → InterPro), `oncomine.org` (discontinued) | `resources.html:1058/2177/2385` | Verify & update; both still marked "Verified Resource" |
| TOC vs heading mismatch ("Python & ML Tools" vs "Data Science Tools") | `resources.html:166` vs `:769` | Make them match |
| Hardcoded "© 2026" | all page footers | `script.js` already computes the year on some pages — apply the dynamic-year snippet everywhere, or accept it updates each Jan |
| Advisor "placeholder" image filenames | `advisor.html:101/142` | Confirm `advisor1/2-placeholder.png` are the final photos or rename |
| Unescaped `&` in markup | `advisor.html:53/270` | Use `&amp;` for consistency/validity |

---

## Suggested action plan (phased)

**Phase 1 — Correctness (a few hours)**
1. Fix the contact form (#1) — highest user impact.
2. Add `</html>` to `protocols.html` (#2).
3. Fix the `#academic` anchor (#3).
4. Fix typos & name inconsistencies (content table).
5. Remove `console.log` and add null checks in `script.js` (#29).

**Phase 2 — Performance (half a day, biggest visible win)**
6. Delete orphaned/duplicate images (#31–33) → repo drops ~30 MB instantly.
7. Compress + generate thumbnails for the gallery and `research.html` images (#4, #5).
8. Make a real small favicon + optimized logo + 1200×630 OG image (#6).
9. Add `width`/`height` to all `<img>`; remove lazy-load on hero images (#8, #9).

**Phase 3 — SEO & Accessibility (half a day)**
10. Add canonical tags, JSON-LD, sitemap.xml, robots.txt, 404.html (#10–13).
11. Add OG tags to research/contact; fix resources/gallery OG issues (#12).
12. Make hamburger + gallery thumbnails keyboard-accessible (#15, #16).
13. Add `prefers-reduced-motion` + visible focus styles (#17, #18).
14. Add `aria-label` to icon-only links; `aria-hidden` to decorative icons (#20).

**Phase 4 — Maintainability (ongoing)**
15. De-duplicate nav/footer via a shared include or build step (#22).
16. Templatize the 193 resource cards (#23).
17. Gradually consolidate CSS: kill duplicate blocks, reduce `!important`, variable-ize the dark theme (#25).

---

## What's already good (keep it)

- Clean, consistent 10-item navigation and a coherent visual system across all pages.
- `<meta name="description">` on **all 10 pages**; solid Open Graph on 8/10.
- A "Skip to main content" link (`index.html:31`) — good baseline accessibility.
- All 24 gallery images have **descriptive `alt` text**; lightbox has `role="dialog"`/`aria-modal` and Esc/arrow/swipe support.
- Filter counts on gallery/resources/protocols are internally accurate.
- Advisor images have a clever `onerror` fallback to initials.
- Publications metrics match the cards; the contact email is consistent everywhere.
- `resources.html` correctly uses `rel="noopener noreferrer"` on all 193 external links — the pattern to copy elsewhere.
