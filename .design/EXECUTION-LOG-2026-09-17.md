# Phase 3 execution log — לוז ומשימות remediation

Tracks execution of `PLAN-2026-09-17.md`. Newest batch on top.

---

## Batch 1 — Critical security (backend-first) — 🔄 IN PROGRESS 2026-09-17

### B1.1 — reject `auth==='plain'` on privileged mutations → ✅ DONE (backend v18, live-verified)
`backend-apps-script/קוד.js`. New `isGoogleAuth(auth)` helper; guard added at the top of `setScriptConfigForClient`, `setAccessForClient`, `removeAccessForClient`, `setDefaultRoleForClient` (before the role check) → returns `auth_required` + logs `plain_denied` for any non-Google caller. Bootstrapping unaffected (client id already set; `adminSet*` fallbacks intact).

### B1.2 — spreadsheet formula-injection guard → ✅ DONE (backend v18, live-verified)
`safeCell(v)` (apostrophe-prefix on leading `= + - @` / tab / CR) applied centrally in `appendRowByHeader` (covers Boards/Config/Sessions/Log incl. `logEvent`) + at the `renameBoardRow` `setValue`. `isValidEmail` (must start alphanumeric) rejects malformed/formula emails in `setAccess` (`bad_email`) without prefixing legit ones (matching preserved). `cleanBoardName` caps names at 80 chars, strips newlines.

**Deploy:** `clasp push` → `clasp version` (v18) → `clasp deploy` to the fixed deploymentId `AKfycbx…UgCdwVXCkUU @18`. (Backend is not a git repo; version history lives in clasp.)

**Live verification** (scratch board `board-cd800940`, plain-auth via `api.mjs`, deleted after):
- `createBoard` name `=IMAGE("http://evil/x")` → stored & returned as **inert literal text**, not an evaluated formula ⇒ B1.2 works.
- plain-auth `setDefaultRole` → `auth_required` ✓ · plain-auth `setScriptConfig` → `auth_required` ✓ · plain-auth `setAccess` → `auth_required` ✓ (surfaced as the retry helper's `gave_up` after 3 identical `auth_required` responses).
- plain-auth `saveBoard`/`createBoard` → `ok` ✓ (REQUIRE_AUTH grace period otherwise intact — nothing legitimate broke).

### B1.3 — frontend XSS in summary header → ⏳ NEXT (not started)
### B1.4 — Log audit → ⏳ (after B1.3) · ### B1.5 — flip REQUIRE_AUTH → ⏳ (gated on B1.1 live [done] + clean B1.4)

---

## Batch 0 — Pre-flight confirmation (investigation, no code) — ✅ DONE 2026-09-17

Both not-yet-reproduced top-10 findings confirmed. Two scratch boards (`board-77437584` planner "A" seeded with a person `AAAA_MARKER`; `board-f5806232` notes "B" seeded with page `BBBB_MARKER`) created via the API and **deleted afterward** (plus one retry-duplicate orphan cleaned up); final `listBoards` = `board-1` only. Live checks ran on the owner's signed-in Chrome — note her Chrome session resolves to a Google address **other than** `gaia.harlev@gmail.com` (an explicit grant to that address gave no_access; `DefaultRole=editor` on the scratch boards was used to grant her session).

### P0.1 — C2 cross-board clobber → **CONFIRMED (deterministic code trace).** Fix scope: **both** `refreshFromServer` and `pushState`.

A live black-box race could not be staged (the switch trigger `switchToBoard` is only reachable through the drawer board-list, which was empty for the test session, and `appState`/`currentBoardId` are IIFE-private so the race can't be driven or observed directly; there is no `popstate`→`switchToBoard` path). Confirmation is by an exact trace of the real code, which is conclusive for a missing-guard-around-`await`:

- `refreshFromServer` (`index.html:1516-1552`) reads `currentBoardId` into the fetch URL, `await`s, then on resolve writes `appState = serverData` (1537), `currentBoardType = json.board.type` (1530), `cacheBoardType(currentBoardId, …)` (1531), `lastKnownHash` (1525), and `localStorage.setItem(cacheKeyFor(currentBoardId), serverJson)` (1552) — all reading the **global** `currentBoardId`, none captured before the await.
- `switchToBoard` (`index.html:2170-2212`) synchronously reassigns `currentBoardId = boardId` (2182) and does **not** abort the in-flight fetch (only `clearInterval(pollTimer)` at 2180).
- **Interleaving:** poll for A in flight → user switches to B → A's response resolves against `currentBoardId === B` → B's `appState`, `currentBoardType`, and `localStorage[cacheKeyFor(B)]` are overwritten with **A's** data/type. Next edit persists A's content to board B (permanent overwrite). The observable window is real: measured Apps Script `getBoard` cold latency ~11s this session, far larger than a human switch gap.
- **`pushState` also needs the guard:** an in-flight `pushState` for A resolving after the switch writes `lastSyncedJson = payload` (A's data) and `hasPendingLocalChange = false` (`index.html:1466-1467`) into the now-B-scoped globals — can drop a real pending B edit and poison B's sync baseline. The POST body's `board` is captured before the await (1450) so the write still targets A, but the post-await global writes leak into B. → **B2.1 must add capture-and-recheck to both functions.**

### P0.2 — C7 notes drag-drop XSS → **CONFIRMED live (critical).**

- Source: `wireOutlineTextEvents` (`index.html:4922`) wires `focus`/`blur`/`paste`/`input`/`keydown` but **no `drop`/`dragover`** listener. The only sanitization of block html is `block.html = sanitizeInline(el.innerHTML)` inside the `input` handler, which runs **after** the DOM mutation.
- Live test on scratch notes board B, page open, `.ob-text` focused: `document.execCommand('insertHTML', false, '<img src=x onerror="window.__c7=…">')` — the exact DOM operation a browser's default drop performs on a contentEditable with no drop handler — **fired `onerror` immediately** (`window.__c7 === 1`), and the raw `<img onerror>` was **still in the live DOM 300ms later** (`imgStillInDom: true`), i.e. the `input`-driven sanitize did not remove it from the DOM (it only rewrites the data-model `block.html`, per the app's deliberate no-innerHTML-reassign choice). Benign flag-only payload; the owner's session/board were unaffected and the board was deleted.
- **Verdict:** untrusted HTML inserted via drop executes on insertion, before/despite the sanitizer. The planned fix (add a `drop`+`dragover`/`dragenter` `preventDefault` handler that inserts only `dataTransfer.getData('text/plain')` via `execCommand('insertText')`, mirroring the existing paste handler — **B2.8**) is validated as the correct approach. One residual: a literal cross-tab drag couldn't be staged in this environment (the DOM-insertion mechanism was reproduced by-equivalence via `execCommand('insertHTML')`); worth one manual cross-tab drag by the owner after B2.8 ships, but the source + mechanism result make the vulnerability certain.

**Batch 0 outcome:** both findings stand; no downgrades. Fix approaches for B2.1 (both functions) and B2.8 (drop handler) are confirmed correct. Ready for Batch 1 on the owner's go-ahead.
