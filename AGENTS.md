# AGENTS.md — Macky (repository root)

This is the **operating manual for AI coding agents** working anywhere in this repository.
Together with [`MACKY.md`](MACKY.md), it is the complete source of project understanding you
should need before editing.

**Read order**

1. This file (root) — always.
2. [`MACKY.md`](MACKY.md) — before any change that touches product behavior, UI, the voice
   pipeline, integrations, branding, or user-facing flows.
3. The **nearest nested `AGENTS.md`** for the folder you will edit (see §3).
4. The specific source file and its direct callers/callees (`rg`).

When current code and `MACKY.md` disagree, **pause and surface the conflict** before
changing behavior — do not silently pick one. Prefer updating the docs when the code is
clearly the shipped source of truth; prefer not inventing product behavior from docs alone
when the code path is incomplete.

A short **User Instructions** section for humans is at the end.

---

## 1. What Macky Is

Macky is a **macOS voice assistant that lives in the notch**. It is not a chatbot or a
copilot you open and type into. You hold a push-to-talk shortcut, speak, and the app
routes audio through a realtime voice model, executes local or cloud tools, and talks
back — usually in under half a second. Voice in, action out.

Core product ideas (full brief: `MACKY.md`):

- The assistant covers the notch edge to edge. Idle, it looks like the notch.
- Listening / thinking / speaking / short tool work are shown **inside** the closed notch
  (waveform, pulse, narration, connector logo) — the panel does not expand for ordinary
  voice turns.
- The panel expands for hover/tap, first-run auth/onboarding, settings, file drops, or the
  Agents / Connectors surfaces. Expansion is earned.
- On displays without a notch, it falls back to a full-width floating bar pinned top center.

### Heritage

Macky is a heavily reworked fork of **Clicky** (MIT-licensed macOS assistant). The macOS
UI primitives remain — `NSPanel`, ScreenCaptureKit, CGEvent push-to-talk tap, design
system. The old brain (AssemblyAI + Claude + ElevenLabs) was removed and replaced with one
Azure realtime voice model, Composio MCP for web services, and Swift-native local tools.

Do **not** assume old Clicky, Auren, makesomething, Boring Notch, or LMCP behavior still
applies unless current files prove it. Some legacy `Auren*` / `makesomething` names remain
in active code on purpose — do not rename them for branding cleanup unless asked.

---

## 2. Architecture at a Glance

```
  ┌──────────────────────────────┐  WebSocket / HTTPS   ┌──────────────────────────┐
  │  Macky macOS app             │ ───────────────────▶ │  Cloudflare Worker       │
  │  (leanring-buddy/)           │  /realtime (bytes)   │  (worker/)               │
  │                              │  /dictation/realtime │  name: realtime-proxy    │
  │  • Notch UI (NSPanel)        │  /agent-config|resp  │                          │
  │  • Push-to-talk + dictation  │  /composio-*         │  • all secrets live here │
  │  • RealtimeClient            │  /spotify-play       │  • Azure realtime proxy  │
  │  • Local Swift tools         │  /auth/*             │  • Composio session      │
  │  • AgentCoordinator/Runtime  │ ◀─────────────────── │  • General Agent SSE     │
  │  • MackyAgentExecutor (XPC)  │                      │  • magic-link auth       │
  └──────────────┬───────────────┘                      └────────────┬─────────────┘
                 │                                                   │
                 │ local tools (EventKit, AppKit,                    │ proxies to
                 │ ScreenCaptureKit, NSWorkspace, CGEvent)           ▼
                 ▼                                     ┌──────────────────────────┐
        macOS-native actions                           │ Azure AI Foundry         │
        (Calendar, Reminders, system,                  │ • gpt-realtime-2.1       │
         apps, screen, cursor, focused text)           │ • gpt-realtime-2.1-mini  │
                                                       │ • gpt-5.6-sol (agents)   │
                                                       └──────────────────────────┘
                                                       ┌──────────────────────────┐
        Cloud services (Slack, Gmail, Spotify, …) ───▶ │ Composio MCP gateway     │
                                                       └──────────────────────────┘
```

**Ownership map**

