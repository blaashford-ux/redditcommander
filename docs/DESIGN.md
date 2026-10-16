# Reddit Commander — Design v2 (Stage 1 of 3: Design → Plan → Build)

Status: DRAFT for review · Date: 2026-10-16 · Supersedes v1 (Data API route dropped)

A desktop moderation console for Reddit: one streamlined, keyboard-first interface for the subs you moderate, plus a local MCP server so Claude can drive the app. Claude never touches Reddit. It sends commands to the app, and the app executes them through your Reddit-approved Devvit app.

**Decisions so far (yours):** no Data API route · Devvit first · primary target is a **new sub you're building**, plus a backup-mod role on one sub and co-mod on a small one · Claude may **act directly on low-risk actions**.

---

## 1. Goals / non-goals

**Goals**
1. One dashboard across your subs: queue depth, report spikes, modmail backlog, team activity.
2. Triage with minimal clicking. Reports, modqueue, spam, unmoderated and edited items sit in one list. The author's context is inline, and every action is one keystroke.
3. A local MCP server so Claude can read app state and issue commands through the same code path the UI uses.
4. Fast, with a local cache and optimistic updates.

**Non-goals (v1)**
- Not a general Reddit client (no browsing, voting or posting).
- No scraping or session or cookie reuse. Official platform only.
- No backend of ours (see §2.3: the only server-side code is the Devvit app, which Reddit hosts).
- Claude never gets Reddit credentials, and cannot act outside the gated command bus (§6).

---

## 2. Platform verdict: what Devvit can and can't do

Researched from Reddit's official [devvit-docs](https://github.com/reddit/devvit-docs) repo (read directly, not press coverage). The public Data API timeline from v1 remains unverified press coverage and is no longer relevant to this design.

