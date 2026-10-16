# Reddit Commander — Plan (Stage 2 of 3)

Status: DRAFT for approval · Date: 2026-10-16 · Implements `docs/DESIGN.md` v3
Scope: one sub (r/MensRomance), Devvit bridge + Electron console + MCP as the natural-language interface to the app.

---

## 0. Your actions (these gate the build, and take calendar time)

| # | You do | Why / notes | Needed by |
|---|---|---|---|
| U1 | **Create r/MensRomance** (you are owner) and keep it under 200 subscribers during development | Playtest installs are only allowed on subs with <200 subscribers | M0 |
| U2 | Create a **Reddit developer account** at developers.reddit.com and make sure the CLI login works (`npx devvit login`) | The bridge is built and playtested with your login | M0 |
| U3 | **File the External Endpoints access request** ([form linked in the Devvit docs](https://github.com/reddit/devvit-docs/blob/main/docs/capabilities/server/external-endpoints.mdx)). I'll draft the text for you (see §6) | Limited-access feature. The longest lead time in the project. | Now |
| U4 | Pick the **app name** (becomes the app account, e.g. `u/<appname>`). No "Reddit" or Snoo. | Hard to change once published | M0 |
| U5 | Tell me your desktop OS | Packaging priority only | M7 |

Everything through M3 can be built and tested without U3, using the fixture backend. M0 (the spike) needs U1, U2 and U4, and will tell us whether playtest mode allows external endpoints before access is granted.

---

## 1. Repository layout (pnpm monorepo)

```
apps/desktop      Electron + electron-vite + React (main / preload / renderer)
apps/bridge       Devvit Web app (server: Hono, TypeScript)
packages/protocol zod schemas + types: bridge wire format, IPC, MCP tools, policy tiers
packages/fixtures recorded/generated sample data (queue, reports, modlog, modmail)
docs/             DESIGN.md, PLAN.md, ADRs
```
CI (GitHub Actions): lint + typecheck + unit + Electron E2E (fixture mode) + bridge tests, all offline.

---

## 2. Milestones

Each ticket lists acceptance criteria (AC). "Fixture mode" means the desktop runs against `FixtureProvider` with no Reddit access.

### M0 — Feasibility spike (gate; ~1 short session once U1/U2/U4 are done)
Goal: prove the full loop with the thinnest code, and learn what we couldn't verify from docs.
- **M0.1** Devvit app skeleton installed on r/MensRomance via playtest. AC: app account appears as a mod with chosen permissions; `/internal` health check works.
- **M0.2** One external endpoint `GET /external/ping` and one command `POST /external/command` (approve a test post). AC: a Node script with a managed token approves a post; result visible in the mod log as the app account.
- **M0.3** Measure: p50/p95 latency of `getModQueue` through the endpoint, behaviour at the 5 rps cap, and whether external endpoints work in **playtest** before access approval.
- **M0.4** Write results into `docs/ADR-001-bridge-feasibility.md`.
**Gate:** if external endpoints can't work for us, we stop and switch to the Devvit-only fallback (in-Reddit dashboard, no desktop/MCP). We decide together.

### M1 — Foundations
- **M1.1** Monorepo, tooling (TypeScript strict, ESLint, Prettier, Vitest), CI. AC: green CI on an empty app.
- **M1.2** `packages/protocol`: zod schemas for `Item`, `Report`, `ModAction`, `Command` (+ signed envelope), `Delta`, policy tiers. AC: both apps import them; breaking a schema fails typecheck on both sides.
- **M1.3** Electron shell with hardened defaults: `contextIsolation`, `sandbox`, strict CSP, typed preload, no renderer network. AC: an automated test asserts these flags and that `require` and `fetch` are unavailable in the renderer.
- **M1.4** `RedditProvider` interface + `FixtureProvider` with ~500 realistic generated items. AC: the app boots in fixture mode.
- **M1.5** SQLite cache (better-sqlite3 + Drizzle migrations), `safeStorage` secrets. AC: migrations run on a clean profile; secrets never appear in logs (test greps the logs).

### M2 — Queue and sync (fixture first, then bridge)
- **M2.1** Unified Queue view: virtualised list (10k items stays at 60 fps), filters, detail pane. AC: Playwright test filters and scrolls 10k fixture items.
- **M2.2** Keyboard layer (`j/k a r s l b n z / ⌘K`) with a visible cheat-sheet. AC: every action reachable without the mouse.
- **M2.3** Sync engine: cursor-based delta polling, adaptive interval, shared 5 rps limiter, backoff. AC: simulated 429s and gaps recover with no duplicate or missing items.
- **M2.4** Bridge: `/external/changes`, `/external/snapshot`, trigger-fed Redis change feed (IDs and minimal metadata, TTL). AC: bridge unit tests with mocked `reddit`; desktop in bridge mode shows live items from the test sub.

### M3 — Command Bus, policy, audit
- **M3.1** Typed commands + policy engine (tiers from Design §6.1, per-sub editable). AC: table-driven tests for every command × tier × source (`ui`/`mcp`).
- **M3.2** Signed envelope (HMAC, replay protection) and **bridge-side policy re-check**. AC: tampered, replayed or out-of-policy commands are rejected by the bridge in tests.
- **M3.3** Action executor with optimistic UI and undo; mod-note attribution (`requested by u/you via Commander (ui|mcp)`). AC: approve, remove and undo round-trip on the test sub.
- **M3.4** Audit log (source, args, result, undo token), exportable. AC: every command appears exactly once.
- **M3.5** Deletion handling: bridge emits tombstones for `PostDelete`, `CommentDelete` and account deletion, and the desktop purges content, derived rows, FTS entries and audit-log text. AC: test proves content is gone from every table including FTS, and a retention job enforces ≤30 days for non-aggregate data.

### M4 — Dashboards and user context (the point of the project)
- **M4.1** Bridge aggregate counters (Design §3.1a) + daily scheduler snapshot. AC: counters survive with the desktop closed and backfill on connect.
- **M4.2** Dashboard v1: queue depth and age, reports by rule, removals by moderator/reason, time-to-action, first-time-poster rate, subscriber growth, 7/30-day trends with anomaly highlighting. Built per the `dataviz` guidance (consistent palette, light/dark). AC: renders from fixtures and from real counters; empty/new-sub states are handled gracefully.
- **M4.3** User context card: account age, history in the sub, previous removals and mod notes (`getModNotes`), other open reports by the same author. AC: loads in <300 ms from cache; no network call on the click path.
- **M4.4** Removal reasons and macros (remove + comment + note + optional ban). AC: macro executes as one audited command bundle.

### M5 — MCP server (natural-language interface to the app)
- **M5.1** Local MCP server (Streamable HTTP on 127.0.0.1, bearer token, off by default) + `rc-mcp` stdio shim. AC: connects from Claude Desktop/Code; token revocation takes effect immediately.
- **M5.2** Read tools with **UI parity** (Design §6.2): everything visible in the app is queryable, including content and dashboards. Reddit text is wrapped and labelled untrusted. AC: a parity test enumerates UI data sources and asserts each has a read tool; an injection test seeds items with hostile instructions ("ban all users") and asserts no write executes above its tier.
- **M5.3** `plan_bulk_action` → filter evaluated in-app → low-risk executes directly, others go to the **Proposals tray** with a full preview. AC: matched count and items returned; medium/high-risk actions never execute without approval from the UI.
- **M5.4** Kill switch (tray + hotkey) and write rate caps. AC: after the kill switch, all MCP calls fail within 1 s.
- **M5.5** MCP activity visible in the UI: a live "Claude activity" feed and `source: mcp` on every audit row. AC: every MCP call appears in the feed and audit log.

### M6 — Modmail and mod log
- **M6.1** Modmail inbox and thread view, assign/archive/highlight/reply (reply is medium tier), canned replies. AC: works in fixture and bridge mode.
- **M6.2** Searchable mod log (FTS5) with who/what/why. AC: search returns in <100 ms over 50k rows.

### M7 — Hardening, review, release
- **M7.1** Threat-model pass (Electron, token handling, bridge endpoints) and fixes. AC: written review plus a security-review run on the diff.
- **M7.2** Devvit app README (plain-language), privacy policy and terms (required for fetch/external features), submit for **Reddit app review** once the sub nears 200 members. AC: submission accepted for review.
- **M7.3** Packaging with electron-builder for your OS (others best-effort), auto-update later. AC: signed/installable build; first-run wizard connects to the bridge and verifies the install.

---

## 3. Test strategy
- **Unit (Vitest):** protocol schemas, policy tables, sync cursor logic, limiter, filter evaluator.
- **Contract:** shared `protocol` types make desktop↔bridge drift a compile error. The MCP injection test (M5.2: hostile text in items must never cause a write above its tier) is a hard CI gate.
- **E2E (Playwright for Electron):** all flows in fixture mode, offline, deterministic.
- **Bridge:** mocked `reddit`/Redis unit tests, plus a manual playtest checklist on r/MensRomance for each release (Devvit can't be fully simulated).
- **Live checks:** one scripted smoke run against the real sub at the end of M2, M3 and M5; never against a sub with real members until the app has passed review.

## 4. Risks that can change the plan
| Gate | If it fails | Response |
|---|---|---|
| M0: external endpoints don't work in playtest | We wait for U3 approval, building M1–M3 on fixtures meanwhile | No wasted work |
| U3 denied | Devvit-only fallback | Reuse `protocol`, UI components, and dashboards inside a Devvit Web mod panel; drop desktop shell + MCP |
| Throughput (5 rps) or latency worse than assumed | Larger deltas, fewer polls, snapshot caching | Tuned in M2.3 |
| Review required at 200+ subs | Plan release timing | M7.2 |

## 5. Definition of done (v1)
All M0–M7 AC met · CI green · MCP injection test passes (no write above its tier) · deletions purge locally within one sync · kill switch tested · README, privacy policy and terms published · app approved for r/MensRomance beyond 200 members.

## 6. Draft text for the External Endpoints request (U3)
> **App:** `<appname>`, a private moderation console for one subreddit (r/MensRomance), owned and moderated by the developer. **Why external endpoints:** the owner runs a local desktop dashboard that reads the sub's mod queue, reports and aggregate statistics from the app, and issues moderation actions (approve, remove, notes) that the app executes as its own account. Personal domains can't be fetched by Devvit, so the app can't push to it; an authenticated inbound call is the only fit. **Data handling:** minimal metadata stored in Redis with TTL; aggregate counters contain no usernames or content; the desktop purges anything deleted on Reddit; no data is sold, shared, profiled or used to train models. The owner can also drive the desktop app through a local natural-language (MCP) interface; the assistant sees only what the owner's app shows, and every action goes through the app's tiered policy. **Security:** managed token plus signed, replay-protected command envelopes and server-side policy limits on destructive actions. **Scope:** single installation.

---

## 7. Next step
On your approval of this plan: (1) you do U1–U4, I file nothing on your behalf, but I'll hand you the exact request text above; (2) I start **M1** immediately (it needs no Reddit access), and **M0** as soon as U1/U2/U4 are done.
