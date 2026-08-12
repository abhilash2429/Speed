# Macky — single project context (for planning with an AI)

**Paste or attach this one file** when talking to an AI about product planning, feature
design, or codebase-aware brainstorming. It reflects the **current shipped architecture**
as of the docs refresh (Aug 2026). It is self-contained — you do not need other repo files
for planning. For implementation work inside the repo, also follow `AGENTS.md` + nested
guides.

---

## 1. One-sentence product

Macky is a **macOS notch-resident voice assistant**: hold a key, speak, it acts (local Mac
tools + cloud apps) and talks back in ~250–500 ms. Voice in, action out — not a chatbot
window.

---

## 2. Product principles (non-negotiable for new features)

1. **Notch-first, unobtrusive.** Idle = looks like the notch. Listening / thinking /
   speaking / ordinary tool work stay **inside the closed notch** (waveform, pulse,
   narration, connector logo). Do not invent UI that constantly expands or steals focus.
2. **Expansion is earned.** Panel (fixed **680×340**) opens for: hover/tap, auth/onboarding,
   settings, file drop, Agents / Connectors. Not for every voice turn.
3. **Realtime is the only voice layer.** Background agents never block voice, never become
   `voiceState` / `operationState`, never require a second conversational brain.
4. **Half-second loop wins.** No per-action confirm UI for routine tools
   (`require_approval: "never"`). Dangerous/irreversible actions get **spoken** confirmation
   only.
5. **Secrets stay on the Worker.** App never holds Azure/Composio/Resend keys.
6. **Two integration buckets only.** Mac = Swift function tools. Cloud = Composio MCP.
   No LMCP, no second local MCP mega-server, no one-off OAuth clients per SaaS unless
   product explicitly decides.
7. **Legacy names stay.** Folder/scheme `leanring-buddy` (typo), some `Auren*` symbols —
   do not rename for branding.

### Explicitly out of v1 (do not plan these as near-term without calling it a strategy shift)

- Ambient always-on screen recording  
- Ambient memory / “second brain” across sessions  
- Local search over general activity  
- Product quotas / trials / subscription gating on the Worker (dev shape is open)  
- Auto-expanding Claude Code–style multi-step step lists for every tool chain (current UX
  is closed-notch narration; building that would be a deliberate UX feature)

---

## 3. Architecture (current)

```
Mac app (leanring-buddy/ → Macky.app)
  Notch NSPanel + CompanionManager + RealtimeClient
  Local Swift tools | Dictation | General Agent (encrypted local state)
  MackyAgentExecutor.xpc (sandboxed JS, 5s)
           │  WSS/HTTPS
           ▼
Cloudflare Worker "realtime-proxy"
  /realtime              → Azure gpt-realtime-2.1          (byte proxy)
  /dictation/realtime    → Azure gpt-realtime-2.1-mini     (text-only)
  /agent-config|response → Azure gpt-5.6-sol (sol-medium)  (stateless SSE)
  /composio-* , /spotify-play , /auth/*
           │
           ├─▶ Azure AI Foundry
           └─▶ Composio MCP gateway (250+ apps; OAuth)
```

**Default Worker host** (single Swift constant `WorkerEndpoints.baseHost`):
`realtime-proxy.winky-secrets.workers.dev`

**Bundle:** `com.speedmac.Macky` · **Min macOS:** 14.2 · **SPM:** Sparkle, PostHog  
**Heritage:** Clicky fork (kept NSPanel / ScreenCaptureKit / CGEvent tap; removed
AssemblyAI + Claude + ElevenLabs). Ignore old Clicky/Auren/Boring Notch assumptions.

---

## 4. How users interact today

| Action | Chord / UI |
|--------|------------|
| Assistant push-to-talk | **Ctrl + Option** (configurable; must not collide with dictation) |
| Dictation into focused field | **Ctrl + Fn** (reserved) |
| Expand panel | Hover / click notch |
| Panel tabs | **Home \| Agents \| Connectors** (+ gear settings, Files via drop) |
| Sign-in | Magic link → `Macky://auth` or Skip for now; anonymous session still bootstraps |
| Connect cloud apps | Connectors tab or mid-turn Composio Connect link → `Macky://connected` |

### Closed-notch states