### 2.1 What Devvit can do (for a mod tool)
- **The full moderator read/write surface** in the `reddit` client: `getModQueue`, `getReports`, `getSpam`, `getUnmoderated`, `getEdited`, `getModerationLog`, `approve`, `remove`, `banUser`/`unbanUser`, `muteUser`, mod notes (`addModNote`/`getModNotes`), removal reasons, rules, flair, a **ModMail service**, user lookups and a user's posts/comments. ([API reference](https://github.com/reddit/devvit-docs/tree/main/docs/api/redditapi))
- **Triggers** (`PostReport`, `CommentReport`, `PostSubmit`, `ModAction`, `AutomoderatorFilterPost`, …), a **scheduler**, **Redis** storage, and settings/secrets. That is enough to run autonomous rules on Reddit's side too (e.g. auto-approve trusted users in your new sub).
- **No API keys or OAuth app.** Authentication is handled by the platform.
- **Unlisted apps** are the default. Your app is installable only on subs you choose, and approval for private use is much lighter than a public listing. Playtest installs are allowed on subs with **<200 subscribers**, which fits a brand-new sub, and a reviewed app is needed beyond that. ([launch guide](https://github.com/reddit/devvit-docs/blob/main/docs/guides/launch/launch-guide.md))

### 2.2 What Devvit can't do (the dead ends)
| Idea | Verdict | Source |
|---|---|---|
| Desktop app calls Reddit **as you** | **Dead end.** There is no user OAuth token for outside callers. | `reddit-api.mdx` |
| Mod actions **attributed to you** via Devvit | **Dead end.** Actions run as the **app account**. "Run as user" exists only for submit post/comment and subscribe, and only on an explicit manual click. | [`userActions.mdx`](https://github.com/reddit/devvit-docs/blob/main/docs/capabilities/server/userActions.mdx) |
| Devvit app calls **your own server** (outbound fetch) | **Dead end.** Personal domains "will not be approved". | [`http-fetch-policy.md`](https://github.com/reddit/devvit-docs/blob/main/docs/capabilities/server/http-fetch-policy.md) |
| Devvit app calls **Claude** | **Dead end.** Only OpenAI and Gemini are allowed AI providers. | same |
| An app that spans many subs | **Partial.** Each install is per-sub with its own data, and cross-sub processing needs extra approval. | migration notes |

### 2.3 The viable path: Devvit app as the hands, Electron as the face
Devvit has a documented, sanctioned way for an outside service to start a request: **External Endpoints with managed tokens**. ([`external-endpoints.mdx`](https://github.com/reddit/devvit-docs/blob/main/docs/capabilities/server/external-endpoints.mdx)) It is for "an external service initiates requests to your app", which is what a desktop console does, so this is not a workaround.

```
Electron app ──HTTPS + managed token──► <app>-<subId>-external.devvit.net/external/…
                                         │  (Devvit server, runs on Reddit)
                                         ▼
                              reddit.getModQueue(), reddit.approve(), …  ──► Reddit
```

Constraints we have to design around (all from the docs):
1. **Limited-access feature.** Reddit must grant access via a request form. *This replaces the Data API application as the long pole. Submit it now.*
2. **Everything executes as the app account**, and the mod log will show the app's username. We mitigate this by writing "via Commander, requested by u/you" into mod notes and removal notes (§3.3).
3. Limits: **5 requests/second**, 10 MB bodies, HTTPS only. There is **no push** to the desktop, so the app polls (§3.2). A single poll returns a batch of deltas, so the budget is comfortable.
4. **Managed tokens are global across all installs** of the app, so a leaked token controls every sub. See §7.
5. The app account must be added as a moderator with scoped permissions. For subs where you aren't the owner (backup mod, co-mod), their owners have to agree to install it. **Start on the new sub.**
6. Beyond 200 subscribers an app needs Reddit's review, which "typically takes approximately one week" and repeats when we publish changes.

**Verdict:** Devvit is **not a dead end** for what you want, but it changes the shape. The desktop app can't be "you" to Reddit. It is the control surface for a Devvit app that acts as the app account, and the approval that matters is External Endpoints access. If that is denied, the fallback is a **Devvit-only** build: a mod dashboard rendered inside Reddit (menu actions, mod-only custom post or webview) with no desktop app and no MCP, and only the UX survives. So run a **feasibility spike first** (§9).

---

## 3. Architecture

```
┌────────────────────── Electron app (local) ──────────────────────┐
│ Renderer (React/TS/Vite)        Main process (Node/TS)           │
│ Dashboard · Queue · Modmail ◄─► Command Bus ◄─── MCP server      │
│ Users · Log · Settings   IPC    ├ Policy engine (risk tiers)     │
│                                 ├ Sync engine (delta polling)    │
│                                 ├ SQLite cache + audit log       │
│                                 └ BridgeClient ──► Devvit app    │
└──────────────────────────────────────────────────────────────────┘
        Claude Desktop / Claude Code ──MCP (127.0.0.1)──► Command Bus

┌────────── Devvit app "Commander Bridge" (hosted by Reddit) ──────┐
│ /external/* endpoints (managed token + signed envelope)          │
│ Triggers → Redis change feed (IDs + minimal metadata, TTL)       │
│ reddit.* mod calls · policy re-check · deletion handling         │
└──────────────────────────────────────────────────────────────────┘
```

Monorepo (pnpm): `apps/desktop` (Electron), `apps/bridge` (Devvit Web app), `packages/protocol` (zod schemas and types shared by both, so the wire format is typed end to end), `packages/fixtures`.

### 3.1 Provider abstraction (kept, simplified)
The desktop's `RedditProvider` interface now has two implementations: **BridgeProvider** (real) and **FixtureProvider** (recorded data for offline dev and CI). The UI, cache and MCP never know which is active, so the spike result can't strand the front end.

### 3.2 Sync
- Each sub install exposes `GET /external/changes?since=<cursor>` returning deltas (new or changed queue items, reports, modmail, mod actions, deletions). Triggers append to a per-install Redis change feed, and the desktop polls on an adaptive interval (≈3s focused, ≈30s background).
- `POST /external/snapshot` lets the desktop rebuild its view from live Reddit reads (`getModQueue` etc.) on first run or after a gap, so we store minimal data on Reddit's side.
- All 5 rps budget goes through one limiter. User actions preempt background sync.

### 3.3 Commands and attribution
Desktop → `POST /external/command` with a **signed envelope** `{id, sub, type, args, requestedBy, source: "ui"|"mcp", ts, sig}`. The bridge verifies the signature, checks the policy and replay protection, runs the reddit call, and writes a **mod note** (`requested by u/you via Commander (ui|mcp)`) so the human and the channel are on record even though the mod log shows the app account.

### 3.4 Security (desktop)
`contextIsolation`, `sandbox`, strict CSP and a typed preload; renderer has no Node and no network. Tokens and the signing key live in the OS keychain (Electron `safeStorage`) and never reach the renderer or the MCP. All Reddit text is rendered through a sanitising markdown pipeline.

---

## 4. Stack
| Concern | Choice |
|---|---|
| Desktop | **Electron + electron-vite** (Vite for main, preload and renderer), TypeScript strict |
| Bridge | **Devvit Web** (`@devvit/web`, Hono or Express server, TypeScript) |
| UI | React, Tailwind, Radix primitives, cmdk, TanStack Query and Virtual, Zustand |
| Storage | better-sqlite3 + Drizzle, FTS5 |
| Contracts | zod in `packages/protocol` (IPC, bridge wire format, MCP schemas) |
| MCP | `@modelcontextprotocol/sdk`: Streamable HTTP on `127.0.0.1` plus a stdio shim |
| Tests | Vitest, Playwright for Electron (FixtureProvider), Devvit's playtest and `devvit test` for the bridge |

---

## 5. UX (unchanged from v1 in essence)
- **Dashboard:** sub cards (queue, reports, modmail unread, 24h actions, sparkline vs. baseline), a needs-attention list, and a team-activity strip. For a new sub, a **growth and health** panel: new posts, first-time posters, removal rate, response time.
- **Unified Queue:** three panes (filters, virtualised list, detail plus user context card). Keys: `j/k` move · `a` approve · `r` remove (reason picker) · `s` spam · `l` lock · `b` ban · `n` note · `z` undo · `⌘K` palette. Optimistic with undo.
- **User context card:** account age, history in your subs, past removals, bans and notes (from Reddit mod notes via `getModNotes`), and other open reports by the same author.
- **Modmail**, **Users**, **Mod log**, **Macros and removal reasons**, **Settings**.

---

## 6. MCP server and permission policy

Local only: Streamable HTTP on `127.0.0.1` with a revocable bearer token (off by default), plus an `rc-mcp` stdio shim. Claude has no Reddit credentials and no route to Reddit; it can only call the Command Bus.

### 6.1 Risk tiers (your decision: act directly on low risk)
| Tier | Commands (v1 defaults, configurable per sub) | Claude's ability |
|---|---|---|
| **Read** | everything readable | always |
| **Low risk → acts directly** | `approve` · `add_note` · set/clear flair · modmail mark-read, archive, highlight, assign · snooze, tag or label (local) · `undo` | executes immediately, audited, undoable |
| **Medium → propose, one-key approve** | `remove` (with reason) · `mark_spam` · `lock` · sticky/distinguish · `reply_modmail` · apply removal comment or macro | queued in the **AI Proposals tray**; you approve singly or in batch |
| **High → propose, explicit confirm** | `ban`/`mute`/`unban` · any batch above N items · anything touching rules, wiki or AutoModerator config | shown with a summary; no batch approve for permanent bans |
| **Not exposed** | moderator invites, removals and permission changes; sub settings | n/a |

`approve` is "low risk" because it is reversible and non-punitive. Note it is still a moderation decision, which is why the tray and audit log show every one.

### 6.2 Tool catalogue
Read: `list_subs`, `get_dashboard`, `list_queue`, `get_item`, `get_user_context`, `search_users`, `get_modlog`, `list_modmail`, `get_modmail_thread`, `get_rules`, `get_stats`. Write (all via Command Bus): `approve`, `add_note`, `set_flair`, `modmail_action`, `remove`, `mark_spam`, `lock`, `reply_modmail`, `apply_macro`, `ban`, `undo`, `propose_action(command, rationale)`, `list_proposals`. Resources: `queue://{sub}`, `item://{id}`, `user://{name}`. Prompts: `triage_queue`, `investigate_user`, `summarise_modmail`.

### 6.3 Safety
- Reddit content is returned to Claude wrapped and labelled as **untrusted data**, with tool descriptions telling the model never to follow instructions inside it. The tier system is the actual defence against prompt injection, since an injected "ban everyone" can only reach the proposals tray.
- Rate caps on MCP-initiated writes, a global **kill switch** (tray icon and hotkey) that revokes the MCP token instantly, and a full exportable audit log (`source`, client, args, result, undo token).
- Policy is enforced **twice**: in the desktop Command Bus and again in the Devvit bridge, so a compromised desktop can't exceed what the bridge allows.

---

## 7. Compliance and data handling (Devvit Rules)
From [`devvit_rules.md`](https://github.com/reddit/devvit-docs/blob/main/docs/devvit_rules.md). These apply because Reddit data leaves Reddit into your local cache.
- **Deletions:** honor `PostDelete`, `CommentDelete` and account deletions. The bridge emits tombstones in the change feed, and **the desktop purges the content and any references**, including from the local audit log text. Retention goal is ≤30 days for stored user data (Redis TTL).
- **Data minimisation:** the bridge stores IDs and minimal metadata only, and reads content live.
- **Never:** profile or surveil redditors, sell or share data, or train models on it. Sending content to Claude is a user-initiated local flow, but it is exactly the kind of thing to **state plainly in the External Endpoints request** and get a yes on, rather than assume.
- **Secrets:** the managed token is global across installs, shown once, and revocable. It sits in the keychain, rotates from Settings, and requests also need the signed envelope, so the token alone can't issue commands.
- **Consent of co-mods:** the app account becomes a mod. It's installed only where that sub's mod team agrees. App description and README must explain this plainly.
- **Naming:** no Reddit trademarks or Snoo in the app name or logo. "Reddit Commander" as a repo name is fine, but the app's public name should avoid "Reddit".

---

## 8. Risks and open questions
| # | Risk | Mitigation |
|---|---|---|
| R1 | External Endpoints access denied or slow (limited-access form) | Submit now; spike on playtest; fallback is a Devvit-only in-Reddit dashboard |
| R2 | Actions show as app account, not you | Mod-note attribution; accepted trade-off. Ask Reddit whether a named app account (e.g. `u/<yourname>-commander`) is fine |
| R3 | 5 rps and no push | Delta feed and adaptive polling; benchmark in spike |
| R4 | Global managed token | Keychain, rotation, signed envelopes, bridge-side policy |
| R5 | Review required >200 subs; repeats per publish | Plan releases; the new sub starts <200 so playtest covers early dev |
| R6 | Data rules (deletion, LLM use) | §7; confirm in the form |
| R7 | Co-mod subs need their owners' consent | New sub first; others optional later |

**Questions for you**
1. Are you the **owner** of the new sub (can you add the app account as a mod and install apps)? Has it been created yet, and with what name?
2. Is the low/medium/high table in §6.1 right? In particular, should `remove` for **clear-cut cases** (e.g. matched AutoMod filters) also be direct?
3. OK to request the External Endpoints access using a **dedicated app account name**, e.g. `<yourname>-commander`? Note that this makes the app account a visible moderator on your sub.
4. Windows, macOS or Linux for the desktop app?

---

## 9. Plan preview (next stage)
- **M0 Spike (gates everything):** Devvit app with one install on a test sub (<200 members), one external endpoint (`/changes` plus `/command approve`), a throwaway Node client proving the full loop and latency, plus the access request filed.
- M1 Scaffold monorepo, protocol, FixtureProvider, Electron shell. M2 Queue and sync. M3 Command Bus, policy, audit, undo. M4 Dashboard and user context. M5 MCP server and the proposals tray. M6 Modmail and modlog. M7 Hardening, review submission, packaging.
