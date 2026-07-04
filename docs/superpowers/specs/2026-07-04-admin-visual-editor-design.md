# Admin Visual Editor — Design

Date: 2026-07-04
Status: Approved by user, pending implementation plan

## Purpose

Let the site owner (single user) edit the live YunPage portfolio site
(index.html, about.html, portfolio.html) through a visual, Canva/Claude-Design-like
editor — select an element, change it, see it reflected on the public
site immediately for the next visitor. No hand-editing HTML or
redeploying through git for routine content changes.

## Scope decisions (from brainstorming)

1. **Persistence: real-time.** Saving in the admin writes to a backend
   immediately; any visitor loading the public page after that point sees
   the change. This requires adding a backend (previously the site had
   none) — Supabase.
2. **Editing approach: DOM overlay, not a block-JSON renderer.** The
   existing hand-written HTML/Tailwind pages stay as the source of truth.
   An override/patch layer is stored separately and applied on top at
   runtime. (Trade-off accepted: this is less structurally clean than a
   full block-based CMS and requires care to avoid unbounded patch growth,
   but keeps the existing design/markup intact and was the user's explicit
   choice over the block-CMS alternative.)
3. **Editable properties:** text, image src, section order, section
   insert/delete, element size (via resize handles), card-level reorder
   within a group (e.g. the 4 pain-point cards), color, spacing
   (padding/margin).
4. **Device scope split:**
   - Structure (section order, insert, delete), text content, and image
     src are **global** — same across mobile/tablet/desktop. Rationale:
     a single freelancer portfolio has no real need for per-device copy,
     and letting structure differ per device would make the override
     model combinatorially harder for little payoff.
   - Color, spacing, and size are **per-breakpoint**, cascading
     mobile-first like Tailwind: a value set at "desktop" wins at desktop
     width; if unset, it falls back to "tablet"; if that's unset too, it
     falls back to "base" (mobile default authored in the HTML).
5. **Layout freedom: resize + reorder, not free canvas.** Elements are
   resized via drag handles and reordered via drag-and-drop, but stay
   inside the existing flex/grid flow. Absolute free-position (Figma-style)
   was explicitly rejected — it would require replacing the responsive
   flow layout with per-breakpoint absolute coordinates, which is large
   effort and a poor fit for a responsive marketing site.
6. **New sections come from a template library**, not free-form
   authoring. A fixed set of reusable section templates (hero,
   pain-point-4-card, solutions-3-card, stats-row, portfolio-preview,
   CTA-banner, plain text block, plain image block) can be inserted
   between existing sections. Once inserted, a template's contents are
   ordinary DOM nodes with their own stable ids and are editable through
   the same text/color/image/spacing/size controls as everything else —
   "insert" is the only thing that is template-driven; everything after
   that is uniform.

## Non-goals (explicitly out of scope)

- Offline editing (admin requires network access to Supabase).
- Per-device text or image variants.
- Free/absolute-position canvas editing.
- Multi-user accounts, roles, or edit history/versioning UI (single
  admin account; Supabase row history is not exposed in the UI).
- Real-time push to *already-open* visitor tabs. "Real-time" means the
  next page load reflects the change, not a live socket push to existing
  sessions.

## Architecture

### New pieces

| Piece | Purpose |
|---|---|
| Supabase project | Stores overrides, hosts admin auth |
| `site_overrides` table | `page` (text, PK), `overrides` (jsonb), `updated_at` |
| Supabase Auth | Single admin account (email/password) gating `/admin` |
| `assets/apply-overrides.js` | Included on index/about/portfolio; fetches this page's overrides on load and applies them to the DOM |
| `assets/templates.js` | Shared library of insertable section templates (used by both the admin picker and, indirectly, by whatever inserted the block previously) |
| `admin-yp7k2.html` (obscure path) | The editor UI itself |
| `assets/editor-bridge.js` | Injected into the iframed public page in edit mode; handles selection highlighting, drag handles, resize handles, and posts messages to the parent admin UI |

### Data model

Every editable node in the three public pages gets a stable identifier
baked in at authoring time:
- Top-level `<section>` elements: `data-block-id="hero"`, `data-block-id="painpoints"`, etc.
- Editable children within a section: `data-edit-id="hero-headline"`, `data-edit-id="hero-photo"`, etc.
- Elements inserted from a template at runtime get generated ids scoped
  to the block: `custom-<uuid>`, `custom-<uuid>__title`, etc.

`site_overrides.overrides` (per page) shape:

```jsonc
{
  "order": ["hero", "painpoints", "solutions", "custom-ab12", "cta"],
  "deleted": ["solutions"],
  "inserted": [
    { "blockId": "custom-ab12", "template": "stats-block", "after": "painpoints" }
  ],
  "content": [
    { "id": "hero-headline", "type": "text", "value": "..." },
    { "id": "hero-photo", "type": "image", "value": "https://..." }
  ],
  "responsive": {
    "base":    [{ "id": "hero-headline", "prop": "color", "value": "#fff" }],
    "tablet":  [{ "id": "hero-headline", "prop": "paddingTop", "value": "24px" }],
    "desktop": [{ "id": "hero-headline", "prop": "fontSize", "value": "56px" }]
  }
}
```