| Concern | Owner |
|---------|--------|
| App entry + accessory lifecycle | `leanring-buddy/App/leanring_buddyApp.swift` |
| Central UI/session state | `CompanionManager` (`@MainActor`) |
| Visible UI host | `NotchPanelController` + `NotchContainerView` |
| Persistent assistant WebSocket, tools, MCP, audio | `RealtimeClient` |
| Mic capture (assistant) | `BuddyDictationManager` (PCM16 24 kHz mono) |
| Global hotkeys | `GlobalPushToTalkShortcutMonitor` (one CGEvent tap) |
| Dedicated Ctrl+Fn dictation | `DictationCoordinator` + `/dictation/realtime` |
| Background General Agent | `AgentCoordinator` / `AgentRuntime` + Worker `/agent-*` |
| Sandboxed JS for agents | `MackyAgentExecutor` XPC (`AgentJavaScriptExecutorClient`) |
| Worker host (single Swift source of truth) | `WorkerEndpoints.baseHost` |
| Secrets / Azure / Composio / Resend | Worker only — never in the app binary |

**Two integration buckets (shipped)**

1. **macOS-native** — Swift function tools registered on the realtime session (Calendar,
   Reminders, system controls, Chrome helpers, Spotify play fast-path, screen, cursor,
   focused text). No local MCP server / LMCP.
2. **Web services** — one Composio MCP gateway URL wired into `session.update`
   (`require_approval: "never"`). OAuth via Composio Connect links.

The app targets a **hosted** Worker by default. Self-hosting = deploy `worker/` and change
**only** `WorkerEndpoints.baseHost` in
`leanring-buddy/Networking/WorkerEndpoints.swift`.

---

## 3. Repository Map & Nested AGENTS

| Path | What it is | Nested guide |
|------|------------|--------------|
| `CONTEXT.md` | **Single pasteable brief** for planning chats with external AIs | — |
| `MACKY.md` | Product + architecture brief (intent and shipped shape) | — |
| `AGENTS.md` | This file — repo-wide agent operating manual | — |
| `CLAUDE.md` | Redirect stub → `@AGENTS.md` | — |
| `README.md` | Human getting-started guide (not agent source of truth) | — |
| `leanring-buddy/` | Active macOS app (SwiftUI + AppKit) | [`leanring-buddy/AGENTS.md`](leanring-buddy/AGENTS.md) |
| `leanring-buddy.xcodeproj/` | Targets, SPM, signing, file membership | [`leanring-buddy.xcodeproj/AGENTS.md`](leanring-buddy.xcodeproj/AGENTS.md) |
| `leanring-buddyTests/` | XCTest unit tests for the app | covered by xcodeproj + app AGENTS |
| `worker/` | Cloudflare Worker proxy (`realtime-proxy`) | [`worker/AGENTS.md`](worker/AGENTS.md) |
| `MackyAgentExecutor/` | Sandboxed XPC JavaScript helper | [`MackyAgentExecutor/AGENTS.md`](MackyAgentExecutor/AGENTS.md) |
| `scripts/` | Release automation (`release.sh`) | [`scripts/AGENTS.md`](scripts/AGENTS.md) |

> The folder, scheme, and project file are all named `leanring-buddy` (note the typo).
> This is intentional legacy naming. **Do not rename** unless explicitly asked.

There is no `boring.notch/` folder. Ignore older references to one.

### Nested AGENTS duties

- **Root** — cross-cutting architecture, constraints, verification, doc index.
- **`leanring-buddy/AGENTS.md`** — Swift app file map, invariants, risky files.
- **`worker/AGENTS.md`** — routes, secrets, KV, General Agent / dictation contracts.
- **`MackyAgentExecutor/AGENTS.md`** — XPC protocol and sandbox limits.
- **`leanring-buddy.xcodeproj/AGENTS.md`** — targets, SPM, membership rules.
- **`scripts/AGENTS.md`** — release pipeline safety.

When editing a subfolder, follow the nested file’s invariants in addition to this root file.

---

## 4. Interaction Model (shipped)

### Shortcuts

| Mode | Default chord | Notes |
|------|---------------|--------|
| Assistant push-to-talk | **Ctrl + Option** | `HotkeyConfiguration.default`; user-configurable; must not collide with dictation |
| Dictation | **Ctrl + Fn** | Reserved; never assignable as the assistant chord |

One global event tap owns both modes (`GlobalPushToTalkShortcutMonitor`).

### Closed-notch states

