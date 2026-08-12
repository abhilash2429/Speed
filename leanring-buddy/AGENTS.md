# AGENTS.md — leanring-buddy (the Macky macOS app)

Operating manual for the active macOS app target. Root [`AGENTS.md`](../AGENTS.md) and
[`MACKY.md`](../MACKY.md) still apply — read them first for cross-cutting architecture and
product rules.

> The folder, scheme, and project are named `leanring-buddy` (with the typo) for legacy
> reasons. This **is** the Macky app. Do not rename anything for branding.

Related nested guides:

- [`../leanring-buddy.xcodeproj/AGENTS.md`](../leanring-buddy.xcodeproj/AGENTS.md) — target membership / SPM
- [`../MackyAgentExecutor/AGENTS.md`](../MackyAgentExecutor/AGENTS.md) — XPC JS sandbox used by agents
- [`../worker/AGENTS.md`](../worker/AGENTS.md) — backend the app talks to

---

## 1. What This Folder Is

A SwiftUI + AppKit **macOS accessory app** (no Dock icon) that:

- renders a notch-first assistant UI in a borderless `NSPanel`
- captures push-to-talk and dedicated dictation audio
- streams assistant audio through the Worker to Azure realtime
- exposes local Swift tools + Composio MCP cloud tools
- runs encrypted local General Agent jobs (voice stays on realtime)

All visible product UI lives in that panel. Product name / bundle: **Macky** /
`com.speedmac.Macky`.

---

## 2. Read First (behavior / architecture work)

1. `../MACKY.md` — product brief.
2. `../AGENTS.md` — repo constraints and interaction model.
3. `App/leanring_buddyApp.swift` — entry.
4. `Harness/Session/CompanionManager.swift` — central state.
5. The file you will edit + callers/callees (`rg`).

**Also read when touching…**

| Area | Start here |
|------|------------|
| Notch / geometry | `Overlay/NotchPanelController.swift`, `NotchUIModel.swift`, `NotchContainerView.swift`, `Notchshape.swift`, `WindowPositionManager.swift`, `AurenStatusBar.swift`, `AurenPanel.swift`, `AurenFileDropPanel.swift`, `PanelTabBar.swift`, `AgentsPanelView.swift` |
| Realtime / tools / audio | `Harness/Dispatcher/RealtimeClient.swift`, `BuddyDictationManager.swift`, `GlobalPushToTalkShortcutMonitor.swift`, `Shared/AudioConversionSupport.swift`, `Overlay/VoiceActivityView.swift`, `Overlay/NotchRightActivityView.swift` |
| Dictation | `Dictation/DictationCoordinator.swift`, `DictationModels.swift`, `DictationTargetIntegration.swift` |
| Agents | `Harness/Agents/*`, `Overlay/AgentsPanelView.swift`, `../MackyAgentExecutor/` |
| Connectors / Skills | `Harness/Registry/ConnectorRegistry.swift`, `SkillRegistry.swift`, `Skills/*`, `Overlay/AurenPanel.swift` |
| Worker URLs | `Networking/WorkerEndpoints.swift` (only place to change host) |

---

## 3. File Map

### App lifecycle & state

- `App/leanring_buddyApp.swift` — accessory activation, `Macky://` URL handler, creates
  `CompanionManager` + `NotchPanelController`.
- `Harness/Session/CompanionManager.swift` — `@MainActor` coordinator: permissions,
  shortcuts, `voiceState`, `operationState`, narration, attachments, connector connect
  links, panel onboarding, history, analytics call sites, agent coordinator ownership.

### Harness

- `Harness/Dispatcher/RealtimeClient.swift` — persistent WebSocket, `session.update`,
  event parsing, local tool dispatch, Composio MCP registration, audio I/O, heartbeat,
  reconnect. URLs derive from `WorkerEndpoints`.
- `Harness/Dispatcher/GlobalPushToTalkShortcutMonitor.swift` — one CGEvent tap;
  assistant chord + reserved Ctrl+Fn dictation routing; `HotkeyConfiguration`.
- `Harness/Dispatcher/BuddyDictationManager.swift` — assistant mic → PCM16 24 kHz mono.
- `Harness/Registry/ConnectorRegistry.swift` — Composio toolkit identity + MCP name match
  for notch branding (`gmail`, `slack`, `googlecalendar`, `notion`, `github`, `linear`,
  `spotify`).
