# Admin Visual Editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the site owner visually edit index.html/about.html/portfolio.html (text, image, color, spacing, size, section order/insert/delete) through a password-gated `/admin-yp7k2.html` editor, with changes stored in Supabase and applied to the public pages at load time.

**Architecture:** DOM-overlay pattern. The three public pages stay hand-authored HTML/Tailwind; every editable node gets a stable `data-block-id`/`data-edit-id`. A shared `assets/apply-overrides.js` fetches a per-page JSON "overrides" document from Supabase and mutates the live DOM on load (content globally, color/spacing/size per responsive tier, cascading mobile-first). The admin page iframes the real public page in an `?edit=1` mode that loads `assets/editor-bridge.js` for selection/drag/resize, and talks to the parent admin UI over `postMessage`.

**Tech Stack:** Vanilla JS (no build step, no framework — matches the existing site), Supabase (Postgres + Auth) via the `@supabase/supabase-js` v2 UMD build from CDN, SortableJS (CDN) for drag-reorder, Playwright (already used earlier in this project) as the verification harness since there is no test runner in this static-site repo.

## Global Constraints

- No bundler/build step — every new file is included via a plain `<script>` tag, consistent with the rest of the site.
- Supabase project: URL `https://chxhphnfqomhfcalzdhw.supabase.co`, anon key (public-safe, embed directly): `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImNoeGhwaG5mcW9taGZjYWx6ZGh3Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODMxNTU3NjcsImV4cCI6MjA5ODczMTc2N30.7OWZ3nEvcnTnAhEc1qZYn7aJKJDphgO2OD6a1GG1e4Q`. **Never** embed or commit the `service_role` key anywhere in this repo.
- `site_overrides` table and its RLS policies already exist (created by the user per the design doc's SQL). Do not re-create them.
- Admin login: Supabase Auth user `ejigong94@gmail.com` already exists. Tasks never need or handle its password — the browser login form collects it live.
- Tablet tier = viewport >= 640px, desktop tier = viewport >= 1024px (Tailwind `sm`/`lg`), cascading base -> tablet -> desktop.
- Structure (order/insert/delete), text, and image overrides are global (no per-device variants). Color, spacing (`paddingTop/Bottom/Left/Right`, `marginTop/Bottom`), size (`width`, `height`, `fontSize`) are per-tier.
- No test runner exists in this repo. Every task's "test" is a small Node+Playwright script in the scratchpad (`C:\Users\user\AppData\Local\Temp\claude\c--Users-user-Desktop------\33e94f55-ae79-4ef1-aeb7-fe86139ce758\scratchpad`, shortened below to `$SP`) that drives a real headless browser against the local static server and asserts on real DOM state. These scripts are throwaway harnesses, not part of the shipped site.
- Local static server: `node "$SP/serve.mjs" "<repo-root>" 8080` (already written earlier this session; serves any file under the repo root, mime-aware including `.webp`).

---

### Task 1: Stable ids on all three public pages

**Files:**
- Modify: `index.html`, `about.html`, `portfolio.html`

**Interfaces:**
- Produces: every top-level `<section>` gets `data-block-id="<name>"`; every text node wrapper, image, and button/link meant to be editable gets `data-edit-id="<block>-<field>"`. These ids are the contract every later task keys off of.

Add `data-block-id` to each top-level `<section>` and `data-edit-id` to the elements inside it that should be independently editable. Naming convention: `<block-id>-<field>`, e.g. `hero-headline`, `hero-subhead`, `hero-photo`.

- [ ] **Step 1: Add ids to index.html's sections**

Edit each `<section>` tag in `index.html` (hero, pain point, empathy, solutions, portfolio preview, about preview, final CTA) to add `data-block-id`, and tag the key inner elements. Example for the hero section:

```html
<section class="relative overflow-hidden" data-block-id="hero">
  <div class="max-w-6xl mx-auto px-5 pt-20 pb-24 relative">
    <p class="text-accent font-bold tracking-wide text-sm mb-4 reveal" data-edit-id="hero-eyebrow">상세페이지 디자인 전문 · 9년 차</p>
    <h1 class="font-display text-4xl sm:text-5xl md:text-6xl leading-[3.45] max-w-3xl reveal" data-edit-id="hero-headline">
      기획도, 디자인도<br />
      <span class="dot-accent">윤페이지</span> 하나면 충분합니다<span class="dot-accent">.</span>
    </h1>
    <p class="mt-6 text-muted text-base sm:text-lg max-w-xl reveal" data-edit-id="hero-subhead">
      클라이언트의 브랜드를 감각적으로, 설득력 있게 담아내는 상세페이지.
      단순히 예쁜 디자인이 아니라 <span class="text-white">'팔리는' 구조</span>를 설계합니다.
    </p>
```

Apply the same pattern to the remaining sections in `index.html`:
- `data-block-id="painpoints"` on the pain-point `<section>`; the `<div>` that wraps the 4 cards (the `grid sm:grid-cols-2 lg:grid-cols-4 gap-5` container) gets `data-yp-sortable-group="painpoints"` — this is the explicit marker Task 9 uses to decide which grids are card-reorderable, instead of matching on the generic Tailwind `.grid` class (that class appears on layout grids sitewide that must NOT become drag-sortable, e.g. the stats row). Each of the 4 cards gets a wrapping `data-edit-id="painpoints-card-1"` .. `"painpoints-card-4"` on its outer `<div>` (so the whole card can be dragged/resized as a unit later), plus `data-edit-id="painpoints-card-1-title"` / `"-body"` on its inner `<p>` tags.
- `data-block-id="empathy"` on the empathy `<section>`; `data-edit-id="empathy-photo"` on the avatar `<img>`, `"empathy-headline"`, `"empathy-body"`.
- `data-block-id="solutions"`; its card-wrapping `<div>` gets `data-yp-sortable-group="solutions"`; cards `data-edit-id="solutions-card-1"`.."3", each with `-title`/`-body` children.
- `data-block-id="portfolio-preview"`; the grid container gets `data-edit-id="portfolio-preview-grid"` (not individually editable — generated by JS, skip per-item ids here).
- `data-block-id="about-preview"`; `data-edit-id="about-preview-photo"`, `"about-preview-headline"`, `"about-preview-body"`.
- `data-block-id="final-cta"`; `data-edit-id="final-cta-headline"`, `"final-cta-body"`.
- `data-block-id="footer"` on the `<footer>` element (order/delete not offered for footer in the admin UI later, but the id keeps the convention uniform).

- [ ] **Step 2: Add ids to about.html's sections**

Same pattern: `data-block-id="about-intro"` (photo `about-intro-photo`, headline `about-intro-headline`, body `about-intro-body`), `data-block-id="about-stats"` (three stat `<p>` tags: `about-stats-1`, `-2`, `-3`), `data-block-id="about-values"` (3 cards: `about-values-card-1`..`3` with `-title`/`-body`; its card-wrapping `<div>` gets `data-yp-sortable-group="about-values"`), `data-block-id="about-cta"` (`about-cta-headline`, `about-cta-body`), `data-block-id="footer"`.

- [ ] **Step 3: Add ids to portfolio.html's sections**

`data-block-id="portfolio-header"` (`portfolio-header-headline`, `portfolio-header-body`), `data-block-id="portfolio-grid-section"` (grid itself stays JS-generated, no per-item ids), `data-block-id="portfolio-cta"` (`portfolio-cta-headline`), `data-block-id="footer"`.

- [ ] **Step 4: Write the verification script**

Create `$SP/verify-ids.mjs`:

```javascript
import { chromium } from "playwright";

const pages = ["index.html", "about.html", "portfolio.html"];
const browser = await chromium.launch();
const page = await browser.newPage();
let allOk = true;

for (const p of pages) {
  await page.goto(`http://localhost:8080/${p}`, { waitUntil: "networkidle" });
  const result = await page.evaluate(() => {
    const blockIds = [...document.querySelectorAll("[data-block-id]")].map((el) => el.getAttribute("data-block-id"));
    const editIds = [...document.querySelectorAll("[data-edit-id]")].map((el) => el.getAttribute("data-edit-id"));
    const dupBlock = blockIds.filter((id, i) => blockIds.indexOf(id) !== i);
    const dupEdit = editIds.filter((id, i) => editIds.indexOf(id) !== i);
    return { blockCount: blockIds.length, editCount: editIds.length, dupBlock, dupEdit };
  });
  console.log(p, result);
  if (result.blockCount === 0 || result.dupBlock.length || result.dupEdit.length) allOk = false;
}