| State | What the user sees |
|-------|--------------------|
| Idle | Looks like the notch |
| Listening | Dim waveform reacting to mic |
| Thinking | Soft pulse |
| Speaking | Quiet output waveform |
| Executing | Narration / connector branding in the closed bar (`operationState`, `narrationText`) |
| Agent notice | Transient local completion / needs-input while voice is idle |

### When the panel opens (680×340)

- Hover or click the notch
- Auth / first-run onboarding incomplete
- Settings (gear)
- File drop → Files page
- Agents page presentation (`shouldPresentAgentsPage`) — Home | Agents | Connectors tabs

Ordinary single- or multi-tool voice turns stay in the **closed** notch with live narration.
Do not assume a Claude Code–style auto-expanding step list exists unless you are
implementing that product feature and reconciling with `MACKY.md`.

### Voice turn lifecycle (assistant)

1. User holds PTT → barge-in stops playback → mic streams PCM16 24 kHz to `/realtime`.
2. Persistent socket already connected; model reasons with tools (local + Composio MCP).
3. User releases → commit / response; model speaks while tools may still run.
4. Reconnect mid-utterance **discards** the utterance (no partial replay).

### Dictation (separate path)

- Validates focused editable field via Accessibility **before** opening a socket.
- On-demand Worker session: `/dictation/realtime` → Azure `gpt-realtime-2.1-mini`, text-only,
  no tools / MCP / audio out / assistant context.
- On release: one final transcript, style mode (Literal / Clean / Smart), revalidate same
  field, insert once. Never types partials, presses Return, or auto-overwrites clipboard on
  focus loss (offers Copy in the notch). Terminal is insertion-only.

---

## 5. Tools & Integrations

### Native realtime function tools (Swift)

Registered in `RealtimeClient` (names are the protocol surface — keep stable unless
explicitly migrating both prompt and dispatch):

`get_screen_context`, `get_focused_text_context`, `apply_focused_text`, `control_cursor`,
`volume_up`, `volume_down`, `toggle_do_not_disturb`, `lock_screen`, `open_url_in_chrome`,
`new_chrome_tab`, `control_music`, `play_spotify_track`, `get_calendar_events`,
`create_calendar_event`, `find_free_slot`, `create_reminder`, `open_app`.

`play_spotify_track` uses Worker `POST /spotify-play` (server-side search→play), not the
slow MCP discovery chain.

### Composio connectors (client catalog)

`ConnectorRegistry` slugs: `gmail`, `slack`, `googlecalendar`, `notion`, `github`,
`linear`, `spotify`. Each needs a Composio dashboard auth config or `/composio-connect`
404s.

### Background General Agent

- Realtime remains the **only** voice layer. Bridge tools:
  `spawn_agents`, `list_agent_tasks`, `get_agent_result`, `cancel_agent`, `open_agents_page`.
- At most **three** concurrent jobs (`AgentRuntime`); queue the rest; no product quotas.
- Worker owns model (`gpt-5.6-sol` / `sol-medium`), schemas, safety, `store: false`, SSE
  normalization via `/agent-config` + `/agent-response`.
- Local agent tools: `read_attachment`, `run_javascript`, `create_artifact`, `ask_question`,
  `final_result`.
- Task state encrypted on Mac. JS runs in `MackyAgentExecutor` (5s timeout; no shell /
  Python / network / arbitrary FS).
- Agents UI: panel tabs Home | Agents | Connectors; not a separate macOS window.

### Skills

Immutable instruction packages (built-in + user). Users create/draft/duplicate/enable/
delete; saved Skills are not edited in place. Full instructions attach only when a Skill
is bound to a background task. User Skill definitions are encrypted locally.

---

## 6. Auth & Sessions

1. First run: `POST /auth/anonymous` mints `{ sessionToken, composioUserId }` → Keychain
   (`macky.session`) so connectors work before email login.
2. Optional magic link: `POST /auth/magic-link` → email → `GET /auth/open` →
   `Macky://auth?token=…` → `POST /auth/verify` (replaces anonymous identity; does **not**
   migrate Composio connections).
3. UI may **Skip for now** (`macky.authSkippedForNow`) for early testing.
4. Connector OAuth callback: `GET /auth/connected` → `Macky://connected?toolkit=…`.

All Composio routes require `Authorization: Bearer <sessionToken>` and resolve identity
from Worker `SESSIONS` KV — never trust a client-supplied Composio user id.