- `Harness/Registry/SkillRegistry.swift` — skill identity / connector-dependency catalog.
- `Harness/Agents/` — General Agent stack:
  - `AgentCoordinator`, `AgentRuntime` — orchestration, ≤3 concurrent jobs, sleep/resume
  - `AgentRealtimeBridge` — realtime spawn/list/result/cancel/open tools
  - `AgentAPIClient` — Worker `/agent-config` + `/agent-response`
  - `AgentContracts`, `AgentModels`, `AgentRegistry`
  - `AgentPersistence`, `AgentLocalStore`, `AgentAttachmentStore` — encrypted local state
  - `AgentJavaScriptExecutorClient` — XPC client to `MackyAgentExecutor`
- `Harness/Loop/LoopPlaceholder.swift` — legacy marker; active loop is `AgentRuntime`.

### Notch & panel UI

- `Overlay/NotchPanelController.swift` — owns `NSPanel`, frames, hosts SwiftUI without
  letting `NSHostingView` resize the window.
- `Overlay/NotchUIModel.swift` — geometry + open/closed **only** (no voice/tool state).
  Open size: **680×340** (`NotchConstants.openNotchSize`).
- `Overlay/NotchContainerView.swift` — closed bar + expanded panel; pages:
  `home | agents | connectors | settings | files`; hover/tap/drop/auth/agents presentation.
- `Overlay/PanelTabBar.swift`, `Overlay/AgentsPanelView.swift` — tab chrome + Agents UI.
- `Overlay/AurenStatusBar.swift`, `AurenPanel.swift`, `AurenFileDropPanel.swift` —
  closed status / home+connectors surfaces / file drop.
- `Overlay/Notchshape.swift`, `VoiceActivityView.swift`, `NotchRightActivityView.swift` —
  shape path, waveforms, right-side activity.
- `Overlay/DesignSystem.swift`, `MackyLogo.swift`, `DotMatrixSpinner.swift`,
  `IconPreviewStrip.swift` — tokens and shared chrome.
- `Overlay/WindowPositionManager.swift`, `Shared/AppKitExtensions.swift` — multi-display
  placement and AppKit helpers.

### Auth, dictation, skills, settings

- `Auth/AuthManager.swift`, `Auth/AuthView.swift` — anonymous + magic-link session;
  Keychain; deep links. Base URL from `WorkerEndpoints.httpsBase`.
- `Dictation/DictationCoordinator.swift` — Ctrl+Fn lifecycle, isolated socket, insert.
- `Dictation/DictationModels.swift`, `DictationTargetIntegration.swift` — modes, glossary,
  AX target snapshot / revalidation / safe insert.
- `Skills/SkillsWindowController.swift`, `SkillsWindowView.swift`,
  `SkillCatalogStore.swift`, `AgentSkillDraftingProvider.swift` — Skills catalog + draft.
- `Settings/HotkeySettingsView.swift`, `Settings/DictationSettingsView.swift`.

### Local integrations (Swift tools — no cloud, no LMCP)

- `Connectors/CalendarIntegration.swift`, `RemindersIntegration.swift` — EventKit.
- `SystemIntegration/SystemControlsIntegration.swift` — volume, DND, lock, etc.
- `SystemIntegration/AppLauncherIntegration.swift` — `NSWorkspace`.
- `SystemIntegration/CompanionScreenCaptureUtility.swift` — ScreenCaptureKit on demand.
- `SystemIntegration/CursorControlIntegration.swift` — CGEvent cursor ops.
- `SystemIntegration/FocusedTextIntegration.swift`, `ForegroundAppContext.swift` —
  focused-field context for assistant tools / policies.

### Networking, analytics, config

- `Networking/WorkerEndpoints.swift` — **single** `baseHost` + every derived URL
  (realtime, dictation, agent, composio, spotify, auth base).
- `Analytics/MackyAnalytics.swift` — PostHog wrapper; no-ops without `POSTHOG_API_KEY`.
  Call sites in `CompanionManager` / `RealtimeClient`.
- `Info.plist`, `leanring-buddy.entitlements`, `Assets.xcassets/` — permissions, URL
  scheme `Macky://`, sandbox/capabilities, icons.

### Tests (sibling folder)

`../leanring-buddyTests/` — XCTest coverage for agents, dictation models, skills,
notch activity, attachments, foreground context. Prefer adding tests next to touched
logic when practical; run from Xcode.

---

## 4. Active Architecture Notes

- **Persistent assistant socket.** Connect once; stay connected. Composio MCP config is
  fetched **concurrently** with socket open; late config triggers a follow-up
  `session.update`. Mid-utterance reconnect **discards** the utterance (no partial replay).
- **One Composio MCP entry** in session tools (`server_label: "composio"`,
  `require_approval: "never"`). Native tools are separate function tools.