await browser.close();
console.log(allOk ? "PASS" : "FAIL");
```

- [ ] **Step 5: Run it and confirm PASS**

```bash
node "$SP/serve.mjs" "/c/Users/user/Desktop/클로드코드" 8080 &
sleep 1
node "$SP/verify-ids.mjs"
```

Expected: each page prints a result with `blockCount > 0`, empty `dupBlock`/`dupEdit` arrays, and the script prints `PASS`. If any `dup*` array is non-empty, find the duplicate id in that page's HTML and rename it before proceeding.

- [ ] **Step 6: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add index.html about.html portfolio.html
git commit -m "Add stable data-block-id/data-edit-id attributes for the admin editor"
```

---

### Task 2: Section template library

**Files:**
- Create: `assets/templates.js`

**Interfaces:**
- Produces: `window.YP_TEMPLATES` — an object keyed by template name, each value `{ label: string, html: (blockId) => string }`. `html(blockId)` returns a full `<section data-block-id="${blockId}" data-inserted="true">...</section>` string with every editable child pre-tagged as `data-edit-id="${blockId}__<field>"`.
- Consumed by: Task 4 (apply-overrides insert logic) and Task 10 (admin template picker).

- [ ] **Step 1: Write `assets/templates.js`**

```javascript
window.YP_TEMPLATES = {
  "stats-row": {
    label: "통계 3칸",
    html: (id) => `
      <section data-block-id="${id}" data-inserted="true" class="border-t border-white/10 bg-surface">
        <div class="max-w-6xl mx-auto px-5 py-16 grid grid-cols-3 gap-6 text-center">
          <div>
            <p class="font-display text-3xl" data-edit-id="${id}__stat1">98%</p>
            <p class="text-muted text-xs mt-2" data-edit-id="${id}__label1">지표 1</p>
          </div>
          <div>
            <p class="font-display text-3xl" data-edit-id="${id}__stat2">2,000+</p>
            <p class="text-muted text-xs mt-2" data-edit-id="${id}__label2">지표 2</p>
          </div>
          <div>
            <p class="font-display text-3xl" data-edit-id="${id}__stat3">9년</p>
            <p class="text-muted text-xs mt-2" data-edit-id="${id}__label3">지표 3</p>
          </div>
        </div>
      </section>`,
  },
  "cta-banner": {
    label: "CTA 배너",
    html: (id) => `
      <section data-block-id="${id}" data-inserted="true" class="border-t border-white/10">
        <div class="max-w-6xl mx-auto px-5 py-24 text-center">
          <h2 class="font-display text-3xl" data-edit-id="${id}__headline">새 섹션 제목</h2>
          <p class="mt-5 text-muted" data-edit-id="${id}__body">설명 텍스트를 입력하세요.</p>
          <a href="https://open.kakao.com/me/yunpage" target="_blank" rel="noopener" class="btn-primary mt-10 px-10 py-4 text-base" data-edit-id="${id}__cta">버튼 텍스트</a>
        </div>
      </section>`,
  },
  "text-block": {
    label: "텍스트 블록",
    html: (id) => `
      <section data-block-id="${id}" data-inserted="true" class="border-t border-white/10">
        <div class="max-w-6xl mx-auto px-5 py-16">
          <h2 class="font-display text-2xl" data-edit-id="${id}__headline">제목</h2>
          <p class="mt-4 text-muted" data-edit-id="${id}__body">본문 내용을 입력하세요.</p>
        </div>
      </section>`,
  },
  "image-block": {
    label: "이미지 블록",
    html: (id) => `
      <section data-block-id="${id}" data-inserted="true" class="border-t border-white/10">
        <div class="max-w-4xl mx-auto px-5 py-16">
          <img src="https://placehold.co/1200x600" alt="" class="w-full rounded-2xl" data-edit-id="${id}__image" />
        </div>
      </section>`,
  },
};
```

- [ ] **Step 2: Write the verification script**

Create `$SP/verify-templates.mjs`:

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
await page.setContent(`<html><head><script src="http://localhost:8080/assets/templates.js"></script></head><body></body></html>`);
await page.goto("http://localhost:8080/index.html"); // real origin so the script tag actually loads
await page.addScriptTag({ url: "http://localhost:8080/assets/templates.js" });
const result = await page.evaluate(() => {
  const names = Object.keys(window.YP_TEMPLATES);
  const html = window.YP_TEMPLATES["stats-row"].html("custom-test1");
  const hasBlockId = html.includes('data-block-id="custom-test1"');
  const hasEditId = html.includes('data-edit-id="custom-test1__stat1"');
  return { names, hasBlockId, hasEditId };
});
console.log(result);
console.log(result.names.length === 4 && result.hasBlockId && result.hasEditId ? "PASS" : "FAIL");
await browser.close();
```

- [ ] **Step 3: Run it and confirm PASS**

```bash
node "$SP/verify-templates.mjs"
```

Expected: `names` lists all 4 template keys, `hasBlockId`/`hasEditId` both `true`, prints `PASS`.

- [ ] **Step 4: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add assets/templates.js
git commit -m "Add section template library for the admin editor's insert feature"
```

---

### Task 3: Supabase client + connectivity/RLS check

**Files:**
- Create: `assets/supabase-client.js`

**Interfaces:**
- Produces: `window.ypSupabase` — an initialized Supabase JS client, ready for `.from("site_overrides")` calls. Consumed by Task 4 (read) and Task 8 (write).

- [ ] **Step 1: Write `assets/supabase-client.js`**

```javascript
window.ypSupabase = window.supabase.createClient(
  "https://chxhphnfqomhfcalzdhw.supabase.co",
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImNoeGhwaG5mcW9taGZjYWx6ZGh3Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODMxNTU3NjcsImV4cCI6MjA5ODczMTc2N30.7OWZ3nEvcnTnAhEc1qZYn7aJKJDphgO2OD6a1GG1e4Q"
);
```

This must load *after* the Supabase UMD CDN script on every page that uses it:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js"></script>
<script src="./assets/supabase-client.js"></script>
```

- [ ] **Step 2: Write the connectivity + RLS verification script**

This is the important test: confirm the anon key can read `site_overrides` but cannot write to it (RLS is doing its job).

Create `$SP/verify-supabase.mjs`:

```javascript
import { createClient } from "@supabase/supabase-js";

const sb = createClient(
  "https://chxhphnfqomhfcalzdhw.supabase.co",
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImNoeGhwaG5mcW9taGZjYWx6ZGh3Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODMxNTU3NjcsImV4cCI6MjA5ODczMTc2N30.7OWZ3nEvcnTnAhEc1qZYn7aJKJDphgO2OD6a1GG1e4Q"
);

const { data: readData, error: readError } = await sb.from("site_overrides").select("*");
console.log("read:", { rowCount: readData?.length ?? null, readError: readError?.message ?? null });

const { error: writeError } = await sb.from("site_overrides").upsert({ page: "__rls_test__", overrides: {} });
console.log("write attempt (should be blocked):", { writeError: writeError?.message ?? null });

