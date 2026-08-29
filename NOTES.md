# NOTES.md — יומן ומשימות (מערכת לוז ומשימות, אוגדה 98)

Single source of truth for resuming this project cold. Update this file in place whenever architecture, state, or decisions change — don't let it drift.

## Goal

A shared weekly planner/task board for the education staff of אוגדה 98 (Hebrew, RTL) — replaces an original Excel file. **This is a live production tool with real users entering real data**, not a dev sandbox.

- Live site: https://ugda98.github.io/shared-weekly-planner/

## Architecture & Stack

The system spans **two separate local folders on this machine — do not confuse them**:

| Piece | Local path | Repo/tooling |
|---|---|---|
| Frontend | `/Users/createuser/Desktop/מערכת לוז ומשימות/` | git repo `UGDA98/shared-weekly-planner`, branch `main`, deployed via GitHub Pages straight from the repo |
| Backend | `/Users/createuser/Desktop/מערכת לוז ומשימות - גב/` | **not a git repo** — a `clasp`-managed Google Apps Script project (`clasp` is authenticated on this machine) |

**Frontend**: single self-contained `index.html` (2860 lines) — no build step, no external dependencies. Rubik font is embedded as base64 inside the CSS. Entire JS is one IIFE inside one `<script>`; entire CSS is one `<style>` block (`:root` design tokens at lines ~19–56: `--bg`, `--surface`, `--accent`, `--ink`, `--border`, `--radius-*`, etc. — dark-mode CSS exists but is force-disabled in JS, light mode is always active).