Idle → Listening (waveform) → Thinking (pulse) → Speaking (output waveform) → Executing
(narration + connector branding) → optional agent completion notice when voice idle.

### Voice turn

Hold PTT → barge-in stops playback → PCM16 24 kHz streams on **persistent** `/realtime`
socket → release → model speaks while tools may run → reconnect mid-utterance **discards**
the utterance (no partial replay).

### Dictation (separate path — do not merge into assistant)

AX-validate focused editable field **first** → short-lived `/dictation/realtime` →
`gpt-realtime-2.1-mini` text-only, no tools/MCP/audio/assistant context → one final
transcript → revalidate same field → insert once. Modes: Literal / Clean / Smart. Focus
loss → Copy in notch, never silent clipboard overwrite. Terminal = insert only, never run.

---

## 5. What already ships (capability inventory)

### Native realtime tools (Swift — keep names stable)

`get_screen_context`, `get_focused_text_context`, `apply_focused_text`, `control_cursor`,
`volume_up`, `volume_down`, `toggle_do_not_disturb`, `lock_screen`, `open_url_in_chrome`,
`new_chrome_tab`, `control_music`, `play_spotify_track`, `get_calendar_events`,
`create_calendar_event`, `find_free_slot`, `create_reminder`, `open_app`

Notes: screen is **on demand** (cursor display by default; `all_screens` optional).
Cursor coords bind to fresh capture + `display_id`. `play_spotify_track` → Worker
`POST /spotify-play` (fast path, not MCP discovery).

### Composio connectors (client catalog)

`gmail`, `slack`, `googlecalendar`, `notion`, `github`, `linear`, `spotify`  
(Each needs a Composio dashboard auth config.)

### Background General Agent

- Bridge tools from realtime: `spawn_agents`, `list_agent_tasks`, `get_agent_result`,
  `cancel_agent`, `open_agents_page`
- ≤3 concurrent jobs; queue rest; no agent-spawns-agent; sleep/quit pauses; resume on wake
- Local tools: `read_attachment`, `run_javascript`, `create_artifact`, `ask_question`,
  `final_result`
- Guards: 5s JS XPC timeout, 10 files / 50 MB attachments, bounded reads, no-progress breaker
- State encrypted on Mac; Worker is **stateless**; 4h recent cards / 30d terminal history
- Skills = immutable instruction packages (not agents); attach to background tasks only

### Auth / session

Anonymous `POST /auth/anonymous` → Keychain `macky.session` → optional magic-link verify
**replaces** identity (does **not** migrate Composio connections from anon).

### Permissions needed

Microphone, Accessibility (hotkey + dictation + focused text), Screen Recording (context),
Calendar/Reminders when used.

---

## 6. Repo map (where to extend)

| Path | Role |
|------|------|
| `leanring-buddy/` | macOS app sources |
| `leanring-buddy.xcodeproj/` | Targets: app, tests, XPC; SPM pins |
| `leanring-buddyTests/` | XCTest |
| `MackyAgentExecutor/` | Sandboxed XPC JS (`com.speedmac.Macky.AgentExecutor`) |
| `worker/src/index.ts` | All Worker routes (no package.json) |
| `scripts/release.sh` | Production ship pipeline (do not run casually) |
| `AGENTS.md`, `MACKY.md`, `*/AGENTS.md` | Living engineering/product docs |

### App ownership (start here when designing a change)

| Concern | Primary symbols / files |
|---------|-------------------------|
| Entry | `App/leanring_buddyApp.swift` |
| Central state | `CompanionManager` (`@MainActor`) |
| Notch UI | `NotchPanelController`, `NotchUIModel`, `NotchContainerView`, `Auren*`, `AgentsPanelView`, `PanelTabBar`, `DesignSystem` |
| Realtime / tools / MCP / audio | `RealtimeClient` |
| Hotkeys | `GlobalPushToTalkShortcutMonitor`, `HotkeyConfiguration` |
| Mic (assistant) | `BuddyDictationManager` |
| Dictation | `DictationCoordinator`, `DictationTargetIntegration` |
| Worker URLs | `Networking/WorkerEndpoints.swift` **only** |
| Connectors catalog | `ConnectorRegistry` |
| Agents | `Harness/Agents/*` + `AgentsPanelView` + XPC |
| Auth | `AuthManager`, `AuthView` |
| Local integrations | `Connectors/*`, `SystemIntegration/*` |
| Analytics | `MackyAnalytics` (PostHog; no-ops without key) |