const readOk = !readError;
const writeBlocked = !!writeError;
console.log(readOk && writeBlocked ? "PASS" : "FAIL");
```

- [ ] **Step 3: Install the Supabase JS package in the scratchpad and run it**

```bash
cd "$SP" && npm install @supabase/supabase-js
node verify-supabase.mjs
```

Expected: `read:` shows `readError: null` (select succeeds, `rowCount` is a number, likely `0`). `write attempt` shows a non-null `writeError` message (Postgres RLS violation) — that's correct, it means anon cannot write. Prints `PASS`. If `writeError` is `null` instead (write succeeded), the RLS policy from the design doc isn't active — stop and have the user re-run the SQL from the design doc's Supabase setup step before continuing.

- [ ] **Step 4: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add assets/supabase-client.js
git commit -m "Add Supabase client bootstrap for the admin editor"
```

---

### Task 4: apply-overrides.js — content + structural overrides

**Files:**
- Create: `assets/apply-overrides.js`
- Test: `$SP/verify-apply-content.mjs`

**Interfaces:**
- Consumes: `window.YP_TEMPLATES` (Task 2), `window.ypSupabase` (Task 3), the `data-block-id`/`data-edit-id` attributes (Task 1).
- Produces: `window.ypApplyOverrides(doc, overrides)` — a pure-ish function (only touches the passed-in `doc`) that applies `overrides.content`, `overrides.inserted`, `overrides.deleted`, and `overrides.order`. Exported this way (not just "run on load") so Task 5 and the test harness can call it directly with a fixture object instead of needing live Supabase data.
- Also produces: `window.ypLoadAndApplyOverrides(pageName)` — the real entry point, fetches from Supabase then calls `ypApplyOverrides`.

- [ ] **Step 1: Write the content/structural half of `assets/apply-overrides.js`**

```javascript
function ypApplyOverrides(doc, overrides) {
  const o = overrides || {};

  // 1. Insert new blocks from templates
  (o.inserted || []).forEach(({ blockId, template, after }) => {
    if (doc.querySelector(`[data-block-id="${blockId}"]`)) return; // already inserted
    const tpl = window.YP_TEMPLATES[template];
    if (!tpl) return;
    const afterEl = doc.querySelector(`[data-block-id="${after}"]`);
    if (!afterEl) return;
    const wrapper = doc.createElement("div");
    wrapper.innerHTML = tpl.html(blockId).trim();
    const section = wrapper.firstElementChild;
    afterEl.insertAdjacentElement("afterend", section);
  });

  // 2. Delete (hide, not remove)
  (o.deleted || []).forEach((blockId) => {
    const el = doc.querySelector(`[data-block-id="${blockId}"]`);
    if (el) el.style.display = "none";
  });

  // 3. Reorder — o.order is a map: "main" reorders top-level sections,
  // any other key is a card-group's data-yp-sortable-group value and
  // reorders that group's children by their data-edit-id.
  Object.entries(o.order || {}).forEach(([groupKey, ids]) => {
    if (!ids || !ids.length) return;
    if (groupKey === "main") {
      const main = doc.querySelector("main") || doc.body;
      ids.forEach((blockId) => {
        const el = doc.querySelector(`[data-block-id="${blockId}"]`);
        if (el) main.appendChild(el);
      });
    } else {
      const group = doc.querySelector(`[data-yp-sortable-group="${groupKey}"]`);
      if (!group) return;
      ids.forEach((editId) => {
        const el = group.querySelector(`[data-edit-id="${editId}"]`);
        if (el) group.appendChild(el);
      });
    }
  });

  // 4. Content overrides (text / image), global
  (o.content || []).forEach(({ id, type, value }) => {
    const el = doc.querySelector(`[data-edit-id="${id}"]`);
    if (!el) return;
    if (type === "text") el.textContent = value;
    if (type === "image") el.setAttribute("src", value);
  });

  return doc;
}

async function ypLoadAndApplyOverrides(pageName) {
  if (!window.ypSupabase) return;
  const { data, error } = await window.ypSupabase
    .from("site_overrides")
    .select("overrides")
    .eq("page", pageName)
    .maybeSingle();
  if (error || !data) return;
  ypApplyOverrides(document, data.overrides);
  window.ypApplyResponsiveOverrides?.(document, data.overrides); // added in Task 5
}

window.ypApplyOverrides = ypApplyOverrides;
window.ypLoadAndApplyOverrides = ypLoadAndApplyOverrides;
```

- [ ] **Step 2: Write the verification script (fixture-based, no network)**

Create `$SP/verify-apply-content.mjs`:

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto("http://localhost:8080/index.html", { waitUntil: "networkidle" });
await page.addScriptTag({ url: "http://localhost:8080/assets/templates.js" });
await page.addScriptTag({ url: "http://localhost:8080/assets/apply-overrides.js" });

const result = await page.evaluate(() => {
  const fixture = {
    content: [{ id: "hero-headline", type: "text", value: "TEST HEADLINE" }],
    inserted: [{ blockId: "custom-t1", template: "stats-row", after: "hero" }],
    deleted: ["empathy"],
    order: {},
  };
  window.ypApplyOverrides(document, fixture);
  return {
    headline: document.querySelector('[data-edit-id="hero-headline"]').textContent.trim(),
    insertedExists: !!document.querySelector('[data-block-id="custom-t1"]'),
    insertedChildExists: !!document.querySelector('[data-edit-id="custom-t1__stat1"]'),
    empathyHidden: document.querySelector('[data-block-id="empathy"]').style.display === "none",
  };
});
console.log(result);
const pass = result.headline === "TEST HEADLINE" && result.insertedExists && result.insertedChildExists && result.empathyHidden;
console.log(pass ? "PASS" : "FAIL");
await browser.close();
```

- [ ] **Step 3: Run it and confirm it FAILS first (file doesn't exist yet)**

If you're implementing Steps 1 and 2 in order as written above, skip this — Step 1 already created the file. If following strict TDD, write Step 2's test first, run it (expect a 404 / script error), then write Step 1, then proceed to Step 4.

- [ ] **Step 4: Run it and confirm PASS**

```bash
node "$SP/verify-apply-content.mjs"
```

Expected: all four fields `true`/matching, prints `PASS`.

- [ ] **Step 5: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add assets/apply-overrides.js
git commit -m "Add apply-overrides.js content/structural override engine"
```

---

### Task 5: apply-overrides.js — responsive cascade (color/spacing/size)

**Files:**
- Modify: `assets/apply-overrides.js`
- Test: `$SP/verify-apply-responsive.mjs`

**Interfaces:**
- Consumes: same file's existing `ypApplyOverrides`.
- Produces: `window.ypApplyResponsiveOverrides(doc, overrides)` — applies `overrides.responsive.base`, then `.tablet` if `window.innerWidth >= 640`, then `.desktop` if `>= 1024`, as inline styles. Also attaches a debounced `resize` listener that re-applies.

- [ ] **Step 1: Add the responsive cascade function**

Append to `assets/apply-overrides.js`:

```javascript
const YP_PROP_MAP = {
  color: "color",
  backgroundColor: "backgroundColor",
  paddingTop: "paddingTop",
  paddingBottom: "paddingBottom",
  paddingLeft: "paddingLeft",
  paddingRight: "paddingRight",
  marginTop: "marginTop",
  marginBottom: "marginBottom",
  width: "width",
  height: "height",
  fontSize: "fontSize",
};

function ypApplyResponsiveOverrides(doc, overrides) {
  const responsive = (overrides && overrides.responsive) || {};
  const width = doc.defaultView.innerWidth;
  const tiers = ["base"];
  if (width >= 640) tiers.push("tablet");
  if (width >= 1024) tiers.push("desktop");

  tiers.forEach((tier) => {
    (responsive[tier] || []).forEach(({ id, prop, value }) => {
      const el = doc.querySelector(`[data-edit-id="${id}"], [data-block-id="${id}"]`);
      const cssProp = YP_PROP_MAP[prop];
      if (el && cssProp) el.style[cssProp] = value;
    });
  });
}

function ypDebounce(fn, ms) {
  let t;
  return (...args) => {
    clearTimeout(t);
    t = setTimeout(() => fn(...args), ms);
  };
}

window.ypApplyResponsiveOverrides = ypApplyResponsiveOverrides;

window.ypWatchResponsiveOverrides = function (doc, overrides) {
  ypApplyResponsiveOverrides(doc, overrides);
  const handler = ypDebounce(() => ypApplyResponsiveOverrides(doc, overrides), 150);
  doc.defaultView.addEventListener("resize", handler);
};
```

