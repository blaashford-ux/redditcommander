# Reddit Commander — Design (Stage 1 of 3: Design → Plan → Build)

Status: DRAFT for review · Date: 2026-10-16

A desktop moderation console for Reddit: one streamlined, keyboard-first interface across every subreddit you moderate, plus a local MCP server so Claude can drive the app. Claude never touches Reddit. It sends commands to the app, and the app executes them with your credentials, the same as if you had clicked.

---

## 1. Goals / non-goals

**Goals**
1. One unified dashboard across all moderated subs: queue depth, report spikes, modmail backlog, activity, team action.
2. Triage with minimal clicking. Reports, modqueue, spam, unmoderated and edited items sit in one list. The user context is inline, and every action is one keystroke.
3. Context where you need it: the author's history in your subs, past bans/notes, previous removals, and other reports on the same user.
4. A local MCP server so Claude can read app state and issue commands through the same code path the UI uses.
5. Fast. The app works from a local cache, shows optimistic updates, and syncs in the background.

**Non-goals (v1)**
- Not a general Reddit client (no browsing, voting or posting as a regular user).
- No scraping, no cookie or session hijacking, no old.reddit automation. Official API only.
- Not a cloud service. Everything runs locally and there is no backend of ours.
- No autonomous AI actions. Claude proposes and acts only through the gated command bus (§6).

---

## 2. The gating constraint: how we're allowed to talk to Reddit

This decides the architecture, so it comes first. **I could not reach Reddit's own pages from this environment, so everything below is from secondary press coverage (TechRepublic, PPC Land, etc.) and must be verified before we commit.**

