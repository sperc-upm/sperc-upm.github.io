# Source fixes: changes made to the built site, and what to fix in the Astro source

This repository holds only the **built output** of the SPERC Astro site. The next
`astro build` from the source will overwrite everything below, so each change
needs to be ported into the Astro source as well.

* Part A lists every edit made directly to the built HTML/XML (branch
  `claude/site-improvements`, November 2026). Port these first.
* Part B is the audit: issues found but **not** changed here, because they
  need a source, CSS or data change, or a fact only SPERC can confirm.

This file is not linked from the site and is disallowed in `robots.txt`. You
can delete it once the fixes are in the source.

---

## How the audit was done

* All 83 HTML files (EN, MS, 404, the two `internal/` pages, and four redirect
  stubs) were served locally and loaded in headless Chromium at 1280, 768 and
  375 px. For each page the crawl recorded console errors, failed requests,
  horizontal overflow, broken images, tap-target sizes (375 px) and text
  contrast (1280 px).
* A static pass over every file checked internal links, `#anchor` targets,
  image/`srcset` paths, `lang`, title/description uniqueness, canonical,
  hreflang, Open Graph, JSON-LD, heading order, and `alt`, `width`/`height`
  and `loading` on images.
* Text was extracted from every EN page and compared with its MS twin: all
  numbers match and no untranslated paragraphs were found outside
  publication/grant/award titles, which are not expected to be translated.
* External links could **not** be checked: the audit environment has no
  outbound access to those hosts.

### Before / after (local crawl, 83 pages × 3 widths)

| Check | Before | After |
|---|---|---|
| JS console errors / page errors | 0 / 0 | 0 / 0 |
| Failed same-origin requests | 0 | 0 |
| Pages with horizontal overflow at 375 / 768 px | 0 / 0 | 0 / 0 |
| Broken images | 0 | 0 |
| Internal links to missing pages (static) | 2 (404 page head) | 0 |
| Sitemap entries that are `noindex` redirects | 2 | 0 |
| "1 citations / 1 documents" plural errors | 78 | 0 |
| Plain-`http` links to UPM profiles | 38 links (9 profiles) | 0 |

The site was in good shape before this pass: no broken internal links or
anchors, no failing assets, no JS errors and no mobile overflow.

---

## Part A: edits made to the built output

### A1. `404.html`: removed head tags that point to pages that do not exist
* **What:** removed `<link rel="canonical" href="https://sperc-upm.github.io/404/">`,
  the three `<link rel="alternate" hreflang=…>` tags (`/404/`, `/ms/404/`,
  x-default `/404/`) and `<meta property="og:url" content="https://sperc-upm.github.io/404/">`.
* **Why:** `/404/` and `/ms/404/` do not exist. GitHub Pages serves `404.html`
  for every missing URL, and the page is already `noindex`, so it should not
  declare a canonical or language alternates.
* **Port to source:** in the shared layout/SEO component, skip canonical,
  hreflang and `og:url` when rendering `src/pages/404.astro` (for example, a
  `noindex` or `is404` prop that suppresses them).

### A2. `sitemap-0.xml`: removed two redirect stubs
* **What:** removed the `<url>` entries for `https://sperc-upm.github.io/people/`
  and `https://sperc-upm.github.io/ms/people/`.
* **Why:** both are meta-refresh redirect pages to `/groups/#people` with
  `<meta name="robots" content="noindex">`. Listing `noindex` pages in a
  sitemap sends search engines conflicting signals.
* **Port to source:** in `astro.config.*`, add a `filter` to `@astrojs/sitemap`
  that excludes `/people/`, `/ms/people/` and any other redirect or `noindex`
  page, e.g.
  `sitemap({ filter: (p) => !/\/(ms\/)?people\/(suraya\/)?$/.test(p) && !p.includes('/internal/') })`.

### A3. English pages: "1 citations" / "1 documents" → singular
* **What:**
  * `"; 1 citations"` → `"; 1 citation"` in the publication meta line on
    `groups/bear/`, `groups/ccbu/`, and `people/` `halimatun`, `liyana` (2),
    `razif`, `rezi` (2), `shafreeza`, `wanazlina`, `yusof`, `zurina`
    (12 occurrences).
  * World map on `index.html`: `"<Country>: 1 co-authored documents"` →
    `"… 1 co-authored document"` in the SVG `<title>` and the pin
    `aria-label`, and `<b>1</b> co-authored documents` → `document` in the
    pop-up card (11 countries × 3 places).
  * World map on `partners/index.html`: the same change for `"1 documents"` →
    `"1 document"` (11 countries × 3 places).