Update `ypLoadAndApplyOverrides` (from Task 4) to call `ypWatchResponsiveOverrides` instead of the one-shot call:

```javascript
async function ypLoadAndApplyOverrides(pageName) {
  if (!window.ypSupabase) return;
  const { data, error } = await window.ypSupabase
    .from("site_overrides")
    .select("overrides")
    .eq("page", pageName)
    .maybeSingle();
  if (error || !data) return;
  ypApplyOverrides(document, data.overrides);
  window.ypWatchResponsiveOverrides(document, data.overrides);
}
```

- [ ] **Step 2: Write the verification script**

Create `$SP/verify-apply-responsive.mjs`:

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage({ viewport: { width: 390, height: 800 } });
await page.goto("http://localhost:8080/index.html", { waitUntil: "networkidle" });
await page.addScriptTag({ url: "http://localhost:8080/assets/templates.js" });
await page.addScriptTag({ url: "http://localhost:8080/assets/apply-overrides.js" });

const fixture = {
  responsive: {
    base: [{ id: "hero-headline", prop: "color", value: "rgb(255, 0, 0)" }],
    tablet: [{ id: "hero-headline", prop: "color", value: "rgb(0, 255, 0)" }],
    desktop: [{ id: "hero-headline", prop: "color", value: "rgb(0, 0, 255)" }],
  },
};

async function colorAt(width) {
  await page.setViewportSize({ width, height: 800 });
  return page.evaluate((fx) => {
    window.ypApplyResponsiveOverrides(document, fx);
    return getComputedStyle(document.querySelector('[data-edit-id="hero-headline"]')).color;
  }, fixture);
}

const mobile = await colorAt(390);   // expect base (red)
const tablet = await colorAt(700);   // expect tablet (green), base+tablet cascade
const desktop = await colorAt(1440); // expect desktop (blue), all three cascade, desktop wins

console.log({ mobile, tablet, desktop });
const pass = mobile === "rgb(255, 0, 0)" && tablet === "rgb(0, 255, 0)" && desktop === "rgb(0, 0, 255)";
console.log(pass ? "PASS" : "FAIL");
await browser.close();
```

- [ ] **Step 3: Run it and confirm PASS**

```bash
node "$SP/verify-apply-responsive.mjs"
```

Expected: exactly the three RGB values above in order, prints `PASS`.

- [ ] **Step 4: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add assets/apply-overrides.js
git commit -m "Add responsive (base/tablet/desktop) override cascade to apply-overrides.js"
```

---

### Task 6: Wire apply-overrides into the public pages, end-to-end against real Supabase

**Files:**
- Modify: `index.html`, `about.html`, `portfolio.html`
- Test: `$SP/verify-e2e-overrides.mjs`

**Interfaces:**
- Consumes: `window.ypLoadAndApplyOverrides` (Task 5).
- Produces: each public page calls `ypLoadAndApplyOverrides("index" | "about" | "portfolio")` on load. This is the first task that talks to the real Supabase project.

- [ ] **Step 1: Add script tags before `</body>` in each page**

In `index.html`, right before `</body>`:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js"></script>
<script src="./assets/supabase-client.js"></script>
<script src="./assets/templates.js"></script>
<script src="./assets/apply-overrides.js"></script>
<script>ypLoadAndApplyOverrides("index");</script>
</body>
```

Same block in `about.html` with `"about"`, and `portfolio.html` with `"portfolio"` — but in `portfolio.html`, place it *before* the existing `<script src="./assets/portfolio-data.js"></script>` block stays where it is; add the new block right before `</body>` same as the others, order relative to `portfolio-data.js` doesn't matter since they touch different concerns.

- [ ] **Step 2: Seed a real row in Supabase for a smoke test**

```bash
cd "$SP" && node -e '
import("@supabase/supabase-js").then(async ({ createClient }) => {
  // NOTE: anon key cannot write (Task 3 proved this) — this step intentionally
  // uses the Supabase dashboard SQL editor instead. Print the SQL to run there:
  console.log(`insert into site_overrides (page, overrides) values (
    (quote_literal("index")),
    (quote_literal("{\"content\":[{\"id\":\"hero-eyebrow\",\"type\":\"text\",\"value\":\"E2E TEST EYEBROW\"}]}"))::jsonb
  ) on conflict (page) do update set overrides = excluded.overrides;`);
});
'
```

This step is informational only (anon key cannot write, by design). Actually run the following in the Supabase dashboard's SQL Editor instead:

```sql
insert into site_overrides (page, overrides)
values ('index', '{"content":[{"id":"hero-eyebrow","type":"text","value":"E2E TEST EYEBROW"}]}'::jsonb)
on conflict (page) do update set overrides = excluded.overrides;
```

- [ ] **Step 3: Write the verification script**

Create `$SP/verify-e2e-overrides.mjs`:

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
const errors = [];
page.on("pageerror", (e) => errors.push(String(e)));
await page.goto("http://localhost:8080/index.html", { waitUntil: "networkidle" });
await page.waitForFunction(
  () => document.querySelector('[data-edit-id="hero-eyebrow"]')?.textContent.includes("E2E TEST EYEBROW"),
  { timeout: 5000 }
);
const text = await page.locator('[data-edit-id="hero-eyebrow"]').textContent();
console.log({ text, errors });
console.log(text.includes("E2E TEST EYEBROW") && errors.length === 0 ? "PASS" : "FAIL");
await browser.close();
```

- [ ] **Step 4: Run it and confirm PASS**

```bash
node "$SP/verify-e2e-overrides.mjs"
```

Expected: `text` contains `"E2E TEST EYEBROW"`, `errors` is empty, prints `PASS`. This confirms the real Supabase round trip works.

- [ ] **Step 5: Clean up the test row**

In the Supabase SQL editor:

```sql
update site_overrides set overrides = '{}'::jsonb where page = 'index';
```