`order` and `deleted` are global (one list, not per-device). `content`
(text/image) is global. `responsive` is keyed by breakpoint tier and
cascades base -> tablet -> desktop when applied.

### Runtime application (public pages)

On page load, `apply-overrides.js`:
1. Fetches `site_overrides` row for the current page from Supabase
   (anon/public read via RLS policy — no auth needed to read).
2. Applies `inserted` blocks (cloning templates from `templates.js`,
   assigning scoped ids to their children).
3. Applies `deleted` (hide, `display:none`, not actually removed — keeps
   it recoverable).
4. Applies `order` (reorders section DOM nodes to match).
5. Applies `content` overrides (text/image), global.
6. Computes current breakpoint tier from viewport width, aligned to the
   Tailwind breakpoints already used across the site's classes (`sm:`,
   `md:`, `lg:`): `tablet` tier = viewport >= 640px (Tailwind `sm`),
   `desktop` tier = viewport >= 1024px (Tailwind `lg`). Applies
   `responsive.base` always, then `.tablet` on top if >= 640px, then
   `.desktop` on top of that if >= 1024px, so later tiers win. The admin
   preview widths (390 / 768 / 1440) are representative device sizes
   that land cleanly inside each tier, not the tier boundaries
   themselves.
7. Re-runs step 6 on resize (debounced), so resizing the real browser
   window also re-cascades correctly — not just the admin's iframe.

A brief flash of unedited content before this runs is accepted (low
traffic, single-page-load personal site); no SSR/blocking-render
mechanism is being added for this.

### Admin editor (`admin-yp7k2.html`)

- Gated by Supabase Auth (email/password login form); session persisted
  via Supabase's client-side session, timeout per default Supabase
  session expiry.
- Top bar: Desktop (1440px) / Tablet (768px) / Mobile (390px) toggle
  controlling the width of the iframe wrapper (the iframe itself always
  loads the real page at its natural responsive breakpoints — this just
  resizes the viewport being previewed); Save button; logout.
- The iframe loads the target public page with `?edit=1`, which loads
  `editor-bridge.js` in addition to `apply-overrides.js`. In edit mode:
  - Hovering an element outlines it (pink); clicking selects it and
    posts its id/type/current computed styles to the parent via
    `postMessage`.
  - The parent renders a right-side property panel appropriate to the
    selected element's type: text (inline contenteditable, committed on
    blur), image (URL input / file picker), color (brand palette
    swatches + custom hex), spacing (padding/margin sliders), size
    (width/height, also settable by dragging the element's resize
    handles directly in the iframe).
  - Top-level sections show a drag handle and delete button on hover;
    gaps between sections show a "+" that opens the template picker
    (reads from `templates.js`).
  - Grouped items inside a section (e.g. the 4 pain-point cards) are
    drag-reorderable among themselves the same way.
  - Whichever device tier is currently selected in the top bar
    determines which `responsive` tier color/spacing/size edits are
    written to; text/image/structure edits always write to the global
    fields regardless of tier.
- Save writes the full updated `overrides` JSON for the current page to
  Supabase (upsert by `page`).

### Security

- `/admin-yp7k2.html` is not linked from anywhere public; combined with
  the Supabase Auth login wall, this matches the project's existing
  "obscure path + password" admin convention.
- Supabase RLS: public (anon) role gets `SELECT` only on
  `site_overrides`; `INSERT`/`UPDATE` restricted to the authenticated
  admin user.
- No secrets in source: Supabase URL + anon key are safe to ship
  client-side (that's the anon key's purpose); nothing sensitive is
  stored in `site_overrides` itself.

### Testing / verification plan

- Add `data-block-id` / `data-edit-id` to all three pages; verify no
  visual regression (screenshot diff against current deployed pages).
- Unit-style manual check: apply a hand-written fake overrides JSON via
  `apply-overrides.js` locally (no Supabase yet) and confirm text/image/
  order/insert/delete/responsive-cascade all apply correctly.
- Once Supabase is wired: full round-trip per editable type (text,
  image, color, spacing, size, reorder, insert, delete) — edit in admin,
  reload the public page in a plain browser tab, confirm it stuck.
- Verify the responsive cascade explicitly: set a desktop-only spacing
  override, confirm tablet/mobile still show the base value.
- Cross-browser: Chrome + at least one other (the user's actual daily
  browser) for the admin drag/resize interactions specifically, since
  those are the most likely to have browser-specific quirks.

## Open dependency

Implementing this requires a Supabase project (URL + anon key) and one
admin login (email/password), the same as the Supabase setup flow
already described in `CLAUDE.md` Phase 3 — to be collected at the start
of implementation.