- **Screen context on demand** — default capture is the cursor’s display; `all_screens`
  opts into every monitor. Coordinate actions need a fresh current-turn capture +
  `display_id` on multi-display.
- **Realtime is the only voice layer.** `AgentRealtimeBridge` registers async agent tools
  before `connect()`, but running jobs never feed `voiceState` / `operationState` /
  realtime tool activity. Idle notch may show a local agent notice.
- **Agent state is local-encrypted.** Worker normalizes Azure Responses SSE into protocol
  v1; it does not store tasks.
- **`AgentRuntime`:** ≤3 concurrent jobs; queue rest; steering/cancel at safe boundaries;
  pause on sleep; resume after wake/relaunch.
- **Panel expansion triggers (code):** hover/tap, incomplete auth/onboarding, settings,
  file drop → Files, Agents presentation. Tool narration for live turns stays in the
  **closed** status bar unless you are implementing a new expansion product feature.

### Native tool names (keep stable)

`get_screen_context`, `get_focused_text_context`, `apply_focused_text`, `control_cursor`,
`volume_up`, `volume_down`, `toggle_do_not_disturb`, `lock_screen`, `open_url_in_chrome`,
`new_chrome_tab`, `control_music`, `play_spotify_track`, `get_calendar_events`,
`create_calendar_event`, `find_free_slot`, `create_reminder`, `open_app`.

Agent bridge tools: `spawn_agents`, `list_agent_tasks`, `get_agent_result`,
`cancel_agent`, `open_agents_page`.

---

## 5. Invariants (do not break)

- Keep `CompanionManager` and UI-observed state on `@MainActor`.
- Keep the assistant realtime socket persistent; preserve heartbeat/reconnect unless the
  task is specifically about connection reliability.
- Preserve **barge-in**: PTT interrupts playback before a new capture.
- Closed notch stays small. Do not expand for ordinary listening/thinking/speaking/tool
  turns.
- Attach file/image context **before** `requestResponse()`.
- Agent attachments are explicit only — never route Agents composer drops through Files,
  never expose original source URLs to the model, never store copied files plaintext.
- Do not merge background-agent execution into Realtime active-state properties.
- Web services go through Composio MCP — no one-off OAuth/API clients for those apps
  unless explicitly asked.
- Do not rename `Auren*` files/symbols just for branding.
- Do not hardcode Worker hosts outside `WorkerEndpoints`.
- Dictation chord **Ctrl + Fn** stays reserved and conflict-checked against the assistant
  hotkey.

---

## 6. Risky Files (state the reason before changing)

| File | Why |
|------|-----|
| `RealtimeClient.swift` | Protocol, heartbeat, tools, MCP, audio |
| `CompanionManager.swift` | Central state transitions |
| `GlobalPushToTalkShortcutMonitor.swift` | Global event tap; breaks core UX if wrong |
| `NotchPanelController.swift` / `NotchUIModel.swift` | Geometry, animation, click-through |
| `NotchContainerView.swift` | Expansion / page routing |
| `DictationCoordinator.swift` | Safety-critical insert path |
| `AgentRuntime.swift` / `AgentPersistence.swift` | Concurrency + encrypted store |
| `DesignSystem.swift` | Shared tokens |
| `Info.plist` / entitlements | Permissions, URL scheme, sandbox |
| `WorkerEndpoints.swift` | Every backend URL |

---

## 7. Validation

- Prefer **Xcode on macOS**, not terminal `xcodebuild` (TCC risk).
- If you cannot build: static review + `rg` + confirm new files are in
  `../leanring-buddy.xcodeproj/project.pbxproj`. Say that no Xcode build ran.
- When adding Swift sources, set correct target membership (`leanring-buddy` and/or tests /
  XPC).
- When removing a symbol, remove only imports/code your change made unused — no drive-by
  dead-code deletion.
- Run relevant `leanring-buddyTests` from Xcode when touching covered modules.

---

## User Instructions

1. Open `../leanring-buddy.xcodeproj`, select scheme `leanring-buddy`, ⌘R (macOS 14.2+).
2. Grant **Microphone**, **Accessibility**, **Screen Recording** (and Calendar/Reminders
   when prompted).
3. Sign in via magic link (`Macky://auth`) or **Skip for now**.
4. Hold **Ctrl + Option** (default), speak, release. Dictation: **Ctrl + Fn** in a text field.
5. Connect cloud apps from **Connectors** or when a Connect link appears.

Backend: hosted Worker by default (`WorkerEndpoints.baseHost`). Local/self-host:
see `../worker/AGENTS.md`.