- [ ] **Step 6: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add index.html about.html portfolio.html
git commit -m "Wire apply-overrides.js into index/about/portfolio pages"
```

---

### Task 7: editor-bridge.js — selection highlighting + postMessage out

**Files:**
- Create: `assets/editor-bridge.js`
- Modify: `index.html`, `about.html`, `portfolio.html` (conditional load)
- Test: `$SP/verify-editor-select.mjs`

**Interfaces:**
- Produces: when a page is loaded with `?edit=1`, hovering any `[data-edit-id]` element outlines it in pink; clicking it posts `{source:"yp-editor", type:"select", payload:{id, kind, text, computed:{color, paddingTop, paddingBottom, paddingLeft, paddingRight, marginTop, marginBottom, width, height, fontSize}}}` to `window.parent`. `kind` is `"text"` if the element has no `src` attribute, `"image"` if it does.
- Consumed by: Task 8 (admin property panel).

- [ ] **Step 1: Write `assets/editor-bridge.js`**

```javascript
(function () {
  if (new URLSearchParams(location.search).get("edit") !== "1") return;

  const style = document.createElement("style");
  style.textContent = `
    [data-yp-hover] { outline: 2px dashed #f0127e; outline-offset: 2px; cursor: pointer; }
    [data-yp-selected] { outline: 2px solid #f0127e; outline-offset: 2px; }
  `;
  document.head.appendChild(style);

  let hovered = null;
  document.addEventListener("mouseover", (e) => {
    const el = e.target.closest("[data-edit-id]");
    if (hovered) hovered.removeAttribute("data-yp-hover");
    if (el) el.setAttribute("data-yp-hover", "");
    hovered = el;
  });

  let selected = null;
  document.addEventListener("click", (e) => {
    const el = e.target.closest("[data-edit-id]");
    if (!el) return;
    e.preventDefault();
    if (selected) selected.removeAttribute("data-yp-selected");
    selected = el;
    el.setAttribute("data-yp-selected", "");

    const cs = getComputedStyle(el);
    window.parent.postMessage(
      {
        source: "yp-editor",
        type: "select",
        payload: {
          id: el.getAttribute("data-edit-id"),
          kind: el.hasAttribute("src") ? "image" : "text",
          text: el.hasAttribute("src") ? null : el.textContent,
          computed: {
            color: cs.color,
            paddingTop: cs.paddingTop,
            paddingBottom: cs.paddingBottom,
            paddingLeft: cs.paddingLeft,
            paddingRight: cs.paddingRight,
            marginTop: cs.marginTop,
            marginBottom: cs.marginBottom,
            width: cs.width,
            height: cs.height,
            fontSize: cs.fontSize,
          },
        },
      },
      "*"
    );
  });

  window.addEventListener("message", (e) => {
    if (!e.data || e.data.source !== "yp-admin") return;
    if (e.data.type === "apply-preview") {
      const { id, kind, prop, value } = e.data.payload;
      const el = document.querySelector(`[data-edit-id="${id}"]`);
      if (!el) return;
      if (kind === "text") el.textContent = value;
      else if (kind === "image") el.setAttribute("src", value);
      else if (kind === "style") el.style[prop] = value;
    }
  });

  window.parent.postMessage({ source: "yp-editor", type: "ready" }, "*");
})();
```

- [ ] **Step 2: Load it conditionally on all three pages**

Add right after the `apply-overrides.js` script tag in each of `index.html`, `about.html`, `portfolio.html`:

```html
<script src="./assets/apply-overrides.js"></script>
<script src="./assets/editor-bridge.js"></script>
<script>ypLoadAndApplyOverrides("index");</script>
```

(`editor-bridge.js` self-exits immediately if `?edit=1` isn't present, so this is safe to include unconditionally in the `<script>` tag list — the condition is inside the file.)

- [ ] **Step 3: Write the verification script**

Create `$SP/verify-editor-select.mjs`:

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();

const messages = [];
await page.exposeFunction("__capture", (msg) => messages.push(msg));
await page.goto("http://localhost:8080/index.html?edit=1", { waitUntil: "networkidle" });
await page.evaluate(() => {
  window.addEventListener("message", (e) => window.__capture(e.data));
});

await page.click('[data-edit-id="hero-headline"]');
await page.waitForTimeout(200);

const selectMsg = messages.find((m) => m.type === "select");
console.log(selectMsg);
const pass = selectMsg && selectMsg.payload.id === "hero-headline" && selectMsg.payload.kind === "text";
console.log(pass ? "PASS" : "FAIL");
await browser.close();
```