* **Why:** grammar. Screen readers read the `aria-label`/`<title>` text aloud.
  The Malay pages (`dokumen`, `sitasi`) do not need this.
* **Port to source:** add a pluralisation helper in the components that
  print these counts: the publication list item (citations), and the
  `WorldMap` component (title, aria-label and card), e.g.
  `` `${n} ${n === 1 ? 'citation' : 'citations'}` `` or `Intl.PluralRules('en')`.

### A4. UPM staff profile links: `http://` → `https://`
* **What:** 38 links to `profile.upm.edu.my` in 28 files (about, the four group
  pages and nine people pages, EN and MS) were changed from `http://` to
  `https://`. The profiles are `asalleh`, `chuah`, `mh_razif`, `mie`,
  `nhafizah` (`profail.html`), `niknorliyana`, `nordin_sab`, `rozitaom` and `zurina`.
* **Why:** the site is served over HTTPS and already links the same host over
  https (`fmy`, `mf_mukhtar`, `shamizhar`, `wanazlina`). Plain-http links
  leak referrers and depend on a redirect.
* **Port to source:** change the `profile` URLs in the people data file.
  Also check `nhafizah/profail.html`: every other profile ends in
  `profile.html` or `professional.html`. It may be correct (Malay "profail"),
  but it is worth clicking once.

### A5. `robots.txt` and this file
* Added `Disallow: /SOURCE-FIXES.md` to `robots.txt` and added this file at
  the repo root. Nothing in the site links to it. This is not needed in the
  Astro source. If you keep the file there, put it outside `public/`.

---

## Part B: recommended fixes in the Astro source (not applied here)

Ordered by priority.

### P1: Accessibility

1. **EN | BM language switch tap targets are too small on mobile.** At
   375 px the links measure 18×42 and 20×42 px, below the WCAG 2.2 AA minimum
   of 24 px. They are the only header controls this small. Add horizontal
   padding (for example `padding-inline: 6px` and `min-width: 24px`) to the
   switch links in the header component (`data-astro-cid-bvxdwma5`).
2. **Contrast failure on `/projects/` and `/ms/projects/`:** the "Research in
   motion / Penyelidikan dalam tindakan" eyebrow is 13 px text in
   `--g:#3F8A4A` on white, which gives 4.25:1 (needs 4.5:1). Darkening it
   to `#387D42` gives 5.0:1 and `#367A40` gives 5.2:1. The value is set inline
   (`style="--th:#C2612C;--bc:#2F8A63;--g:#3F8A4A"`) in the project-activity
   component.
3. **Heading level skip on the four group pages (EN and MS):** `h1` (group
   name) is followed directly by `h3` "What we work on / …" before the first
   `h2` "Focus". Make it an `h2`, or move it under "Focus", and keep the
   visual size with a class.
4. **404 page:** the big decorative "404" (`--fibre #E6D8DA` on white,
   1.38:1) is `aria-hidden` and its count-up already stops under
   `prefers-reduced-motion`. No change is needed. It is mentioned only
   because automated contrast checkers will flag it.
5. Already good, keep it: there is a skip link, every page has `<main
   id="main">`, focus outlines are visible (3 px solid `#A82B3A`), the
   mobile menu is a dialog that returns focus on Esc, SMIL and CSS
   animations are hidden under `prefers-reduced-motion`, and every content
   image has alt text.

### P2: SEO

6. **Thin, duplicated meta descriptions on people pages.** Five CCBU pages
   share "Senior Lecturer, Carbon Capture & Biomass Utilisation Research
   Group", three HyNeX pages share one, and so on (the same pattern on MS
   pages). Build each description from the name plus expertise, e.g.
   "Dr. Izzudin Ibrahim, Senior Lecturer in CCBU at SPERC, UPM: biomass
   carbonization and pyrolysis, oil palm biomass utilisation…". Also, the
   head of centre's description says only "Professor, CCBU". Consider adding
   "Head of Centre".
7. **The 404 page reuses the home page meta description.** Give it its own
   description ("Page not found …").
