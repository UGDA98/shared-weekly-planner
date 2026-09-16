# Storage recommendation — לוז ומשימות (2026-09-16)

## 1. Executive recommendation

**Stay on Google Sheets + Apps Script. Do not plan a migration. Do plan hardening — and do it before the notes board type ships, not after.** The storage ceiling is not the problem: at the measured 1.9k chars/week the planner board reaches the 540k chunk ceiling in roughly **273 weeks (~5.2 years)**, and even a 200k-char notes board has multi-year headroom. The problem is **read amplification**, and it arrives at notes launch, not in 2031. Two compounding issues: (i) the 20s full-blob poll (Watch Out #14, threshold ~150KB) — a 200k-char notes board with 10 open tabs is ~2.9 GB/day of egress through one Apps Script quota bucket; (ii) a bug-grade inefficiency in the current code — `getBoardState`, `getBoardsTable` and `saveBoardStateRaw` all call `sheet.getDataRange().getValues()`, so **every poll and every `listBoards` deserializes every board's full state**, not just the requested one. With 5 notes boards at 200k chars each, a single `getBoard` for a 1k-char board still pulls ~1M characters off the sheet. Fixing that plus a hash-based `ifHash` poll costs ~2–4 person-days, keeps the inspectable Sheet the org values, keeps the Apps Script access model the new sign-in work is being built on, and stretches the runway past every alternative's migration payback period. **This flips if** a single board exceeds ~300k chars (half the chunk ceiling, and past the point where hand-repair in a Sheet is realistic), or if notes boards routinely exceed ~250k, or if `locked`/interstitial errors become routine rather than transient. The escape hatch at that point is **(b) Drive JSON files, one per notes page**, not Firestore — it keeps Apps Script, the Sheet metadata, the access model and the exit path, and is a 3–5 day change rather than a rewrite.

## 2. Comparison

| Candidate | Free-tier ceiling that binds here | Runway (planner + notes) | Migration effort | Ops burden | Inspectability | Real-time | Exit path |
|---|---|---|---|---|---|---|---|
| **(a) Sheets + hardening** | 50k chars/cell → 540k/board (12 chunks); 10M cells/spreadsheet; 30 simultaneous executions per deploying user; 20k UrlFetch/day (consumer) | Planner ~5 yrs; notes fine to ~500k/board. Poll cost solved by `ifHash` | **2–4 d** (incremental, no data move) | None new | Full — org opens the Sheet | 20s poll (unchanged) | Already just JSON in a cell |
| **(b) Apps Script + Drive JSON** | 15 GB shared Drive pool; ~50 MB/file via DriveApp; UrlFetch/day unchanged | Effectively unbounded for this workload | **3–5 d** | Low (files in a Drive folder) | Good — readable JSON files, Sheet keeps metadata | 20s poll | Files are plain JSON; trivial |
| **(c) Firestore (Spark)** | 1 MiB/document; 50k reads, 20k writes, 20k deletes/day; 1 GiB stored | Doc limit forces per-page split anyway; 50k reads/day is ~tight with listeners × staff | **6–12 d** + relearning access control | Medium — security rules are a new failure surface | **Lost** — no Sheet; console only | Yes, real listeners (kills the poll) | Export tooling exists, but rules+auth are rewritten |
| **(d) Supabase free** | **Projects pause after ~1 week of inactivity**; 500 MB DB; 2 projects | Disqualified — a staff tool used in bursts will find it paused | **8–15 d** | High (unpause, no backups on free) | Medium (SQL editor) | Yes (Realtime) | Plain Postgres — best exit of the lot |
| **(e) CF Workers + KV/D1** | Workers 100k req/day, 10 ms CPU; **KV: 1,000 writes/day free**, 25 MiB value; D1: 500 MB DB, 10 DBs | KV's 1,000 writes/day is below a normal editing day; D1 workable | **8–15 d** (new auth, new deploy, new repo) | Medium — a second vendor to maintain | Poor (wrangler CLI) | Polling still, unless Durable Objects | Good (SQLite dump) |
| PropertiesService / CacheService | 9 KB per value / 500 KB per store; Cache 100 KB per value, 6 h max TTL | — | — | — | — | — | Not a store. Ruled out in one line: too small and, for Cache, non-durable. |

## 3. Per-candidate detail

### (a) Sheets + targeted fixes — recommended

Relevant hard numbers: a cell holds **50,000 characters**; a spreadsheet holds **10 million cells** total across tabs ([Sheets limits](https://support.google.com/docs/thread/11221515/google-sheets-scripting-workaround-for-50-000-character-cell-limit?hl=en), [Zapier](https://zapier.com/blog/google-sheets-cell-limit/)) — the 8k-row `Log` tab is ~50k cells, a non-issue. Apps Script: **6 min per execution**, **30 simultaneous executions per user**, **20,000 UrlFetch calls/day** on a consumer account (100,000 on Workspace), Properties 9 KB/value and 500 KB/store, Cache 100 KB/value ([Apps Script quotas](https://developers.google.com/apps-script/guides/services/quotas), [CacheService limits](https://justin.poehnelt.com/posts/exploring-apps-script-cacheservice-limits/)).

Three quota facts that matter for the changes being built:

1. **`executeAs: USER_DEPLOYING` funnels every request through one account**, so the *30 simultaneous executions per user* cap is the real concurrency ceiling — and `LockService` already serializes saves behind it. ~20 tabs polling every 20s is ~1 request/second average; fine, but bursts plus the lock is exactly the shape that produces the HTML interstitial in Watch Out #18. `ifHash` cuts poll work by ~99%, which is the best available mitigation.
2. **Never call the tokeninfo endpoint on the poll path.** 20 tabs × 3 polls/min × 8 h ≈ **28,800 requests/day**, above the 20,000 UrlFetch/day consumer quota. Verify the Google ID token **only at login** (then the 90-day session token carries it) and UrlFetch usage stays in the low hundreds per year. This is a design constraint on change (1), worth writing into the code comments.
3. **The `Log` tab appends on every save.** At 600ms debounce, a busy editing session generates many saves; each one is now a locked read-modify-write *plus* an append. Batch or sample the save-log entries, or the audit trail becomes the dominant write cost.

Runway arithmetic: planner at 21,342 chars + 1.9k/week → 150KB poll threshold in ~68 weeks (early 2028); 540k ceiling in ~273 weeks. Notes at 25k–200k chars **start past the poll threshold**, which is why hardening is the prerequisite, not a follow-up. At 5× usage (5 notes boards, 100 tabs) `ifHash` + per-row reads keep per-request work roughly constant; at 20× the binding limit is the 30-simultaneous-execution cap, and the answer then is a longer poll interval (60s) on notes boards, which are reference material and do not need 20s freshness.

### (b) Apps Script + Drive JSON

`DriveApp` handles files to roughly **50 MB**; free Google accounts share a **15 GB** pool across Gmail/Drive/Photos ([Drive limits](https://developers.google.com/workspace/drive/api/guides/limits)). One JSON file per notes page (or per board) with the Sheet keeping `Boards`/`Config`/`Log` removes the 45k-chunk machinery entirely and makes per-page saves possible — a single edited block stops rewriting 200k chars. Costs: Drive read latency is higher and more variable than a `getRange` on an already-loaded sheet, and the data stops being visible in the spreadsheet the org values (though a JSON file in a Drive folder is still openable and hand-editable, which preserves the "maintainer can repair it" constraint). **This is the recommended next step if, and only if, a board crosses ~300k chars.**

### (c) Firestore (Spark)

Document limit **1 MiB**; free tier **50,000 reads / 20,000 writes / 20,000 deletes per day, 1 GiB stored, 10 GiB/month egress** ([Firestore quotas](https://firebase.google.com/docs/firestore/quotas)). The compat SDK still ships as plain script tags — `https://www.gstatic.com/firebasejs/12.19.0/firebase-app-compat.js` and `firebase-firestore-compat.js` ([alt setup](https://firebase.google.com/docs/web/alt-setup)) — so the no-build-step constraint survives. Real listeners would delete the poll problem outright, and security rules keyed on Google sign-in are the idiomatic fit for the identity work already underway. Against it: the 1 MiB doc limit forces the same per-page split as (b) but with a full rewrite attached; 50k reads/day is comfortable *only* if listeners are scoped tightly (a listener on a collection bills per document delivered); the Sheet disappears, which is a stated org value and the maintainer's repair surface; and the Apps Script `Config`-tab access model is replaced by rules the single maintainer must debug without a staging environment. 6–12 days and a new class of silent-failure mode, to solve a problem `ifHash` solves in two.

### (d) Supabase free

**Free projects pause after ~1 week of inactivity**, 500 MB DB, 2 active projects, no backups ([Supabase pricing](https://supabase.com/pricing), [2026 limits roundup](https://automationatlas.io/answers/supabase-free-tier-limits-2026/)). For education staff whose usage clusters around the training calendar, a paused project during a quiet fortnight means the planner is simply down, with no ops staff to restore it. That alone disqualifies it regardless of the otherwise-good Postgres exit path and CDN-loadable `@supabase/supabase-js`.

### (e) Cloudflare Workers + KV/D1

Workers free: **100,000 requests/day, 10 ms CPU/request** ([Workers limits](https://developers.cloudflare.com/workers/platform/limits/)). KV free: **100,000 reads/day but only 1,000 writes/day**, 25 MiB values, one write/second per key ([KV limits](https://developers.cloudflare.com/kv/platform/limits/)) — 1,000 writes/day is below a normal debounced editing day, so KV is out. D1 free: **500 MB per DB, 10 DBs, 5 GB per account**, 2 MB max row ([D1 limits](https://developers.cloudflare.com/d1/platform/limits/)) — workable. Google ID token verification is straightforward (fetch `https://www.googleapis.com/oauth2/v3/certs` and verify RS256 with WebCrypto, or call tokeninfo). But this adds a second vendor, a `wrangler` deploy path, and loses the Sheet; 100k req/day also caps the 20s poll at roughly 70 concurrently-open tabs.

## 4. Stay-on-Sheets hardening, ordered by value / effort

1. **Read only the target board's row.** Replace `getDataRange().getValues()` in `getBoardState` with a row lookup on the `BoardId` column, then read only that row's `State*` cells. Largest single win; ~0.5 d.
2. **Stop `listBoards` reading state at all.** `getBoardsTable` pulls every chunk column; restrict `readTableByHeader` to the columns actually requested. ~0.5 d.
3. **`getBoardMeta` + `ifHash` poll.** Store an MD5 of the JSON in a new `StateHash` column on save; the poll sends its hash and gets `{ok:true, unchanged:true}` (~40 bytes) when it matches. ~1 d, cuts poll egress ~99%.
4. **Verify the Google ID token at login only.** Never on the poll path — 28,800 polls/day would blow the 20,000/day UrlFetch quota. Design constraint, not extra work.
5. **Batch or sample `Log` appends for `saveBoard`.** Every 600ms-debounced save currently means an extra locked append; log logins/creates/renames/deletes fully, saves coarsely. ~0.5 d.
6. **Longer poll interval for notes boards (60s).** Reference material does not need 20s freshness; one constant, gated on board type. ~0.2 d.
7. **Emit board size in `getBoardMeta` and surface it in the admin menu.** Makes the triggers below observable instead of needing a manual measurement session. ~0.5 d.
8. **Only then, if needed:** per-page rows for notes boards, or archiving weeks older than 12 months to an `Archive` tab. Both change read semantics (see the Decisions row rejecting archiving) — defer until a threshold below actually fires.

## 5. Re-evaluate when…

- **Any single board's blob exceeds 300,000 chars** (55% of the chunk ceiling) → move that board type to Drive JSON (option b).
- **A notes board exceeds 250,000 chars**, or the notes board count exceeds ~8 → per-page storage becomes mandatory regardless of backend.
- **Steady-state open tabs exceed ~40**, or `locked` / HTML-interstitial responses stop being transient (more than ~1% of requests over a day) → the 30-simultaneous-executions ceiling is the binding constraint; raise the poll interval first, then reconsider (c).
- **Daily save volume exceeds ~5,000 POSTs** → the LockService serialization becomes the bottleneck; per-page writes (b) or a real DB (c) become justified.
- **Total spreadsheet approaches ~2M cells** (the `Log` tab plus growth) → split the Log into its own spreadsheet; not a migration trigger on its own.
- **The project ever gains a second maintainer or a budget** → the Firestore calculus changes, because the two real objections to (c) are inspectability-for-one-person and rewrite cost, not the technology.