### Worker routes

| Route | Purpose |
|-------|---------|
| `GET /realtime` | Byte-proxy WS → gpt-realtime-2.1 |
| `GET /dictation/realtime` | Text-only WS → gpt-realtime-2.1-mini |
| `GET /agent-config` | Agent capability + kill switch |
| `POST /agent-response` | Stateless agent SSE → gpt-5.6-sol |
| `GET /composio-config` | `{ url, key }` for MCP |
| `POST /composio-connect` | OAuth connect link |
| `GET /composio-connections` | Active toolkits |
| `POST /spotify-play` | Direct search→play |
| `POST /auth/anonymous` `/auth/magic-link` `/auth/verify` | Sessions |
| `GET /auth/open` `/auth/connected` | HTML → deep links |

Secrets: `AZURE_OPENAI_API_KEY`, `COMPOSIO_API_KEY`, `RESEND_API_KEY`  
KV: `AUTH_TOKENS`, `SESSIONS`

---

## 7. Feature-planning cheat sheet

Use this when proposing work — map the idea to the right layer first.

| If the feature is about… | Prefer extending… | Avoid… |
|--------------------------|-------------------|--------|
| Faster / better voice conversation | `RealtimeClient` prompt/tools, Worker `/realtime` proxy only | Second voice model, per-utterance connect |
| New Mac capability (Finder, Mail.app, etc.) | New **Swift** function tool + register in `RealtimeClient` | LMCP / local MCP server / shell-out without sandbox plan |
| New cloud app (Asana, HubSpot, …) | Composio toolkit + `ConnectorRegistry` + dashboard auth config | Custom OAuth client in the app |
| Long research / docs / artifacts | General Agent tools/UI (`Harness/Agents`, `/agent-*`) | Blocking the realtime turn |
| Safer text insertion | `Dictation/*` only | Routing dictation through assistant tools |
| New notch chrome / panels | `Overlay/*` + `CompanionManager`; keep closed-default | Expanding for every tool call by default |
| Backend / keys / models | `worker/src/index.ts` | Putting secrets in the app |
| Sandboxed compute for agents | `MackyAgentExecutor` (explicit security review to widen) | Network/FS/shell in XPC without a product decision |
| Shipping | `scripts/release.sh` (human-triggered only) | CI assumptions — no `.github` workflows in tree today |

### Design questions to answer before coding a feature

1. Does it stay in the **closed notch**, or does it **earn** panel space?  
2. Is it a **realtime turn** (seconds) or a **background agent** (minutes)?  
3. Mac-native Swift tool, Composio MCP, Worker route, or Agent local tool?  
4. What TCC permissions / OAuth / encryption does it need?  
5. Does it preserve barge-in, persistent socket, and no-approval-for-routine-actions?  
6. What is explicitly **not** in scope so we do not scope-creep into memory/ambient/quotas?

---

## 8. Constraints for any implementation agent

- Prefer Xcode builds on macOS; terminal `xcodebuild` can disturb TCC on a real machine.  
- Surgical diffs; preserve unrelated dirty work; never commit secrets.  
- New Swift files need correct Xcode target membership.  
- Worker + Swift client contract changes should land together.  
- Do not run `release.sh` unless explicitly asked.  
- When product brief and code disagree, surface the conflict — do not silently invent UX.

---

## 9. Competition context (strategy only)

Closest: heyclicky (hand-built connectors, older voice chain). Others: Perplexity PC
(local/long-horizon), Raycast AI (keyboard-first), Apple Intelligence (narrower). Macky’s
bet: realtime latency + Composio cloud MCP + native Mac tools + calm notch UI +
background agents for long jobs.

---

## 10. How to use this file in a chat

Good opening prompt:

> Here is Macky’s current project context. Help me plan [feature]. Respect the product
> principles, map the work to the right layer (realtime / native tool / Composio / agent /
> Worker / notch UI), list risks and open questions, and propose a minimal milestone
> sequence against the existing codebase — do not redesign the architecture unless needed.

Keep `AGENTS.md` / `MACKY.md` / nested `*/AGENTS.md` as the living source inside the repo;
refresh **this** file when architecture or shipped capabilities change materially.
