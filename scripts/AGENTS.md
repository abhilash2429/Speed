# AGENTS.md — scripts (release automation)

Operating manual for release/automation scripts. Root [`AGENTS.md`](../AGENTS.md) still
applies. Xcode project facts: [`../leanring-buddy.xcodeproj/AGENTS.md`](../leanring-buddy.xcodeproj/AGENTS.md).

> Scope: **production** release tooling only — not a dumping ground for experimental
> helpers.

---

## 1. Contents

- `release.sh` — full release pipeline (build → sign → DMG → notarize → Sparkle appcast →
  GitHub Release → push appcast).

There is no separate `scripts/README.md` in tree; this file is the guide for both agents
and humans.

---

## 2. What `release.sh` Does

Runs `set -euo pipefail` and, in order:

1. Auto-detects version + build from the latest GitHub Release (or takes args), then
   prompts for confirmation.
2. Archives via `xcodebuild` (scheme `leanring-buddy`).
3. Exports a signed `.app` with Developer ID.
4. Wraps a DMG (create-dmg; optional background image).
5. Notarizes the DMG (`notarytool`).
6. Sparkle EdDSA-signs the DMG (key in Keychain).
7. Updates `appcast.xml`.
8. Creates a GitHub Release with the DMG.
9. Pushes `appcast.xml` to the releases repo.

### Key configuration (top of `release.sh`)

- `SCHEME="leanring-buddy"`
- `APP_NAME="Macky"`
- `GITHUB_REPO="julianjear/makesomething-mac-app"` — existing releases destination;
  retain until an explicit replacement is supplied
- `DMG_BACKGROUND` — optional; absent → create-dmg defaults
- Sparkle CLI tools are discovered from Xcode SPM DerivedData — build in Xcode once first

### Usage

```bash
./scripts/release.sh            # auto-bump marketing + build
./scripts/release.sh 2.0        # set marketing version, auto-bump build
./scripts/release.sh 2.0 10     # set both
```

---

## 3. Safety Rules (for agents)

- **Do not run `release.sh`** unless the user explicitly asks for a release. It signs,
  notarizes, publishes, and pushes — hard to undo.
- Do not modify signing, notarization, Sparkle, GitHub-release, or repo-push steps unless
  the task is specifically about release automation.
- Preserve `set -euo pipefail`.
- Keep paths and repo names explicit; avoid clever expansion for destructive operations.
- Do not add commands that delete outside the repo or a known build-output directory.
- Changing `GITHUB_REPO` requires an explicit replacement destination from the user.

---

## 4. Validation

- Prefer static review of the command flow and quoting.
- Syntax only: `bash -n scripts/release.sh`
- **Never** perform a live release, notarization, GitHub Release, or push as “validation.”

---

## User Instructions

### One-time prerequisites

1. Xcode with **Developer ID** signing certificate installed.
2. `brew install create-dmg gh`
3. `gh auth login`
4. Notary credentials in Keychain:
   ```bash
   xcrun notarytool store-credentials "AC_PASSWORD" \
       --apple-id YOUR_APPLE_ID --team-id YOUR_TEAM_ID
   ```
   (App-specific password from appleid.apple.com.)
5. Sparkle EdDSA key in Keychain.
6. Build the project in Xcode at least once (SPM + Sparkle CLI tools).

### Ship it

```bash
./scripts/release.sh          # or pass explicit version / build
```

The script prints version/build/previous release and waits for `y`. If the tag already
exists on GitHub it exits early.
