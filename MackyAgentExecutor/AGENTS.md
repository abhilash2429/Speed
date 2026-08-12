# AGENTS.md — MackyAgentExecutor (sandboxed XPC JavaScript)

Operating manual for the XPC service that runs **bounded JavaScript** for Macky's General
Agent. Root [`AGENTS.md`](../AGENTS.md) and [`MACKY.md`](../MACKY.md) still apply.

App-side client: `leanring-buddy/Harness/Agents/AgentJavaScriptExecutorClient.swift`.  
App nested guide: [`../leanring-buddy/AGENTS.md`](../leanring-buddy/AGENTS.md).  
Xcode membership: [`../leanring-buddy.xcodeproj/AGENTS.md`](../leanring-buddy.xcodeproj/AGENTS.md).

---

## 1. Purpose

`MackyAgentExecutor` is a separately embedded **XPC service**
(`com.speedmac.Macky.AgentExecutor`) that executes the General Agent tool
`run_javascript`. It exists so agent JS cannot reach shell, Python, network, or arbitrary
filesystem APIs available to the main app process.

The Worker advertises `run_javascript` as a fixed local tool; the Mac runtime calls this
service and returns the tool output on the next stateless `/agent-response` turn.

---

## 2. Files

| File | Role |
|------|------|
| `main.swift` | `NSXPCListener.service()` entry; installs the exported interface |
| `MackyAgentExecutorXPCProtocol.swift` | `@objc` protocol: `executeRequest(_:withReply:)`, `cancelRequest(_:)` |
| `AgentJavaScriptExecutorService.swift` | Execution implementation + timeout / cancel handling |
| `Info.plist` | XPC bundle metadata |
| `MackyAgentExecutor.entitlements` | Sandbox / capability surface — treat as security-critical |

Built as target **MackyAgentExecutor** → `MackyAgentExecutor.xpc`, embedded in `Macky.app`
via the app target’s Embed XPC Services phase.

---

## 3. Contract (do not widen casually)

- Client service name: `com.speedmac.Macky.AgentExecutor`
- Client timeout: **5000 ms** (product structural guard; keep in sync with docs/prompts)
- No shell execution bridge
- No Python runtime bridge
- No network access for agent scripts
- No arbitrary filesystem bridge — only what the explicit request payload allows
- Cancellation must be honored via `cancelRequest`

If a task needs more power (network, FS, subprocess), that is a **product decision**, not
a silent sandbox expansion. Surface it; do not “just add” entitlements.

---

## 4. Invariants

- Keep the XPC boundary: do not move JS execution into the main app target to “simplify.”
- Keep protocol methods backward-compatible or migrate client + service in one change.
- Do not log full script source or tool payloads in production paths.
- Entitlements changes require an explicit security rationale in the PR/commit message.

---

## 5. Validation

- Static review of protocol + service + entitlements together with
  `AgentJavaScriptExecutorClient`.
- Confirm the XPC target still embeds into the app after `project.pbxproj` edits.
- Prefer Xcode build of the app scheme (embeds the service). State clearly if no build ran.
- Exercise via General Agent `run_javascript` paths / `leanring-buddyTests` where covered.

---

## User Instructions

You do not run this target alone in day-to-day use. Building/running the `leanring-buddy`
scheme in Xcode builds and embeds `MackyAgentExecutor.xpc` inside `Macky.app`.