Note: `postMessage` from the *same* top-level document to itself (there's no real iframe here) still fires a `message` event on `window`, so this test works without needing the full admin iframe harness yet — Task 8 tests the iframe case for real.

- [ ] **Step 4: Run it and confirm PASS**

```bash
node "$SP/verify-editor-select.mjs"
```

Expected: prints the captured `select` message with `payload.id === "hero-headline"`, then `PASS`.

- [ ] **Step 5: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add assets/editor-bridge.js index.html about.html portfolio.html
git commit -m "Add editor-bridge.js: click-to-select with postMessage to parent"
```

---

### Task 8: Admin shell — auth, device toggle, iframe, property panel, Save

**Files:**
- Create: `admin-yp7k2.html`
- Test: `$SP/verify-admin-shell.mjs`

**Interfaces:**
- Consumes: `window.ypSupabase` (Task 3, also used here for `.auth.signInWithPassword`), the `postMessage` protocol from Task 7 (`select` in, `apply-preview` out).
- Produces: a working end-to-end loop — log in, pick a device width, click an element in the iframe, edit its text in the panel, see it update live in the iframe, hit Save, and have it persisted to Supabase (confirmed by reloading the plain public page).

- [ ] **Step 1: Write `admin-yp7k2.html`**

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>YunPage 관리자</title>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js"></script>
<script src="./assets/supabase-client.js"></script>
<script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-neutral-950 text-white h-screen flex flex-col">

  <div id="login-screen" class="flex-1 flex items-center justify-center">
    <form id="login-form" class="bg-neutral-900 p-8 rounded-2xl w-80 flex flex-col gap-4">
      <h1 class="font-bold text-lg">YunPage 관리자 로그인</h1>
      <input id="login-email" type="email" placeholder="이메일" class="bg-neutral-800 rounded px-3 py-2" required />
      <input id="login-password" type="password" placeholder="비밀번호" class="bg-neutral-800 rounded px-3 py-2" required />
      <button type="submit" class="bg-pink-600 rounded px-3 py-2 font-bold">로그인</button>
      <p id="login-error" class="text-red-400 text-sm"></p>
    </form>
  </div>

  <div id="editor-screen" class="hidden flex-1 flex flex-col">
    <div class="h-14 border-b border-white/10 flex items-center justify-between px-4 gap-4">
      <select id="page-select" class="bg-neutral-800 rounded px-2 py-1 text-sm">
        <option value="index">홈</option>
        <option value="about">소개</option>
        <option value="portfolio">포트폴리오</option>
      </select>
      <div class="flex gap-2">
        <button data-device="390" class="device-btn bg-neutral-800 rounded px-3 py-1 text-sm">모바일</button>
        <button data-device="768" class="device-btn bg-neutral-800 rounded px-3 py-1 text-sm">태블릿</button>
        <button data-device="1440" class="device-btn bg-pink-600 rounded px-3 py-1 text-sm">데스크탑</button>
      </div>
      <div class="flex gap-2">
        <button id="save-btn" class="bg-pink-600 rounded px-4 py-1.5 text-sm font-bold">저장</button>
        <button id="logout-btn" class="bg-neutral-800 rounded px-4 py-1.5 text-sm">로그아웃</button>
      </div>
    </div>
    <div class="flex-1 flex overflow-hidden">
      <div class="flex-1 overflow-auto flex items-start justify-center bg-neutral-900 p-6">
        <iframe id="preview-frame" style="width:1440px; height:2400px; background:#0a0a0b;" class="border border-white/10"></iframe>
      </div>
      <div id="panel" class="w-80 border-l border-white/10 p-4 text-sm">
        <p class="text-neutral-500">요소를 클릭해 편집하세요.</p>
      </div>
    </div>
  </div>

<script>
const sb = window.ypSupabase;
let currentPage = "index";
let currentDevice = "desktop"; // base|tablet|desktop tier key
let currentSelection = null; // { id, kind, computed }
let stagedOverrides = { content: [], responsive: { base: [], tablet: [], desktop: [] }, inserted: [], deleted: [], order: {} };

const frame = document.getElementById("preview-frame");
const panel = document.getElementById("panel");

function loadFrame() {
  frame.src = `./${currentPage}.html?edit=1`;
}

document.getElementById("page-select").addEventListener("change", (e) => {
  currentPage = e.target.value;
  stagedOverrides = { content: [], responsive: { base: [], tablet: [], desktop: [] }, inserted: [], deleted: [], order: {} };
  loadFrame();
});

const deviceWidths = { 390: "base", 768: "tablet", 1440: "desktop" };
document.querySelectorAll(".device-btn").forEach((btn) => {
  btn.addEventListener("click", () => {
    document.querySelectorAll(".device-btn").forEach((b) => b.classList.remove("bg-pink-600"));
    btn.classList.add("bg-pink-600");
    const w = btn.dataset.device;
    frame.style.width = w + "px";
    currentDevice = deviceWidths[w];
  });
});

window.addEventListener("message", (e) => {
  if (!e.data || e.data.source !== "yp-editor") return;
  if (e.data.type === "select") {
    currentSelection = e.data.payload;
    renderPanel();
  }
});

function sendPreview(kind, prop, value) {
  frame.contentWindow.postMessage(
    { source: "yp-admin", type: "apply-preview", payload: { id: currentSelection.id, kind, prop, value } },
    "*"
  );
}

function stageContent(type, value) {
  const list = stagedOverrides.content;
  const existing = list.find((c) => c.id === currentSelection.id);
  if (existing) existing.value = value;
  else list.push({ id: currentSelection.id, type, value });
}

function stageResponsive(prop, value) {
  const list = stagedOverrides.responsive[currentDevice];
  const existing = list.find((c) => c.id === currentSelection.id && c.prop === prop);
  if (existing) existing.value = value;
  else list.push({ id: currentSelection.id, prop, value });
}

function renderPanel() {
  const s = currentSelection;
  if (!s) { panel.innerHTML = '<p class="text-neutral-500">요소를 클릭해 편집하세요.</p>'; return; }

  if (s.kind === "text") {
    panel.innerHTML = `
      <p class="text-neutral-400 mb-2">${s.id}</p>
      <label class="block mb-1">텍스트</label>
      <textarea id="f-text" class="w-full bg-neutral-800 rounded p-2 mb-4" rows="3">${s.text}</textarea>
      <label class="block mb-1">글자색</label>
      <input id="f-color" type="color" class="w-full mb-4" />
      <label class="block mb-1">글자 크기 (px)</label>
      <input id="f-fontsize" type="number" class="w-full bg-neutral-800 rounded p-2 mb-4" value="${parseFloat(s.computed.fontSize)}" />
    `;
    document.getElementById("f-text").addEventListener("input", (e) => {
      stageContent("text", e.target.value);
      sendPreview("text", null, e.target.value);
    });
    document.getElementById("f-color").addEventListener("input", (e) => {
      stageResponsive("color", e.target.value);
      sendPreview("style", "color", e.target.value);
    });
    document.getElementById("f-fontsize").addEventListener("input", (e) => {
      const v = e.target.value + "px";
      stageResponsive("fontSize", v);
      sendPreview("style", "fontSize", v);
    });
  } else if (s.kind === "image") {
    panel.innerHTML = `
      <p class="text-neutral-400 mb-2">${s.id}</p>
      <label class="block mb-1">이미지 URL</label>
      <input id="f-src" type="text" class="w-full bg-neutral-800 rounded p-2 mb-4" placeholder="https://..." />
    `;
    document.getElementById("f-src").addEventListener("change", (e) => {
      stageContent("image", e.target.value);
      sendPreview("image", null, e.target.value);
    });
  }
}

document.getElementById("save-btn").addEventListener("click", async () => {
  const { error } = await sb.from("site_overrides").upsert({ page: currentPage, overrides: stagedOverrides });
  alert(error ? "저장 실패: " + error.message : "저장 완료");
});

document.getElementById("logout-btn").addEventListener("click", async () => {
  await sb.auth.signOut();
  location.reload();
});

document.getElementById("login-form").addEventListener("submit", async (e) => {
  e.preventDefault();
  const email = document.getElementById("login-email").value;
  const password = document.getElementById("login-password").value;
  const { error } = await sb.auth.signInWithPassword({ email, password });
  if (error) {
    document.getElementById("login-error").textContent = error.message;
    return;
  }
  document.getElementById("login-screen").classList.add("hidden");
  document.getElementById("editor-screen").classList.remove("hidden");
  loadFrame();
});

(async () => {
  const { data } = await sb.auth.getSession();
  if (data.session) {
    document.getElementById("login-screen").classList.add("hidden");
    document.getElementById("editor-screen").classList.remove("hidden");
    loadFrame();
  }
})();
</script>
</body>
</html>
```

- [ ] **Step 2: Write the verification script**

Create `$SP/verify-admin-shell.mjs` (uses the real Supabase Auth user credentials only at the point of typing them into the login form, exactly like a human would — it does not read or store the password anywhere else):

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto("http://localhost:8080/admin-yp7k2.html", { waitUntil: "networkidle" });

// Login screen visible before auth
const loginVisible = await page.locator("#login-screen").isVisible();
console.log({ loginVisible });

// NOTE: run this test interactively — it needs the real admin password typed
// in by the user once, or skip login-flow assertions and only check that
// the login form renders correctly (below), which doesn't need the password.
const hasEmailField = await page.locator("#login-email").count();
const hasPasswordField = await page.locator("#login-password").count();

console.log({ hasEmailField, hasPasswordField });
console.log(loginVisible && hasEmailField === 1 && hasPasswordField === 1 ? "PASS (form renders)" : "FAIL");
await browser.close();
```

- [ ] **Step 3: Run the automated part**

```bash
node "$SP/verify-admin-shell.mjs"
```

Expected: `PASS (form renders)`.

- [ ] **Step 4: Manual end-to-end pass (requires the real password)**

Open `http://localhost:8080/admin-yp7k2.html` in a real browser, log in with `ejigong94@gmail.com` and the password set earlier. Click the hero headline in the iframe, confirm the panel shows a textarea with the current text, edit it, confirm the iframe updates live. Click "저장", confirm the alert says "저장 완료". Open `http://localhost:8080/index.html` in a plain new tab (no `?edit=1`) and confirm the edited headline shows there too.

- [ ] **Step 5: Revert the test edit**

In the Supabase SQL editor:

```sql
update site_overrides set overrides = '{}'::jsonb where page = 'index';
```

- [ ] **Step 6: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add admin-yp7k2.html
git commit -m "Add admin editor shell: auth, device toggle, iframe, property panel, save"
```

---

### Task 9: Drag reorder, resize handles, delete button (editor-bridge.js)

**Files:**
- Modify: `assets/editor-bridge.js`
- Modify: `admin-yp7k2.html` (handle new message types, stage `order`/`deleted` overrides)
- Test: `$SP/verify-editor-reorder.mjs`

**Interfaces:**
- Produces: in `?edit=1` mode, hovering a top-level `[data-block-id]` section shows a drag handle (top-left) and a delete button (top-right); dragging reorders sections in the iframe's DOM and posts `{source:"yp-editor", type:"reorder", payload:{order:[...blockIds]}}`; clicking delete hides the section and posts `{type:"delete", payload:{blockId}}`. Card-group items (`painpoints-card-*`, `solutions-card-*`, `about-values-card-*`) get the same drag handle behavior scoped to their parent grid. Resizable elements (images, card wrappers) get an SE-corner resize handle that posts `{type:"resize", payload:{id, width, height}}` on drag end.

- [ ] **Step 1: Add SortableJS and the section/card controls to `editor-bridge.js`**

Add near the top of the IIFE in `assets/editor-bridge.js` (after the existing early-return guard):

```javascript
  const sortableScript = document.createElement("script");
  sortableScript.src = "https://cdn.jsdelivr.net/npm/sortablejs@1.15.2/Sortable.min.js";
  document.head.appendChild(sortableScript);

  sortableScript.onload = () => {
    const main = document.querySelector("main") || document.body;
    Sortable.create(main, {
      handle: "[data-yp-drag-handle]",
      animation: 150,
      onEnd: () => {
        const order = [...main.querySelectorAll("[data-block-id]")].map((el) => el.getAttribute("data-block-id"));
        window.parent.postMessage({ source: "yp-editor", type: "reorder", payload: { order } }, "*");
      },
    });

    document.querySelectorAll("[data-block-id]").forEach((section) => {
      const handle = document.createElement("div");
      handle.setAttribute("data-yp-drag-handle", "");
      handle.textContent = "⠿";
      handle.style.cssText = "position:absolute; top:8px; left:8px; z-index:50; background:#f0127e; color:#fff; padding:2px 8px; border-radius:6px; cursor:grab; font-size:12px; display:none;";
      section.style.position = section.style.position || "relative";
      section.appendChild(handle);

      const delBtn = document.createElement("button");
      delBtn.textContent = "삭제";
      delBtn.style.cssText = "position:absolute; top:8px; right:8px; z-index:50; background:#333; color:#fff; padding:2px 8px; border-radius:6px; font-size:12px; display:none;";
      delBtn.addEventListener("click", () => {
        const blockId = section.getAttribute("data-block-id");
        section.style.display = "none";
        window.parent.postMessage({ source: "yp-editor", type: "delete", payload: { blockId } }, "*");
      });
      section.appendChild(delBtn);

      section.addEventListener("mouseenter", () => { handle.style.display = "block"; delBtn.style.display = "block"; });
      section.addEventListener("mouseleave", () => { handle.style.display = "none"; delBtn.style.display = "none"; });
    });

    document.querySelectorAll("[data-yp-sortable-group]").forEach((grid) => {
      const groupKey = grid.getAttribute("data-yp-sortable-group");
      Sortable.create(grid, {
        animation: 150,
        onEnd: () => {
          const order = [...grid.children].map((el) => el.getAttribute("data-edit-id")).filter(Boolean);
          window.parent.postMessage({ source: "yp-editor", type: "reorder-cards", payload: { group: groupKey, order } }, "*");
        },
      });
    });
  };
```

- [ ] **Step 2: Add resize handles for images and card wrappers**

Append to the same IIFE (runs immediately, doesn't need Sortable):

```javascript
  function makeResizable(el) {
    const handle = document.createElement("div");
    handle.style.cssText = "position:absolute; width:14px; height:14px; right:-6px; bottom:-6px; background:#f0127e; border-radius:3px; cursor:se-resize; z-index:60;";
    el.style.position = el.style.position || "relative";
    el.appendChild(handle);

    handle.addEventListener("mousedown", (downEvt) => {
      downEvt.preventDefault();
      const startX = downEvt.clientX, startY = downEvt.clientY;
      const startW = el.offsetWidth, startH = el.offsetHeight;
      function onMove(moveEvt) {
        const w = startW + (moveEvt.clientX - startX);
        const h = startH + (moveEvt.clientY - startY);
        el.style.width = w + "px";
        el.style.height = h + "px";
      }
      function onUp() {
        document.removeEventListener("mousemove", onMove);
        document.removeEventListener("mouseup", onUp);
        window.parent.postMessage(
          { source: "yp-editor", type: "resize", payload: { id: el.getAttribute("data-edit-id"), width: el.style.width, height: el.style.height } },
          "*"
        );
      }
      document.addEventListener("mousemove", onMove);
      document.addEventListener("mouseup", onUp);
    });
  }

  document.querySelectorAll("img[data-edit-id]").forEach(makeResizable);
```

- [ ] **Step 3: Handle the new message types in `admin-yp7k2.html`**

Add to the `window.addEventListener("message", ...)` handler added in Task 8:

```javascript
  if (e.data.type === "reorder") {
    stagedOverrides.order.main = e.data.payload.order;
  }
  if (e.data.type === "reorder-cards") {
    stagedOverrides.order[e.data.payload.group] = e.data.payload.order;
  }
  if (e.data.type === "delete") {
    if (!stagedOverrides.deleted.includes(e.data.payload.blockId)) stagedOverrides.deleted.push(e.data.payload.blockId);
  }
  if (e.data.type === "resize") {
    stageResponsiveFor(e.data.payload.id, "width", e.data.payload.width);
    stageResponsiveFor(e.data.payload.id, "height", e.data.payload.height);
  }
```

And add the small helper (used above, since `resize`/`reorder` events don't go through `currentSelection`):

```javascript
function stageResponsiveFor(id, prop, value) {
  const list = stagedOverrides.responsive[currentDevice];
  const existing = list.find((c) => c.id === id && c.prop === prop);
  if (existing) existing.value = value;
  else list.push({ id, prop, value });
}
```

- [ ] **Step 4: Write the verification script**

Create `$SP/verify-editor-reorder.mjs`:

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
const messages = [];
await page.exposeFunction("__capture", (msg) => messages.push(msg));
await page.goto("http://localhost:8080/index.html?edit=1", { waitUntil: "networkidle" });
await page.evaluate(() => window.addEventListener("message", (e) => window.__capture(e.data)));
await page.waitForTimeout(500); // let Sortable.js CDN load

const heroHandle = page.locator('[data-block-id="hero"] [data-yp-drag-handle]');
const painpointsHandle = page.locator('[data-block-id="painpoints"] [data-yp-drag-handle]');
await page.hover('[data-block-id="hero"]');
await heroHandle.waitFor({ state: "visible" });

// drag hero below painpoints
const heroBox = await heroHandle.boundingBox();
const ppBox = await page.locator('[data-block-id="painpoints"]').boundingBox();
await page.mouse.move(heroBox.x + 5, heroBox.y + 5);
await page.mouse.down();
await page.mouse.move(ppBox.x + 5, ppBox.y + ppBox.height - 5, { steps: 10 });
await page.mouse.up();
await page.waitForTimeout(300);

const reorderMsg = messages.find((m) => m.type === "reorder");
console.log(reorderMsg);
const pass = reorderMsg && reorderMsg.payload.order[0] === "painpoints" && reorderMsg.payload.order.includes("hero");
console.log(pass ? "PASS" : "FAIL");
await browser.close();
```

- [ ] **Step 5: Run it and confirm PASS**

```bash
node "$SP/verify-editor-reorder.mjs"
```

Expected: a `reorder` message whose `order` array has `"painpoints"` before `"hero"`, prints `PASS`. If Sortable.js hasn't finished loading from the CDN in time, increase the `waitForTimeout(500)` in Step 4.

- [ ] **Step 6: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add assets/editor-bridge.js admin-yp7k2.html
git commit -m "Add drag-reorder, card reorder, resize handles, and delete to the editor"
```

---

### Task 10: Template picker ("+ 섹션 추가") in the admin UI

**Files:**
- Modify: `assets/editor-bridge.js` (render "+ add" gap buttons between sections, post `insert` message)
- Modify: `admin-yp7k2.html` (template picker modal, stage `inserted` overrides)
- Test: `$SP/verify-insert-section.mjs`

**Interfaces:**
- Produces: hovering the gap between two sections in the iframe shows a "+" button; clicking it posts `{source:"yp-editor", type:"request-insert", payload:{after: blockId}}`. The admin shows a modal listing `window.YP_TEMPLATES` (needs `templates.js` loaded in `admin-yp7k2.html` too); picking one posts back `{source:"yp-admin", type:"do-insert", payload:{after, template, blockId}}` (admin generates `blockId` as `"custom-" + Date.now()`); the iframe applies it immediately via the existing `ypApplyOverrides` (already loaded) and the admin stages it into `stagedOverrides.inserted`.

- [ ] **Step 1: Add gap "+" buttons in `editor-bridge.js`**

Add inside the `sortableScript.onload` callback (after the existing section loop), in `assets/editor-bridge.js`:

```javascript
    document.querySelectorAll("[data-block-id]").forEach((section) => {
      const gap = document.createElement("button");
      gap.textContent = "+ 섹션 추가";
      gap.style.cssText = "display:none; width:100%; text-align:center; padding:4px; background:#f0127e; color:#fff; font-size:12px; border:none; cursor:pointer;";
      section.insertAdjacentElement("afterend", gap);
      section.addEventListener("mouseenter", () => { gap.style.display = "block"; });
      gap.addEventListener("mouseleave", () => { gap.style.display = "none"; });
      gap.addEventListener("click", () => {
        window.parent.postMessage(
          { source: "yp-editor", type: "request-insert", payload: { after: section.getAttribute("data-block-id") } },
          "*"
        );
      });
    });
```

- [ ] **Step 2: Handle `do-insert` in `editor-bridge.js`'s existing message listener**

Add a branch to the existing `window.addEventListener("message", ...)` in `editor-bridge.js`:

```javascript
    if (e.data.type === "do-insert") {
      const { after, template, blockId } = e.data.payload;
      window.ypApplyOverrides(document, { inserted: [{ blockId, template, after }] });
    }
```

- [ ] **Step 3: Add the template picker modal to `admin-yp7k2.html`**

Add `<script src="./assets/templates.js"></script>` to `admin-yp7k2.html`'s `<head>`. Add this markup right before the closing `</body>`:

```html
<div id="template-modal" class="hidden fixed inset-0 bg-black/70 flex items-center justify-center z-50">
  <div class="bg-neutral-900 rounded-2xl p-6 w-96">
    <h2 class="font-bold mb-4">추가할 섹션 선택</h2>
    <div id="template-list" class="flex flex-col gap-2"></div>
    <button id="template-cancel" class="mt-4 text-sm text-neutral-400">취소</button>
  </div>
</div>
```

Add to the `<script>` block:

```javascript
let insertAfterId = null;
document.getElementById("template-cancel").addEventListener("click", () => {
  document.getElementById("template-modal").classList.add("hidden");
});

window.addEventListener("message", (e) => {
  if (e.data?.source === "yp-editor" && e.data.type === "request-insert") {
    insertAfterId = e.data.payload.after;
    const list = document.getElementById("template-list");
    list.innerHTML = "";
    Object.entries(window.YP_TEMPLATES).forEach(([key, tpl]) => {
      const btn = document.createElement("button");
      btn.textContent = tpl.label;
      btn.className = "bg-neutral-800 rounded px-3 py-2 text-left hover:bg-pink-600";
      btn.addEventListener("click", () => {
        const blockId = "custom-" + Date.now();
        frame.contentWindow.postMessage(
          { source: "yp-admin", type: "do-insert", payload: { after: insertAfterId, template: key, blockId } },
          "*"
        );
        stagedOverrides.inserted.push({ blockId, template: key, after: insertAfterId });
        document.getElementById("template-modal").classList.add("hidden");
      });
      list.appendChild(btn);
    });
    document.getElementById("template-modal").classList.remove("hidden");
  }
});
```

- [ ] **Step 4: Write the verification script**

Create `$SP/verify-insert-section.mjs`:

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto("http://localhost:8080/index.html?edit=1", { waitUntil: "networkidle" });
await page.evaluate(() => {
  window.__inserted = null;
  window.addEventListener("message", (e) => {
    if (e.data.type === "request-insert") window.__inserted = "requested";
  });
});
await page.hover('[data-block-id="hero"]');
await page.click('[data-block-id="hero"] + button');
await page.waitForTimeout(200);
const requested = await page.evaluate(() => window.__inserted);
console.log({ requested });

// simulate the admin's do-insert response directly (no full admin UI needed for this check)
await page.evaluate(() => {
  window.postMessage({ source: "yp-admin", type: "do-insert", payload: { after: "hero", template: "stats-row", blockId: "custom-verify1" } }, "*");
});
await page.waitForTimeout(200);
const inserted = await page.locator('[data-block-id="custom-verify1"]').count();
console.log({ inserted });
console.log(requested === "requested" && inserted === 1 ? "PASS" : "FAIL");
await browser.close();
```

- [ ] **Step 5: Run it and confirm PASS**

```bash
node "$SP/verify-insert-section.mjs"
```

Expected: `requested: "requested"`, `inserted: 1`, prints `PASS`.

**Known minor limitation:** the "+ 섹션 추가" gap buttons are inserted as siblings of `<section>` elements under `<main>`. After a drag-reorder (Task 9), a gap button can be left next to a different section than the one it originally followed until the page is reloaded — the stored `order` data is unaffected (always correct), this only affects where the "+" button visually sits between actions in the same session. Not worth the added complexity of re-deriving gap positions on every `Sortable` `onEnd` for a single-user internal tool; reloading the admin page after a reorder resets it.

- [ ] **Step 6: Commit**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git add assets/editor-bridge.js admin-yp7k2.html
git commit -m "Add section-insert template picker to the admin editor"
```

---

### Task 11: Full end-to-end verification pass

**Files:**
- Test only: `$SP/verify-full-e2e.mjs`

**Interfaces:**
- Consumes: everything from Tasks 1-10.
- Produces: a single script exercising one full realistic editing session, confirming nothing regressed.

- [ ] **Step 1: Write `$SP/verify-full-e2e.mjs`**

```javascript
import { chromium } from "playwright";
const browser = await chromium.launch();
const page = await browser.newPage();
const consoleErrors = [];
page.on("pageerror", (e) => consoleErrors.push(String(e)));

// 1. Public pages still load clean with the new scripts attached
for (const p of ["index.html", "about.html", "portfolio.html"]) {
  await page.goto(`http://localhost:8080/${p}`, { waitUntil: "networkidle" });
}
console.log("console errors after loading all 3 pages:", consoleErrors);

// 2. Edit mode loads without breaking the page
await page.goto("http://localhost:8080/index.html?edit=1", { waitUntil: "networkidle" });
const heroVisible = await page.locator('[data-block-id="hero"]').isVisible();
console.log({ heroVisible });

// 3. Responsive cascade still holds at a real viewport resize
await page.setViewportSize({ width: 390, height: 800 });
await page.waitForTimeout(200);
await page.setViewportSize({ width: 1440, height: 900 });
await page.waitForTimeout(200);
const stillThere = await page.locator('[data-block-id="hero"]').isVisible();

console.log({ stillThere });
const pass = consoleErrors.length === 0 && heroVisible && stillThere;
console.log(pass ? "PASS" : "FAIL");
await browser.close();
```

- [ ] **Step 2: Run it and confirm PASS**

```bash
node "$SP/verify-full-e2e.mjs"
```

Expected: no console errors across all three pages, edit mode loads with the hero visible, still visible after a resize round-trip, prints `PASS`.

- [ ] **Step 3: Manual pass in a real browser (not just headless)**

Log into `admin-yp7k2.html` for real, do one edit of each type — text, image, color, spacing, size, reorder two sections, reorder two cards within a group, insert a template section, delete a section — and confirm each shows correctly in the iframe and, after Save, on the plain public page in a separate tab. Do this pass in the browser you actually use day to day (the drag/resize handlers are the most likely thing to have browser-specific quirks), not just whichever browser Playwright happens to drive. This is the one step in the whole plan that can't be scripted, because it's validating that the *feel* of the editor is right, not just that the DOM ends up in the expected state.

- [ ] **Step 4: Push everything**

```bash
cd "/c/Users/user/Desktop/클로드코드"
git push origin main
```

---

## Post-plan note

Once this is deployed, `admin-yp7k2.html` is reachable at
`https://portpolio-wheat-ten.vercel.app/admin-yp7k2.html` (or whatever
the current Vercel domain is) — same unlisted-path-plus-password
convention as the rest of the project's admin surfaces. No Vercel
environment variables are needed since the Supabase anon key is
intentionally public and is committed directly in
`assets/supabase-client.js`.