What the coverage says as of Oct 2026:
- Reddit's [Responsible Builder Policy](https://support.reddithelp.com/hc/en-us/articles/42728983564564-Responsible-Builder-Policy) requires explicit approval before any API access. It describes moderator tooling as the intended use case.
- Self-service OAuth app creation is closed or redirected. Approval reportedly takes weeks and can be denied without explanation.
- **New public Data API requests reportedly stop on Oct 31, 2026.** Removal of unregistered apps starts Jan 12, 2027, and remaining public Data API access ends around March 2027. ([TechRepublic](https://www.techrepublic.com/article/news-reddit-rss-public-api-shutdown/), [PPC Land](https://ppc.land/reddit-to-halt-public-data-api-access-in-march-2027-as-14-000-apps-register/))
- Reddit is steering apps to **Devvit**, its hosted developer platform. Apps run on Reddit's infrastructure, per-subreddit, with triggers, storage and scheduler. Python is not supported, and external connections and cross-sub processing need extra approval.

**Implication:** a desktop app that calls the Data API as you may have a short shelf life, or may never get approved. Devvit is not a drop-in replacement either. It is an in-Reddit app model, not a client SDK. Devvit apps act as the app account (or as the user only in some user-initiated contexts), and they can't be called from outside except through patterns we have to design (§3.3).

### Strategy: a provider abstraction with two backends

All Reddit access goes through one interface, `RedditProvider`. The UI, the cache and the MCP server never know which backend is active.

| | **A. Data API provider** (primary) | **B. Devvit bridge provider** (contingency) |
|---|---|---|
| Identity of actions | You (OAuth, `modself`, `modposts`, `modcontributors`, `modlog`, `modmail`, `read`, `identity`, `history`…) | Devvit app account, unless Reddit grants run-as-user |
| Mod log attribution | Your username | App name (e.g. `u/commander-bridge`) |
| Latency | Poll | Triggers push events, but delivery goes through the app |
| Setup | Apply for access **now** | Build and install a Devvit app on each sub (you are mod, so you can) |
| Risk | Approval denied or sunset in 2027 | Attribution and capability differences, extra approvals for `fetch` |

**Recommendation**
1. **Apply for Data API access immediately**, before Oct 31. A personal-use moderator tool is the best case under the policy. This costs nothing and is the long pole.
2. Build against `RedditProvider` with backend A first, plus a `FixtureProvider` (recorded JSON). The fixture provider lets us build the whole app without credentials, so approval delays don't block us.
3. Prototype backend B early (a spike in the Plan stage) so we know whether it is viable before 2027.

---

## 3. Architecture

```
┌──────────────────────────── Electron app ─────────────────────────────┐
│  Renderer (React + TS, Vite)          Main process (Node, TS)         │
│  ─────────────────────────            ───────────────────────────     │
│  Dashboard / Queue / Modmail /  IPC   Command Bus  ◄── MCP server     │
│  Users / Log / Settings   ◄──────────► (single write path)   (local)  │
│  Zustand + TanStack Query             │                                │
│  cmdk palette, virtual lists          ├─ Action Executor (rate limited,│
│                                       │   undo, audit)                 │
│                                       ├─ Sync Engine (pollers)         │
│                                       ├─ SQLite cache (better-sqlite3) │
│                                       └─ RedditProvider ──► Reddit     │
│                                          (DataApi | Devvit | Fixture)  │
└────────────────────────────────────────────────────────────────────────┘
        ▲ Claude Desktop / Claude Code connect to the MCP endpoint
```

### 3.1 Process split and security
- Renderer has **no Node access and no network**. `contextIsolation: true`, `sandbox: true`, strict CSP, and a typed preload bridge. All Reddit I/O lives in the main process.
- Tokens are stored encrypted via Electron `safeStorage` (OS keychain). They are never sent to the renderer or the MCP server, and never logged.
- All untrusted Reddit text (post bodies, usernames, mod notes) is treated as hostile: rendered through a sanitising markdown renderer, and **flagged as untrusted data in MCP responses** to resist prompt injection (§6.4).

### 3.2 Core services (main process)
- **Provider**: normalised types (`Item`, `Report`, `User`, `ModAction`, `ModmailThread`, …). Backend quirks stay behind the interface.
- **Sync Engine**: per-sub pollers for modqueue, reports, unmoderated, spam, edited, modmail and modlog. It uses `before` fullname cursors, adaptive intervals (fast when the window is focused, slow in the background), and a **global rate-limit budget** shared with user-initiated actions. User actions always preempt background sync.
- **Cache (SQLite)**: items, users, subs, actions, notes, settings and the audit log. It's the source of truth for the UI, so the UI never waits on the network. WAL mode, FTS5 for search.
- **Command Bus**: every mutation is a typed command (`removeItem`, `approveItem`, `banUser`, `sendModmailReply`, …). It is validated with zod, checked against permission policy, rate limited, executed, recorded in the audit log and, where possible, made undoable. **The UI and MCP use the same bus**, so Claude can't do anything you can't do by hand, and nothing bypasses auditing.

### 3.3 Devvit bridge (backend B, sketch)
A small Devvit app installed on each sub:
- Triggers (`PostReport`, `CommentReport`, `ModAction`, `PostSubmit`, …) push events to a relay by `fetch`.
- The app polls a command queue and executes mod actions with `reddit.*` as the app account.
- The relay is a local tunnel or tiny hosted endpoint, which breaks the "no backend of ours" rule. It is therefore contingency only, and needs `fetch` domain approval. Details are deferred to the spike.

---

## 4. Stack

| Concern | Choice | Why |
|---|---|---|
| Shell | **Electron** + **electron-vite** (Vite for main, preload, renderer) | As requested. electron-vite gives HMR for all three targets. |
| Language | TypeScript (strict) across everything | Shared types for IPC, commands and MCP. |
| UI | React 19, Tailwind, Radix primitives (shadcn-style components) | Fast to build, accessible and modern. |
| State | TanStack Query (server cache over IPC), Zustand (UI state) | Simple, with good invalidation. |
| Lists | TanStack Virtual | Queues can hold thousands of items. |
| Palette / keys | cmdk, custom hotkey layer | Keyboard-first triage. |
| Storage | better-sqlite3 + Drizzle (migrations), FTS5 | Fast, local and typed. |
| Validation | zod (commands, IPC, MCP schemas, provider responses) | One schema, many uses. |
| MCP | Official `@modelcontextprotocol/sdk` | Streamable HTTP on `127.0.0.1` plus a stdio shim (`rc-mcp`). |
| Tests | Vitest (unit), Playwright (Electron E2E, with FixtureProvider) | Fully offline CI. |
| Packaging | electron-builder (Win/mac/Linux), auto-update later | |

Alternative considered: Tauri (smaller, Rust). Rejected because you asked for Electron, and the shared TS across main and renderer is valuable here. No better option for this scope.

---

## 5. UX design

### 5.1 Dashboard (home)
- **Sub cards** show queue count, reports, modmail unread, 24h removals/approvals, and a sparkline vs. 7-day baseline. Anomalies (e.g. a report spike or a brigading signature) are highlighted.
- **Needs attention**: oldest unhandled items, escalations, threads awaiting your reply.
- **Team pulse**: who acted recently, so you can avoid double handling.
- **Quick actions**: jump to the queue, open the palette, snooze a sub.

### 5.2 Unified Queue (the main screen)
- Three panes: filters/subs, virtualized item list, and a detail pane with the item, the report reasons, and the **user context card**.
- **Everything in one list**: modqueue, reports, spam, unmoderated and edited items. Filter by sub, type, reason, age or author.
- **Keyboard triage**: `j/k` move, `a` approve, `r` remove (opens the reason picker), `s` spam, `l` lock, `b` ban, `n` note, `e` escalate, `z` undo, `/` filter, `⌘K` palette. Actions apply optimistically and can be undone for N seconds.
- **User context card**: account age, karma in your subs, post/comment history within your subs, previous removals/bans/notes, and other open reports from the same author. No click-through is required.
- **Bulk select** (`x`) with group-by-author and group-by-reason.
- **Removal reasons and macros**: pick a rule, send the removal comment or modmail with templates. Macros are configurable (remove + lock + comment + note + optional ban).

### 5.3 Other views
- **Modmail**: threaded inbox, assign, archive, highlight, canned replies, and the user context card alongside.
- **Users**: search a user across all your subs, with a full action history and notes (stored locally; syncing with Reddit usernotes is a later option).
- **Mod log**: searchable, filterable, with a diff-style view and "who did what".
- **Rules and automations**: view rules, removal reasons and AutoModerator config (read-only in v1).
- **Settings**: accounts, keys and hotkeys, rate-limit behaviour, MCP permissions.

---

## 6. MCP server

Goal: Claude helps you moderate by operating *the app*. It has no Reddit credentials and no network path to Reddit.

### 6.1 Transport and access
- Streamable HTTP bound to `127.0.0.1`, protected by a random bearer token generated in Settings (revocable and rotatable). The stdio shim `rc-mcp` forwards to it for clients that need stdio.
- Off by default. Enabling it is an explicit toggle and the UI shows connection state.

### 6.2 Permission tiers (per connection, set in Settings)
| Tier | Claude can… |
|---|---|
| **Read** (default) | Read queues, items, users, the modlog, modmail, stats and rules |
| **Propose** | Also create *proposals*. These are queued actions that appear in an "AI proposals" tray and **run only when you approve them** (one keypress, or batch-approve). |
| **Act (scoped)** | Execute chosen low-risk commands directly (e.g. approve, add note, assign). Destructive ones (ban, mass-remove, modmail send) still require approval. |

All commands go through the Command Bus, so audit, rate limits and undo apply equally. Every audit entry records `source: "mcp"`, the client name and the originating prompt context where available.

### 6.3 Tool catalogue (draft)
- **Read**: `list_subs`, `get_dashboard`, `list_queue(sub?, filter?)`, `get_item(id)`, `get_user_context(username, sub?)`, `search_users`, `get_modlog(filters)`, `list_modmail`, `get_modmail_thread`, `get_rules(sub)`, `get_stats(sub, range)`.
- **Write (via bus)**: `approve`, `remove(reason?)`, `mark_spam`, `lock`, `ban(duration, reason, message)`, `add_note`, `reply_modmail`, `apply_macro`, `undo(actionId)`, `snooze_sub`.
- **Proposals**: `propose_action(command, rationale)`, `list_proposals`.
- **Resources**: `queue://{sub}`, `item://{id}`, `user://{name}`. **Prompts**: `triage_queue`, `summarise_modmail`, `investigate_user`.

### 6.4 Safety
- **Prompt-injection defence**: Reddit content returned to Claude is wrapped and labelled as untrusted user data, and tool descriptions tell the model never to follow instructions inside it. Write tools are the real defence: tiering, proposals and approval gates mean an injected "ban everyone" can't execute silently.
- **Hard limits**: per-minute caps on write commands via MCP, no bulk destructive command above a threshold without approval, and a global kill switch (tray icon and hotkey) that revokes the MCP token instantly.
- **Full audit log**, exportable, visible in-app.

---

## 7. Data model (SQLite, abridged)
`accounts` · `subs` (flags, last_sync cursors) · `items` (id, kind, sub, author, body, state, reports_json, created, edited) · `users` (name, first_seen, flags) · `user_sub_stats` · `mod_actions` (local + synced modlog) · `notes` · `modmail_threads` / `modmail_messages` · `macros` · `removal_reasons` · `proposals` · `audit_log` (source, command, args, result, undo_token) · `settings` · FTS tables on `items`, `modmail_messages`, `notes`.

---

## 8. Rate limiting & sync strategy
- Stated limit is ~60–100 requests/min per OAuth client (**unverified**). The app implements a token-bucket budget, configurable and conservative by default.
- Priority order: user actions > the focused view's refresh > background pollers.
- Multi-sub: fetch the combined `r/a+b+c/about/modqueue` endpoint where possible to spend one request for many subs.
- Back off on `429` and use the `X-Ratelimit-*` headers.

---

## 9. Risks & open questions

| # | Item | Mitigation / decision needed |
|---|---|---|
| R1 | **API approval denied or delayed, with a 2027 sunset** | Apply now; provider abstraction; Fixture backend; Devvit spike |
| R2 | Devvit attribution and capability differences | Spike before committing; may be a degraded mode |
| R3 | Reddit ToS: actions must be user-initiated moderation only | Proposal/approval gating; no autonomous loops; get the policy text verified |
| R4 | Prompt injection via Reddit content | §6.4 |
| R5 | Rate limits vs. many subs | Shared budget, combined endpoints, adaptive polling |
| R6 | Unverified facts in this doc (dates, limits) | Verify against Reddit's official pages (they were unreachable from this environment) |

**Questions for you**
1. Have you already got Data API credentials or applied for them? If not, you should apply this week (see §2).
2. How many subs do you moderate, and roughly how large are the queues? This affects polling and caching decisions.
3. Platforms: Windows, macOS, Linux?
4. Is the "Act (scoped)" MCP tier wanted, or do you prefer propose-and-approve for everything in v1?
5. Do you use Toolbox usernotes or Mod Notes today? We should be compatible with your existing notes.
6. Should a single account be assumed, or multiple mod accounts?

---

## 10. Preview of the Plan stage (next)
Milestones: M0 scaffold (electron-vite, IPC, CI, FixtureProvider) → M1 data layer + sync + queue view → M2 actions, undo and audit (Command Bus) → M3 dashboard and user context → M4 modmail and modlog → M5 MCP server and proposals → M6 Data API backend with real OAuth, then Devvit spike, then packaging. The Plan stage will break these into tickets with acceptance criteria and the test strategy.