**Backend**: Google Apps Script Web App (`קוד.js`, 12.5KB) bound to a Google Sheet.
- Web App URL (fixed, referenced in frontend as `API_URL` at ~[index.html:698](index.html#L698)): `https://script.google.com/macros/s/AKfycbxO8tTn_6c1CxqcFSB35kgjQ3LYrmdzfjixTEdADlLCYyPB7sgL3Kw-UgCdwVXCkUU/exec`
- Script ID: `1ldjlbM_ccZDoF9YuFvtk8_7KuAJnBXA6N9xusBWAFxr5CCnwmNTcPpSN`
- Deploy flow: edit in backend folder → `clasp push` → `clasp version "<description>"` → `clasp deploy --deploymentId AKfycbxO8tTn_6c1CxqcFSB35kgjQ3LYrmdzfjixTEdADlLCYyPB7sgL3Kw-UgCdwVXCkUU --versionNumber <new>`. **Must reuse this exact deploymentId** or the live site's API URL breaks.
- `doGet`/`doPost` route on an `action` param: `listBoards`, `getBoard` (GET); `saveBoard`, `createBoard`, `renameBoard`, `deleteBoard` (POST). No `action` at all routes to `LEGACY_BOARD_ID` (`board-1`) for backward compatibility with old cached tabs.

**Data model (multi-board, schema-less)**: Google Sheet with two tabs — `Boards` (BoardId/Name/DefaultRole/CreatedBy/CreatedAt/State-JSON) and `Config` (Board/Email/Role, per-board access control). Each board's entire app state is one JSON blob that round-trips as-is through `saveBoard`/`getBoard`/`listBoards`/`createBoard`/`renameBoard`/`deleteBoard` — the backend doesn't understand the shape, just stores/returns it.

**Frontend data flow / sync**: poll server every 20s (`POLL_MS`, ~[index.html:702](index.html#L702)) via `refreshFromServer()` (~[index.html:803](index.html#L803)); local edits save to `localStorage` immediately + debounce 600ms before POSTing to server (`persistState`/`pushState`, ~[index.html:752](index.html#L752)/[769](index.html#L769)); immediate flush via `navigator.sendBeacon` on `visibilitychange`/`pagehide`; one-step-back backup always kept in `localStorage` with a "restore from last backup" button in the admin menu.

**Board switching**: `?board=<id>` URL param (read in `resolveInitialBoardId()`, ~[index.html:876](index.html#L876)) takes priority over the last-used board cached in `localStorage`; falls back to `LEGACY_BOARD_ID`. `switchToBoard()` (~[index.html:1149](index.html#L1149)) handles the actual swap.

**Deployment behavior**: frontend push → live in ~1-2 min, but GitHub Pages serves with `max-age=600` (~10 min cache) — a stale-looking page right after push is usually cache, not a bug; hard-refresh or wait.

### Key frontend functions (verify with `grep -n` before editing — line numbers drift)

| Area | Functions | ~Lines |
|---|---|---|
| Data model / migration | `emptyTask`, `migratePersonIds`, `migrateLegacy`, `loadWeek` | 1378–1450 |
| Undo toast | `showUndoToast(message, onUndo)`, `hideUndoToast()` — one toast at a time, ~6s auto-dismiss, wired into all 4 delete sites below | 1323–1350 |
| Filters / summary | `itemMatchesFilter`, `applyFilterVisuals`, `itemMatchesKind`, `collectItemsFor`, `renderSummaryRow`, `renderSummaryGroup`, `renderSummary` | 1563–1916 |
| Task pool | `poolItems`, `renderPoolRow`, `renderPoolCard` | 1768–1871 |
| Person/topic picker | `openPicker(kind, anchorEl, currentId, onPick, opts)` — `opts.multi` for multi-select | 2000–2241 |
| Drag system | `setupDrag(handle, listEl, getArray, rowSelector, onReorder, opts)`, `endActiveDrag`, `updateBoardAutoScroll` (ramped edge-scroll speed, added this session) — `opts.dropzone` for cross-container drag, `opts.threshold` for delayed-drag (e.g. off a checkbox); `dragHandleEl()` | 2163–2340 |
| Collapse (shared) | `readCollapseState`, `writeCollapseState`, `makeCollapsible(headEl, bodyEl, storageKey, itemCount)` — `COLLAPSE_AUTO_THRESHOLD = 6` | 2353–2367 |
| Board render | `renderDay`, `renderDayBanner`, `renderEventRow`, `renderTaskRow`, `renderBoard` — each delete handler now wires `showUndoToast` | 2400–2729 |
| Day title (pinned banner) | `renderDayBanner(dayDef, data)` — renders `.day-banner` (filled, `contentEditable` via `makeEditable`) or `.day-banner-add` (`.add-line`-style, empty state); reads/writes `data.title` on the day object; called from `renderDay` right after the `.day-head` append, before `.zone-events` — sibling of both, so it's invisible to `setupDrag`'s dropzone scan and to the `.zone-events` height-equalization (see Decisions) | 2400–2430 |
| Event-zone height equalization | `equalizeEventZoneHeightsNow`, `equalizeEventZoneHeights`, `watchEventZoneSizes` (ResizeObserver) — `.zone-events` now has a `min-height` CSS transition so corrections settle instead of snapping | 2686–2729 |
| Board list / sync | `fetchBoardsList`, `switchToBoard`, `refreshFromServer`, `startPolling` | 758–1149 |
| Wire-chip-reveal (task metadata) | `wireChipReveal(row, main, isAssigned, labelText)` — also renders the always-visible `.task-flag-label` (assignee name/count) next to the flag dot, not just the flag itself | search `function wireChipReveal` |

### Reusable patterns (use these, don't build parallel mechanisms)

- **Collapse**: any new collapsible UI → use `makeCollapsible`.
- **Drag**: `setupDrag` is the only drag API in the file (`opts.dropzone` for cross-container, `opts.threshold` when the handle overlaps another interactive element like a checkbox).
- **Person/topic tagging**: `makeMetaChip(icon, filled)` + `openPicker(...)`.
- **Destructive row actions**: any new delete affordance → call `showUndoToast(message, onUndo)` instead of `confirm()`/a modal — matches the pattern already used for tasks/events/pool/summary-row deletes (see Decisions).

## Current State

Everything below is built, live, and verified on `main`:

- **Day title (pinned banner)** (this session): each day card can carry a free-text title pinned above the events zone (`data.title` on the day object) — for "the whole day is X" cases (training, leave, etc.) without occupying an hourly event slot. Empty state shows a "+ הוספת כותרת ליום" add-line matching the existing "+ הוספת אירוע"/"+ הוספת משימה" pattern; filled state shows a bold `--accent`/`--accent-ink` banner, always `contentEditable` (same pattern as event/task text); clearing the text reverts it to the add-line automatically. No backend change needed — travels as part of the existing schema-less per-board JSON blob. Backward-compatible via `migrateLegacy`.
- Multi-board system: create/rename/delete boards, per-board editor/viewer roles, board switcher drawer (RTL-correct).
- Person/topic customization: drag-to-reorder, confirm-before-delete when in use.
- Multi-person task assignment (`personId` → `personIds[]`, auto-migrated via `migrateLegacy`).
- Cross-container drag (events/tasks between days, sections, and the task pool) via `data-dropzone` + board auto-scroll at edges.
- Permanent "מאגר משימות" (task pool) card for unscheduled tasks.
- Drag directly from a task's checkbox (not a separate handle) — `setupDrag` with `threshold:7px`, suppresses the synthetic click after a drag.
- Shared collapse/expand for day sections ("משימות"/"צוות") and summary cards ("לפי אדם"/"לפי מופע") — auto-closed above 6 items, manual choice always wins and persists.
- Person/topic tags hidden behind a small dot indicator until clicked (dropdown), to declutter task rows.
- Equal height across all "white events box" zones in a week — handled by a `ResizeObserver` (see Decisions) instead of fixed-timing correction passes; the correction now transitions smoothly instead of snapping (this session).
- Data-loss prevention: `sendBeacon` flush on tab close/hide, one-step-back backup + restore button, **plus a per-action undo toast** (this session) on every task/event/pool/summary-row delete — delete stays instant, ~6s to reverse it.
- Task assignment is always visible, not just on click: a compact name/count label sits next to the flag dot at all times (`.task-flag-label`, this session) — the dot alone used to be the only always-visible signal.
- Section/group item counts (`.collapse-count`, `.summary-group-count`) now decrement on delete, not just increment on add (this session) — they used to go stale until the next full re-render.
- Touch/mobile UX: always-visible delete/rename buttons (not hover-only), `pointer:coarse` breakpoints, popover vertical clamping, no double-scroll, correct RTL nav arrows, checkbox at the 24px WCAG minimum, collapse-toggle hit area grown to 24px via padding+negative-margin (visually unchanged) — all this session's additions on top of the pre-existing touch pass.
- Contrast pass (this session, WCAG AA): `--muted` darkened `#867e97→#6a627d`; done-task text no longer stacks `opacity:.7` on top of it (was ~2.2:1, now passes); unchecked-checkbox border reuses `--muted` instead of the shared `--border-strong` (scoped, so decorative dividers elsewhere stay as light as before); `--today-ring` nudged `#c9922e→#b8842a`, same hue, now passes the 3:1 UI-component minimum.
- Drag feedback and drop-zone highlighting now transition (opacity/background/outline-color, ~150ms) instead of snapping instantly; drag auto-scroll ramps in with proximity to the board edge instead of jumping straight to full speed (this session — motion review, no changes to the reorder/commit logic itself).
- Dark mode forced off (always light).
- Wider day cards (300px, 330px for the weekend/wide day).

**Explicitly rejected, do not re-propose**: a free-text "הערות" (notes) section on the board — raised previously, explicitly removed, nothing implemented. Also rejected this session: replacing the checkbox-as-drag-handle with a separate visible handle icon (would touch working drag wiring for a P3 discoverability issue) — used a `title` tooltip on the checkbox instead.

## Decisions

| Decision | Why | Alternative considered & why rejected |
|---|---|---|
| Single self-contained `index.html`, no build step | Simplicity for a small, single-page tool; easy to deploy straight to GitHub Pages | — |
| Google Sheets (via Apps Script) as the data store, not a real database | Keeps existing org data location, avoids new infra | An earlier, now-abandoned early plan considered calling the Sheets API directly from the frontend — rejected in favor of an Apps Script Web App middle layer for access control per board (`Config` tab) |
| Schema-less JSON blob per board, not normalized rows | Fast to iterate on frontend data shape without backend migrations; backend doesn't need to understand the app's structure | — |
| 20s poll + optimistic local writes + debounce, over websockets/real-time push | Simple, no extra infrastructure; acceptable staleness window for this use case | — |
| Dark mode force-disabled | Product decision for consistency (see git history) | — |
| Multi-board model (`Boards`/`Config` tabs) over one shared sheet | Lets each unit/person have their own board with its own access list | Legacy single-board (`board-1`) kept working via the no-`action` fallback path, for old cached tabs |
| Event-zone height equalization: `ResizeObserver` watching each day's `.events` list, not fixed-timing passes | Investigated a live user report ("boxes go uneven a few seconds after refresh, every time, desktop Chrome") — root cause was that real-world font/text metrics can settle later than all of the fixed correction points (double-rAF, 200ms timeout, `window.load`, `fonts.ready`), leaving the equalized height stale with nothing left to re-correct it. A `ResizeObserver` on the inner `.events` element (not the `.zone-events` wrapper, which carries our own forced `minHeight` and would create a feedback loop) reacts to any future natural-size change, indefinitely | The previous fix (this session's predecessor) added more fixed-timing passes (double-rAF, `setTimeout`, `load`, `fonts.ready`) — insufficient, still had a window where late reflows went uncaught |
| `applyFilterVisuals()` scoped to `#board .task-row[data-id]` | Selector was also matching task-pool rows outside `.day` elements, causing a null-dereference crash on `switchToBoard` | — |
| Testing rule: never test on real boards, always use a disposable scratch board created/deleted via direct API calls | A past incident: testing accidentally happened on the real board "לוז מרכזי - אוריה" and required manual repair | — |
| Delete → undo toast, not a confirm dialog | User's explicit call when both were offered: keeps fast deletes fast, only adds friction (a few seconds' reversibility window) rather than an extra click on every delete | A `showConfirmModal` step before every task/event delete — rejected as too much friction for a frequent action; people/topic deletion still uses confirm since it's rarer and higher-blast-radius (cascades to referencing tasks) |
| `--muted` darkened, checkbox border moved off the shared `--border-strong` onto `--muted` | Contrast audit found `--muted` failing AA (3.4–3.9:1) and the checkbox border failing the 3:1 UI-component minimum (1.57:1); scoping the checkbox fix to `--muted` (not darkening `--border-strong` itself) avoids darkening every decorative divider in the app, which shares that token | Darkening `--border-strong` globally — rejected, too large a visual footprint for a refinement pass, would have darkened dividers that don't need the higher contrast ratio |
| Checkbox keeps double-duty as drag handle; added a `title` tooltip instead of a separate visible handle icon | The checkbox-as-handle is a recent, deliberate, already-shipped decision (see above) — swapping in `dragHandleEl()` for task rows would mean re-wiring `setupDrag`'s target element, risking the working drag system for a P3 (discoverability, not a bug) | A visible `.drag-handle` dot on task rows matching event rows — rejected as touching drag logic for a low-severity, cosmetic-consistency issue |
| CSS transitions added for drag-ghost opacity, drop-zone highlight, and event-zone height correction; auto-scroll speed ramped via a numeric formula change | A dedicated motion review found all three snapping instantly (no `transition` anywhere in the relevant rules) — genuinely jarring/ambiguous during drag, per a live user-facing UX critique. Transitions had to go on the *base* row/container rules, not just the modifier class, since a class-removal transition needs the property present in both the before and after computed style | Re-architecting the drag system (floating drag-ghost following the cursor, FLIP-animated reordering) — explicitly out of scope; this was a "don't touch drag logic" constrained review, transitions/timing only |
| Day title rendered as a sibling of `.day-head`/`.zone-events` (not inside either), with its own `.day-banner` class | Standard calendar UX pattern (Google/Outlook/Apple: a distinct "all-day" area above the timed grid, not a slot). Keeping it out of `.zone-events` means `setupDrag`'s dropzone scan and the `.zone-events` `ResizeObserver`/height-equalization never see it — zero risk to either system | Putting the title inside `.day-head` as a second line — rejected, would force `.day-head` height to vary with title length and complicate the existing header layout; a dedicated sibling element is simpler and fully decoupled |
| Day title always `contentEditable` (mirrors event/task text), not gated behind an edit-pencil button (unlike board rename) | Consistent with how every other piece of day-card text is edited in this app; a pencil-button pattern exists only for the rarer, higher-friction board-rename action | A pencil-button edit affordance like board rename — rejected as unnecessary friction for a same-frequency-as-event-text field |

## In Progress / Next Up

No open technical work as of this writing — the full Impeccable critique/audit cycle (UX critique, WCAG/RTL/touch audit, motion review) that ran this session is closed out and pushed. Waiting on the next change request from the user.

## Open Questions

None outstanding.

## Watch Out

1. **Never test/experiment on real boards.** Always: create a disposable scratch board via a direct `action:'createBoard'` API call, test on it, **delete it when done** (`action:'deleteBoard'`). Login defaults to the last-loaded board, which is sometimes a real one — always confirm the board name in the header/title before touching anything, and navigate via the board-management drawer or `?board=<id>` URL param if needed.
2. **Code deploys (frontend or backend) never touch saved data** — deploy only changes what the code *does* from that point forward. The only risk is manual API calls made while debugging.
3. Git: commit+push at the end of each successful step without waiting for approval — but always ask before merging any PR to `main` (not currently relevant, work happens directly on `main`).
4. Cosmetic/visual changes should go through Plan Mode first (plan → clarify → execute).
5. **Local browser testing**: files outside the session's own project folder (frontend lives in a different folder than the backend, which is the usual cwd) load as a static snapshot with no JS execution when opened directly. Working fix: create a temporary `.claude/launch.json` in the backend folder (the cwd) running `python3 -m http.server --directory <frontend path>`, `preview_start` with that name, then test at `http://localhost:<port>`. **Delete the temporary `.claude/launch.json` when done.** Also: the in-app preview browser can silently serve a *cached* copy of a local page across `navigate()` calls — if the dev server logs don't show a fresh request, force a real refetch with a cache-busting query string (e.g. `?cb=2`).
6. **Line numbers drift** — always re-verify with `grep -n` before editing, don't trust remembered line numbers (including the ones in this file — they were accurate as of this session but will drift).
7. `/Users/createuser/Desktop/מערכת לוז ומשימות - גב/task-platform-handoff.md` is a **stale, outdated handoff doc** from a very early session (describes the project as pre-code, "Hosted vs. Local" undecided, etc.) — already marked with a banner at the top pointing here; this NOTES.md file supersedes it. Don't treat its content as current.
8. The backend folder (`מערכת לוז ומשימות - גב`) is **not a git repo** — only the frontend is version-controlled. Backend changes are tracked only via `clasp version` history in Apps Script itself.
9. Automated/headless browser testing environments (e.g. the in-session preview browser) can suspend `requestAnimationFrame`/`ResizeObserver` callbacks until a real paint is forced (e.g. by taking a screenshot) — don't conclude a rAF/RO-based fix is broken just because it doesn't fire immediately in that environment; force a paint and recheck before deciding.
10. Synthetic `PointerEvent`s dispatched from `javascript_tool` in the preview browser only reach the app's `setupDrag` listeners *after* that same rAF-suspension (point 9) has been cleared by a forced paint — a synthetic drag test that shows no reaction isn't proof the drag code is broken, it's likely proof no paint happened yet. Screenshot first, then dispatch.
11. A full `/impeccable critique`/`audit` + motion-review pass ran this session (scoped to the board/day view, task pool, picker, drawer, and the drag/height-equalization motion) — see git log on `main` for the three resulting commits (delete/contrast/visibility fixes, touch-target/checkbox/contrast fixes, then the motion-transition pass). All findings from that pass are resolved; a fresh pass would need to re-scope to whatever's changed since.