8. **Redirect stubs** `/people/`, `/people/suraya/`, `/ms/people/`,
   `/ms/people/suraya/` use a relative canonical with a fragment
   (`/groups/#people`). For a `noindex` redirect, drop the canonical, or make it
   absolute and fragment-free (`https://sperc-upm.github.io/groups/`). See
   also item 21 about `suraya`.
9. **JSON-LD:** every page carries the same `ResearchOrganization` block,
   which is valid. To improve it, add `address` (PostalAddress: 43400 UPM
   Serdang, Selangor), `email`, and `sameAs` (UPM and department URLs).
   Use a proper logo of at least 112 px rather than the favicon. Add a
   `Person` block on each profile page (`name`, `jobTitle`, `worksFor` →
   `#org`, `sameAs` → ORCID/Scopus/Google Scholar, `image`).
10. **Open Graph image:** every page uses `/og.png`. Profile pages could use
    the person's portrait, and group pages could use their hero image.
11. **Title nit:** `/about/` is "About SPERC | SPERC". Consider "About | SPERC".
12. Already good: the `lang` attribute is correct on all EN and MS pages,
    reciprocal hreflang (en/ms/x-default) and sitemap `xhtml:link`
    alternates are on every content page, canonicals are absolute and
    self-referencing, `og:locale` and `og:locale:alternate` are swapped
    correctly on MS pages, and the language switch links carry `lang` and
    `hreflang`.

### P2: Performance

13. **The build ships about 62 MB of unreferenced images.** 237 of the 720
    files in `_astro/` are referenced by no HTML, CSS or JS file. These are
    mostly original PNG/JPG sources such as the `l2-p13…p45.*.png` series at
    0.7–1.2 MB each, and `l6-sp_upm_pilot_plant_2024.By71xGnh.jpg` (1.3 MB).
    That is a third of the repo's 178 MB. They are usually emitted when an
    image is `import`ed but rendered only through `<Image>`/`getImage()` with
    a different format, or imported but unused. Remove the unused imports, or
    move unused sources out of `src/`.
14. **The heaviest HTML documents:** `publications/` is 1.78 MB raw (256 KB
    gzipped) with 1,641 list items, and about 400 KB of that is
    `data-astro-cid-*` attributes alone. `projects/` is 760 KB. `index.html`
    and `partners/` are about 510 KB each, mostly the inline SVG world map
    (1,376 paths). Options: paginate the publication list by year, or load
    older years on demand. Use `:global()` or class-based styles in the
    list-item components so scoped-CSS attributes are not stamped on every
    node. Move the world map into one external SVG `<use>` or
    `<img>` with the pins overlaid.
15. **The largest image files actually served:**
    `l6-sp_upm_pilot_plant_2024…_OMTpU.webp` is 575 KB at 4257×1650
    (gallery). `l3-p18_13_384x384…png` (341 KB) and `l3-p17_06_399x361…png`
    (325 KB) are on the impact page, and `l5-p40_05_419x299…png` (283 KB) is
    on projects. These are small PNGs that should be WebP/AVIF at under
    40 KB. Lightbox targets that link straight to the original JPG are
    `l6-sp_ibcex2018_chia_ken_ling…jpg` (470 KB), `569808f649…jpg` (400 KB),
    `ig0887-1…jpg` (366 KB), `news-2025-zurina-searca…jpg` (340 KB) and
    `ig0088-0…jpg` (268 KB). Point the lightbox `href` at a 1600 px WebP
    from `getImage()` instead.
16. Already good: every `<img>` has `width`/`height` (the only exception is the
    empty lightbox placeholder, which is fine). Below-the-fold images are
    `loading="lazy"` and the home hero is `fetchpriority="high"`. Fonts are
    self-hosted woff2 with `font-display: swap`. No render-blocking scripts
    were found: all scripts are `type="module"` at the end of `body`. Each page
    loads 2–5 small CSS files. `build.inlineStylesheets: 'auto'` would remove a
    few round trips.

### P3: Content and data (please verify; nothing here was changed)

17. **Department staff numbers on `/about/` do not add up.** The text says
    "44 full-time staff" and "29 academic staff". The breakdown is 6
    professors + 8 associate professors + 15 senior lecturers (= 29) + 14
    support staff, which totals **43**, not 44.
