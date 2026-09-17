# Design Review: Notes board ("לוח כתיבה") — gallery, page view, new-board modal

Reviewed against: no formal `DESIGN_BRIEF.md` exists for this feature — the
design was approved incrementally in chat (2026-09-17) against a short
proposal list, with decisions recorded in `frontend/NOTES.md` ("In Progress /
Next Up" → the three 2026-09-17 notes-board entries, and the relevant
Decisions rows). This review treats those NOTES.md entries as the brief.

Philosophy: Notion-inspired polish pass — subtler chrome, discoverable
per-item actions, a document-like reading feel — applied to an existing
nested-outline notes board, not a from-scratch design.

Date: 2026-09-17

## Screenshots Captured

Captured live via the built-in browser pane against a local static server
(`python3 -m http.server 8765`), with the real sign-in gate bypassed by
direct DOM manipulation (Google FedCM cannot be automated in this
environment — see `NOTES.md` Watch Out #20) and representative state
seeded via injected JS that mirrors the real render functions' output
shape. Screenshots were reviewed inline in the tool session; this
environment has no mechanism to export them to disk, so no PNG files are
attached under `screenshots/` — the descriptions below are from direct
visual inspection at capture time, not reconstructed from memory.

| State | Viewport | Result |
| --- | --- | --- |
| Gallery, 4 cards (icon, empty title, long title, collapsed-marker preview) | 800×563 (desktop) | Clean; RTL grid order correct (first-inserted card renders rightmost) |
| Gallery, same 4 cards | 375×812 (mobile) | **Found real overflow bug** (below), fixed and re-verified clean |
| Empty gallery state | 800×563 (desktop) | Clean; dashed border, centered copy |
| Page view (title, icon, breadcrumb, h1/h2, nested block, bold/italic/underline) | 800×563 (desktop) | Clean |
| Page view, same content | 375×812 (mobile) | **Found invisible in-page menu icon** (below), fixed and re-verified visible with correct 18px box |
| "⋯" dropdown (שכפול / שינוי שם / מחיקה) | 800×563 (desktop) | Clean; reuses `.picker` popover, matches existing highlight-filter menu styling |
| New-board modal (type cards + name field) | 800×563 (desktop) | Clean |
| New-board modal | 375×812 (mobile) | Clean; cards stack, no overflow |

## Summary

The notes-board feature is visually consistent with the rest of the app
(shares design tokens, `.picker` popover component, existing button/card
patterns) and the Notion-inspired polish items (pill button, hover lift,
breadcrumb, per-page icon, typography) read as intended on desktop. Mobile
testing surfaced two real, user-facing defects that code review alone would
not have caught: a horizontal-overflow bug in the gallery grid on narrow
screens, and a completely invisible icon on the in-page "⋯" menu (present
on **every** screen size, not mobile-only — a genuine, if subtle,
functionality-discoverability bug, not just polish). Both are fixed and
re-verified in this session (commit `47729e5`). A separate, pre-existing
mobile overflow bug was found in the **planner** (non-notes) header's
`.week-nav` — out of scope for this review (unrelated to today's work,
and already correctly hidden for notes boards) but worth a heads-up.

## Must Fix

1. **Gallery cards forced horizontal overflow on mobile (375px)** — `.notes-card`
   (`index.html:735`) had no `min-width:0`. As a CSS Grid item, its
   "automatic minimum size" defaulted to its content's min-content width;
   with `white-space:nowrap` on `.notes-card-title`, a long unbroken title
   forced the whole card (and grid track) to 411px in a 359px-wide
   single-column track, causing the entire gallery to scroll horizontally.
   Confirmed via `document.body.scrollWidth` (431 before, 359 = viewport
   width after). **Fixed**: added `min-width:0` to `.notes-card`.
2. **In-page "⋯" menu icon rendered at 0×0 — invisible on every viewport,
   not just mobile** — `.notes-page-menu-btn`'s `<svg>` (`index.html:1002`)
   has no explicit `width`/`height` attributes, and no CSS rule ever sized
   it (unlike the `.ob-toolbar` icon buttons, which set `width="18"
   height="18"` inline, or the gallery card's plain-text `⋯`). Measured
   `getBoundingClientRect()` on the SVG: `{width:0, height:0}`. This means
   the duplicate/rename/delete menu inside an open page has been
   undiscoverable since it was wired up earlier in this session — the
   button is clickable (padding gives it a small hit area) but shows
   nothing. **Fixed**: added `.notes-page-menu-btn svg{ width:18px;
   height:18px; display:block; }` and `display:flex; align-items:center`
   on the button itself for correct centering.

## Should Fix

3. **No visible formatting UI on desktop at all** (raised directly by the
   user mid-review, confirmed by code inspection: `index.html:847`,
   `@media (pointer:fine){ .ob-toolbar{ display:none !important; } }`).
   Every formatting action — heading level, bold/italic/underline,
   indent/outdent, move up/down, collapse, delete — is **only** reachable
   via keyboard shortcut or markdown-style typing (`# `, `## `) on a mouse
   + keyboard device; the `.ob-toolbar` that exposes these as buttons is
   deliberately hidden outside `pointer:coarse`. A first-time user with a
   mouse has no way to discover that headings, bold, or indent exist. This
   is a real gap, not a cosmetic one — flagged here as a finding, but
   fixing it is a small scoped feature (a persistent or selection-triggered
   toolbar, plus a proper block-type dropdown instead of the current H-key
   cycle-through-three-styles button) that needs its own short design
   proposal before implementation, matching how the rest of today's
   UI changes were approved. Not implemented in this pass.
4. **Long, unbroken board names in the drawer / board switcher** were not
   covered by this review's screenshots — worth a quick pass if boards
   ever get long names, using the same `min-width:0` pattern just fixed
   in finding #1 as a guide if a similar overflow shows up there.

## Could Improve

5. The empty-title card fallback ("ללא כותרת") and the long-title
   truncation now both render correctly, but there's no `title=` tooltip
   attribute on the truncated card title, so a user can't hover to see
   the full name without opening the page. Minor; Notion does show a
   native tooltip here.
6. `.notes-card-menu` (gallery) and `.notes-page-menu-btn` (page view) use
   two different icon styles for the same action (plain `⋯` text vs. an
   SVG three-dot icon). Not wrong, but unifying to one would be slightly
   more consistent — low priority, no functional impact.

## What Works Well

- The `.picker` popover component reuse for the new שכפול/שינוי שם/מחיקה
  menu is exactly right — no new UI primitive, consistent positioning
  logic (`positionPopover`, flips above the anchor near the bottom of the
  viewport), consistent with the existing highlight-filter menu.
- RTL grid ordering, breadcrumb direction, and heading hierarchy all read
  correctly without any special-casing beyond what the existing
  `dir="rtl"` root already provides.
- The rename interaction (open page, select the title's full text) avoids
  building a whole separate rename modal/prompt — reuses the title field
  that was already editable.
- Touch-target handling for the gallery "⋯" (always visible under
  `pointer:coarse`, hover-revealed under `pointer:fine`) correctly follows
  the app's existing established pattern for exactly this kind of control
  (`.row-del`, `.drag-handle`), rather than inventing a new convention.
- The mobile new-board modal and gallery empty-state both degrade cleanly
  with no changes needed.

## Out of scope, flagged for awareness

**Pre-existing mobile overflow in the planner (non-notes) header.** At
375px, `.week-nav` (`index.html:99`, day/week nav + view toggle) measures
434px wide — wider than the viewport — forcing the whole page (not just the
header) to scroll horizontally on a **planner-type** board. This is
unrelated to today's notes-board work (not touched this session) and does
not affect notes boards, which already hide `.week-nav` entirely
(`renderAll()`, `index.html:2162`). Confirmed by hiding `.week-nav`
manually: overflow disappears completely (`innerWidth` 456→375). Not fixed
in this pass — flag for a separate, explicit go-ahead before touching the
planner view's header layout.