---

## 7. Hard Constraints

- **Names are frozen.** Do not rename `leanring-buddy` paths, scheme, or project unless
  asked.
- **Build through Xcode on macOS** for normal verification. Terminal `xcodebuild` can
  disturb TCC (Accessibility, Screen Recording, Microphone) on a developer machine.
- **Preserve dirty work.** Do not revert or clean unrelated uncommitted changes.
- **Never commit secrets.** No API keys, `.dev.vars`, Keychain material, Apple signing
  material, OAuth tokens.
- **Keep changes surgical.** Mention leftover legacy branding separately unless cleanup
  is the task.
- **No LMCP / no second local MCP.** Web = Composio MCP; Mac = Swift tools.
- **No one-off OAuth clients** for Slack/Gmail/Spotify/etc. unless explicitly asked.
- **Do not run `scripts/release.sh`** unless the user explicitly requests a release.

---

## 8. Coding Rules

- Before editing: read the file, then callers/callees. For Swift UI/state, start with
  `CompanionManager` plus the view/controller that observes it.
- Prefer existing patterns: `@MainActor` for UI/state owners, `@Published` for observed
  state, async/await for new async Swift, small static methods for local integrations.
- Do not introduce speculative abstractions; milestones stay explicit.
- Keep names descriptive (product-state names, tool names, route paths).
- Comments explain non-obvious timing, macOS/TCC behavior, or protocol constraints — not
  obvious code.
- New Swift files must be added to the correct Xcode target membership
  (`leanring-buddy` and/or `MackyAgentExecutor` / tests).
- Worker route or response-shape changes require updating the Swift client in the same
  change set (or an explicit follow-up called out to the user).

---

## 9. Verification

| Area | How |
|------|-----|
| Swift app | Prefer Xcode build/run on macOS. If unavailable: static review + `rg` callers + confirm `project.pbxproj` membership; state that no Xcode build ran. |
| Unit tests | `leanring-buddyTests` exists; run from Xcode when touching covered modules. |
| Worker | Static review of `worker/src/index.ts`; optional `npx wrangler dev` when secrets exist. Tests: `node --experimental-transform-types --test test/*.test.ts`. Deploy only when asked. |
| XPC | Review protocol + entitlements; do not widen sandbox without an explicit product decision. |
| Scripts | `bash -n scripts/release.sh` only — never a live release as validation. |
| Docs | Keep nested AGENTS links accurate; fix conflicts with code rather than leaving agents to guess. |

---

## 10. Ask Before Changing

- Product expansion rules, voice latency targets, or approval policy for dangerous actions
- Azure model IDs / Worker route names / auth or Composio response shapes
- Adding npm/`package.json` to `worker/`
- Renaming legacy `Auren*` / `leanring-buddy` symbols
- Release destination repo or signing/notarization flow
- Opening the Worker to multi-tenant hardening (rate limits, stronger session binding)

---

## User Instructions

For a human getting this project running locally.

### Prerequisites

- macOS 14.2+ and Xcode
- Node.js + `npx` (Wrangler) if you will run or deploy the Worker
- Cloudflare account (`npx wrangler login`) for Worker work

### Run the app

1. Open `leanring-buddy.xcodeproj` in Xcode.
2. Select the `leanring-buddy` scheme → Build & Run (⌘R). First build fetches Sparkle + PostHog via SPM.
3. Grant Microphone, Accessibility, and Screen Recording when prompted.
4. Sign in via magic link, or **Skip for now**, then hold **Ctrl + Option**, speak, release.
5. Connect cloud apps from the Connectors tab or when the assistant surfaces a Connect link.

Default Worker host is already set in `WorkerEndpoints.baseHost`. You do not need a local
backend for normal use of the hosted deployment.

### Run / deploy the backend

See [`worker/AGENTS.md`](worker/AGENTS.md). From `worker/`: `npx wrangler dev` or
`npx wrangler deploy`. Secrets: `AZURE_OPENAI_API_KEY`, `COMPOSIO_API_KEY`, `RESEND_API_KEY`,
plus `AUTH_TOKENS` and `SESSIONS` KV.

### Ship a release

See [`scripts/AGENTS.md`](scripts/AGENTS.md). From repo root:
`./scripts/release.sh` or `./scripts/release.sh <version> [build]`.