18. **Grant data shown twice:** on `/people/wanazlina/` (EN and MS) the grant
    reads "Visiting grant on biodegradable film packaging, **Kazakhstan
    Kazakhstan**, 2024". The funder and country fields probably both hold
    "Kazakhstan".
19. **Awards list (`/impact/`):** there are near-duplicate entries for
    Prof. Luqman, e.g. "World's Top 2% Scientists (Stanford University –
    Elsevier), 2025 – Single-year impact" next to "World Top 2% Scientists
    (Citation Impact in Single Calendar Year), Stanford University, USA,
    2019-2025." Formatting is also inconsistent: "INVENTION, INNOVATION &
    DESIGN EXPOSITION 2024 (IIDEX 2024)" is in all caps, and "Competition
    &exhibition 2024(ITEX 24)" is missing spaces. "2025 Short Video
    Competition" names no result.
20. **Grant titles (`/projects/`):** these look like typos carried over from
    the grant records. Fix them only if the official titles differ:
    "Nanosilver Hybride", "mechanism fir strawberry", "Process Integration
    andsupply Chain", "Application of cub-critical water", "Hydrogel
    Derrived Biochar", "From Hospitalwaste", "Castor 01" (Oil?), "Herbicide
    Glyhosate", "Enterrococcus", "anthogenic", "(Sub-Project1 )",
    "Graphene- Based". Possible duplicate grants: "Production of Valuable
    Materials From Sago Bark…" is listed twice for 2017–2019 with different
    capitalisation. "Development of a cost-effective cultivation method for
    microalgae." is listed for 2012, 2013 and 2015. Other titles appear twice
    with different years: "Sustainable synthesis of fluorescent nitrogen
    doped carbon dots…", "Process kinetics and modelling of microalgal
    protein extraction…", "Evaluation and modeling of subcritical water
    extraction…", "Hylocereus Undatus Extract…" and "Development of Bioresin…".
    These could be genuine renewals, so please confirm.
21. **`/people/suraya/` and `/ms/people/suraya/`** are leftover redirect stubs.
    No page links to them and Suraya is not among the 20 listed members.
    If the person has left, delete the source route or turn it into a proper
    redirect in `astro.config` `redirects`. Do not just leave the page.
22. **Partners (Scopus affiliation data):** "University of Western Australia
    (9)" and "The University of Western Australia (6)" are listed as two
    institutions. "Mandsaur of R. V. S. Agriculture University" looks garbled.
    "De La Salle University- Manila" has a stray space. On the home map,
    "SSERL" is shown as a separate partner institution under Palestine. It is
    probably a laboratory at An-Najah National University (already listed), so
    the two may need merging.
23. **Copyright and dates:** there is no copyright line in the footer, so
    nothing is stale. All pages show "Last updated 11 Oct 2026 / Kemas kini
    terakhir 11 Okt 2026" consistently, and the data stamps (Scopus
    29 Sep 2026, SPERC records 7 Oct 2026) are the same on EN and MS. The
    member counts (20 = 7 + 5 + 4 + 4) match across home, about and groups.
    Group totals for publications (1,538) and grants (255) are higher than the
    centre totals (1,357 and 249). The page explains that shared items count
    in each group, so this is consistent.

### P3: `internal/` (members' area)

24. `/internal/` and `/ms/internal/` are **StatiCrypt** password pages. The
    content is AES-encrypted in the HTML and decrypted in the browser. They
    are `noindex,nofollow`, disallowed in `robots.txt`, absent from the
    sitemap and not linked from any public page, so they are not meant to be
    indexed and are not. Note the following:
    * Anyone can download the encrypted file. Protection depends entirely on
      the password's strength, so use a long passphrase and rotate it when
      people leave. "Remember me" keeps the derived key in the visitor's
      browser `localStorage`.
    * Do not place anything there that would be harmful if the password
      leaked. GitHub Pages cannot enforce real access control.
    * `/ms/internal/` is `lang="en"` with bilingual text. That is harmless, but
      it could be `lang="ms"` for consistency.
    * Because `robots.txt` disallows the folder, crawlers never see the
      `noindex` tag. That is fine while nothing links to it. If a link ever
      appears, Google may index the bare URL without content.

### Not verified

* External links (1,421 unique, of which 1,213 are DOIs) could not be
  checked from the audit environment. A link checker such as `lychee` in CI
  would catch broken partner, ORCID and Scopus URLs.
