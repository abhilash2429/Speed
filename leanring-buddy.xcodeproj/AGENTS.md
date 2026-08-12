# AGENTS.md — leanring-buddy.xcodeproj (Xcode project metadata)

Operating manual for the Xcode project configuration. Root [`AGENTS.md`](../AGENTS.md)
still applies. App sources: [`../leanring-buddy/AGENTS.md`](../leanring-buddy/AGENTS.md).
XPC helper: [`../MackyAgentExecutor/AGENTS.md`](../MackyAgentExecutor/AGENTS.md).

---

## 1. Purpose

This folder defines targets, build settings, Swift Package products, resources,
entitlements wiring, and source-file membership for Macky. Everything that tells Xcode
*how* to build `leanring-buddy/` and `MackyAgentExecutor/` lives here.

---

## 2. Layout

- `project.pbxproj` — project graph: targets, phases, settings, file refs, SPM links.
  Authoritative; prefer edits via Xcode when possible.
- `project.xcworkspace/` — workspace wrapper; `xcshareddata/swiftpm/Package.resolved`
  pins SPM versions.
- `xcuserdata/` — per-user Xcode state. Avoid unless intentionally changing shared schemes.

---

## 3. Current Project Facts

| Fact | Value |
|------|--------|
| Main app target | `leanring-buddy` → product **`Macky.app`** |
| Bundle ID | `com.speedmac.Macky` |
| XPC target | `MackyAgentExecutor` → `MackyAgentExecutor.xpc` (`com.speedmac.Macky.AgentExecutor`) |
| Test target | `leanring-buddyTests` (sources under `../leanring-buddyTests/`) |
| Min macOS | 14.2 |
| Swift (project setting) | 5.0 |
| SPM products | **Sparkle** (auto-update), **PostHog** (analytics; transitive PLCrashReporter) |

Legacy path/scheme name `leanring-buddy` is intentional — do not rename unless asked.

---

## 4. Rules

- Edit `project.pbxproj` only when necessary: add/remove sources or resources, build
  settings, package products, entitlements, or embed phases.
- New Swift under `leanring-buddy/` must be in the **`leanring-buddy`** target (and tests
  target if applicable). Files missing membership silently never ship.
- New XPC sources must be in **`MackyAgentExecutor`**; keep Embed XPC Services intact.
- Package add/update is a dependency decision — call it out; commit `Package.resolved`
  deliberately.
- Preserve signing, bundle id, and version settings unless the task is distribution /
  identity.

---

## 5. Validation

- After any `project.pbxproj` edit, inspect the diff hunk manually — easy to corrupt.
- Prefer opening the project in Xcode and building the `leanring-buddy` scheme.
- If you cannot build, say so and rely on static review of the pbxproj diff + file
  reference / PBXSourcesBuildPhase membership.

---

## User Instructions

- **Open:** `open leanring-buddy.xcodeproj` from the repo root.
- **First build:** once, so SPM downloads Sparkle + PostHog (Sparkle CLI tools are also
  required by `scripts/release.sh`).
- **Add a source file:** add to the right group in Xcode; tick the correct target
  membership checkbox.
- **Update packages:** *File ▸ Packages ▸ Update to Latest Package Versions*; commit
  `Package.resolved` on purpose.
- **Stuck SPM:** *File ▸ Packages ▸ Reset Package Caches*, then rebuild.
