# MACKY.md — product & architecture brief

This is the product source of truth for **what Macky is** and **how it is supposed to
behave**. AI agents must read it before changing product behavior, UI, the voice
pipeline, integrations, or branding.

Engineering operating details (file maps, invariants, verification) live in
[`AGENTS.md`](AGENTS.md) and nested `AGENTS.md` files. When this brief and the current
code disagree, **pause and surface the conflict** — do not silently invent behavior.

---

## What we're building

A macOS AI assistant that lives in the notch. Not a chatbot, not a copilot, not another
thing you open and type into. You press a button, talk, and it does the thing.

That's the whole idea. Your Mac should do what you tell it — play a song, message someone,
check your calendar, open something, run a long task in the background. No switching apps,
no typing, no clicking around. **Voice in, action out.**

---

## Where it came from

Built on top of Clicky — an open-source macOS AI assistant (MIT). It already solved the
hard macOS problems: menu-bar / overlay panel, multi-monitor screen capture, system-wide
push-to-talk, rendering over everything.

We kept the macOS UI shell. We gutted the old brain (AssemblyAI → Claude → ElevenLabs) and
replaced it with:

- one persistent realtime voice model (Azure `gpt-realtime-2.1`)
- Swift-native local tools for macOS actions
- Composio MCP for cloud apps
- a Cloudflare Worker that holds every secret and proxies sockets / agent traffic

---

## The notch

The assistant lives in the notch. Covers it edge to edge. Idle, you don't notice it —
black, sitting there like the hardware cutout.

When you talk to it, the notch **stays the same size**. Listening, thinking, speaking, and
ordinary tool work are shown with subtle motion and short narration **inside** the closed
footprint: waveform while you talk, soft pulse while it thinks, quiet glow / waveform when
it speaks, connector branding + status text while tools run. Unobtrusive. You know it's
working without it owning the screen.

The panel expands only when there is a real surface to show:

- you hover or click the notch
- first-run auth / onboarding / settings
- file input (Files page)
- the Agents or Connectors experience inside the fixed **680×340** panel

Expansion is intentional. It earns the space. On external monitors without a notch it falls
back to a full-width floating bar pinned top center — same logic.

Expanded panel navigation: **Home | Agents | Connectors** (settings via gear; Files via
drop). Agents is a page inside the notch panel, not a separate macOS window.

---

## The voice pipeline

One persistent WebSocket. One realtime voice brain. No transcription → thinking → TTS
handoffs.

**Assistant model:** Azure AI Foundry `gpt-realtime-2.1`  
**Latency target:** roughly 250–500 ms to first audio  
**Transport:** app ↔ Cloudflare Worker `/realtime` (byte-forwarding proxy) ↔ Azure  
The app never holds the Azure API key.

The model takes raw audio, can call tools while speaking, supports MCP and image input
(on-demand screen context), and keeps a large session context. Prefer this generation —
not older realtime variants — unless the product explicitly migrates.

Default push-to-talk: **Ctrl + Option** (configurable; must not collide with dictation).

### How a turn feels

You hold the shortcut. A dim waveform appears in the notch. You say what you want. You
release. The model may already be deciding what it needs — screen glance, Spotify, calendar
then Slack — the way a person would. You don't configure tool routing.

While tools run it talks (“on it”, “let me check”) because the system prompt tells it to
acknowledge immediately. Through all of this the notch stays **closed** for normal turns;
status and connector logos update in place. When the work finishes, it fades back to idle.

Barge-in: pressing push-to-talk again interrupts current playback before a new capture.

---

## Dictation

Dictation is a **separate safety-first path**, not a shortcut into the assistant.

Hold **Ctrl + Fn** over an editable field. Macky verifies that exact field through
Accessibility, then opens a short-lived authenticated Worker session
(`GET /dictation/realtime`) to Azure `gpt-realtime-2.1-mini`:

- text-only, PCM16 24 kHz while held
- no tools, no MCP, no audio out, no assistant-session context
- user glossary keyterms may be sent; focused text, selections, titles, URLs, recipients,
  and page contents are **not**

On release: one final transcription, optional style formatting, revalidate the same app and
field, insert once. Never types partial text, presses Return, sends a message, or submits a
form.

If focus changes mid-dictate, Macky does not write into the new control. It keeps the
result transiently in the notch with an explicit **Copy** action — it does not overwrite
the clipboard automatically. Terminal is insertion-only (stage text; never execute).

Style modes (all inside the same mini response — no second provider handoff):

- **Literal** — wording preserved; only explicit spoken layout commands
- **Clean** — conservative punctuation / cleanup
- **Smart** — app-aware polish

---

## Screen & cursor

Screen context is **on demand**. The realtime model can request a fresh screenshot to
answer questions about visible UI. Coordinate-based actions always bind to that fresh
capture (multi-display actions include `display_id`). No ambient always-on recording.

Cursor control is a local macOS tool: move, click, double-click, right/middle click, drag,
scroll when the user asks Macky to operate visible UI. Standalone cursor actions invalidate
cached coordinates for the next action.

Focused-text tools let the model read/apply text in the frontmost editable context when
that is the right action — still local, still permission-gated by Accessibility.

---

## Background agents

Realtime remains the **sole** voice and conversation layer. Small and medium actions stay
in the realtime session. Genuinely long-running work — public-web research, multi-document
synthesis, artifact generation, local analysis — goes to Macky's built-in **General Agent**.

Properties:

- asynchronous; never blocks voice; collapsing the notch does not pause jobs
- at most three concurrent jobs; agents cannot spawn agents
- Mac sleep or app quit stops execution; queued/running work resumes on wake/relaunch
- Worker is a **stateless** Azure Responses boundary (`/agent-config`, `/agent-response`)
- canonical task state, continuation items, results, sources, artifacts, questions, and
  history stay **encrypted on the Mac**
- no Durable Object task DB, LiteLLM proxy, subscription, trial, or product quotas in this
  development shape

v1 agent execution tools only:

1. `read_attachment` — bounded chunk from an explicitly attached file  
2. `run_javascript` — sandboxed XPC (`MackyAgentExecutor`); no shell/Python/network/arbitrary FS  
3. `create_artifact`  
4. `ask_question` — expires after 24 hours  
5. `final_result` — spoken summary, Markdown, sources, artifacts, limitations, suggestions  

Structural guards (not product quotas): three active jobs, 5s JS timeout, attachment limits
(10 files / 50 MB), bounded reads, no-progress breaker.

### Agents in the notch

Agents page uses the existing design system with heavier scrolling. Square task cards for
active/recent work; tasks move to history after four hours; terminal history retained
locally for 30 days. Opening a card shows the typed thread (progress, tools, sources,
artifacts, questions, steering, cancel, restart, export, confirmed delete).

Agent composer attachments are **explicit agent attachments** — they do not route to the
general Files page. Background agents never become `voiceState`, `operationState`, or
realtime tool activity. When voice is idle, the closed notch may show a local
completion/needs-input notice.

Realtime bridge tools: `spawn_agents`, `list_agent_tasks`, `get_agent_result`,
`cancel_agent`, `open_agents_page`.

### Skills

Skills are immutable, user-configurable instruction packages — not agents and not
connectors. Built-ins are Macky-controlled. Users may create, AI-draft, duplicate,
enable/disable, and delete user Skills; a saved Skill is never edited in place. Enabled
Skill metadata is available to realtime; full instructions copy into a background task only
when explicitly attached. User Skill definitions are encrypted locally.

---

## Integrations

### The two buckets (shipped architecture)

**Web services** (Slack, Gmail, Spotify, GitHub, Notion, Linear, Google Calendar, …) go
through the **Composio MCP gateway**. One MCP server URL is registered on the realtime
session. Composio handles OAuth; the user connects each service once. The Worker creates
per-session Tool Router credentials (`/composio-config`) and connect links
(`/composio-connect`).

**macOS-native actions** (Apple Calendar, Apple Reminders, volume, DND, lock, app launch,
Chrome open/tab, screen capture, cursor, focused text, music controls, fast Spotify play)
are implemented as **Swift function tools** on the realtime session. There is **no LMCP**
and no second local MCP server in the current product.

> Older drafts described “LMCP with 221 tools” and “two MCP URLs.” That is **not** the
> shipped design. Do not reintroduce a local MCP mega-server unless the product explicitly
> decides to.

### Approvals

**Decision (2026-07-11):** Composio MCP tools use `require_approval: "never"` for normal
clear commands. Macky does not insert a per-action confirm UI for routine messages,
calendar/reminders, music, or system controls — that would break the half-second loop. The
model confirms ordinary actions after success.

Narrow exception: permanent deletion/discard, payments/purchases/transfers, cancellations,
account/security changes, or comparable irreversible harm. The model names the consequence
and gets an explicit **voice** confirmation before calling that tool. Conversational, not a
new gating UI. `AssistantOperationState.awaitingApproval` stays retired.

### Fast Spotify path

Named-track playback uses a dedicated Worker route (`POST /spotify-play`) behind the
native `play_spotify_track` tool so search→play does not stall in MCP discovery.

### v1 connector set (Composio catalog on client)

Gmail, Slack, Google Calendar, Notion, GitHub, Linear, Spotify — see `ConnectorRegistry`.

Native v1 coverage emphasizes: Spotify/music, Apple Calendar & Reminders, Slack & Gmail
(via Composio), Chrome open/search, system controls, screen + cursor when asked.

---

## Auth and onboarding

First launch expands the notch panel for a clean setup / magic-link sign-in. Users can
**Skip for now** during early testing. The app still bootstraps an anonymous Worker session
so connectors can work before email login.

Signing in with email replaces the anonymous session identity; connected Composio accounts
from the anonymous identity are **not** migrated. Token refresh / session persistence is
Keychain-backed (`macky.session`). Deep links: `Macky://auth`, `Macky://connected`.

macOS permissions (Microphone, Accessibility, Screen Recording, Calendar, Reminders) go
through standard Apple dialogs. Cloud OAuth goes through Composio Connect links from the
Connectors tab or mid-turn connect prompts.

---

## The Cloudflare Worker

All privileged calls go through the Worker (`realtime-proxy`). The Swift app never talks to
Azure or Composio with a raw vendor key.

| Route | Role |
|-------|------|
| `GET /realtime` | Byte-proxy WebSocket → `gpt-realtime-2.1` |
| `GET /dictation/realtime` | Short-lived text-only → `gpt-realtime-2.1-mini` |
| `GET /agent-config` / `POST /agent-response` | Stateless General Agent boundary → `gpt-5.6-sol` |
| `/composio-config` `/composio-connect` `/composio-connections` | Composio session + OAuth |
| `POST /spotify-play` | Direct Spotify search→play |
| `/auth/*` | Anonymous session, magic link, verify, HTML bridges |

Hosted default host is `realtime-proxy.winky-secrets.workers.dev`, defined once in
`WorkerEndpoints.baseHost`. Self-hosters change that one constant after deploying `worker/`.

Pure WebSocket proxying matters: Cloudflare CPU limits would bite a compute-heavy
persistent socket; byte forwarding does not.

---

## What we're not building in v1

- No ambient always-on screen recording  
- No ambient assistant memory across sessions  
- No local search engine over general activity / “second brain”  
- Encrypted background-agent task records are the narrow history exception (4h recent /
  30d terminal retention)

v1 is: voice in, action out, fast, transparent enough that you trust it, top integrations
people actually use. Get that right first.

---

## The competition (context)

- **heyclicky** — closest concept; hand-built connectors; older voice chain. We are
  MCP-native for cloud apps and realtime-first.
- **Perplexity Personal Computer** — more local-file / long-horizon angled.
- **Raycast AI** — keyboard-first, different user.
- **Apple Intelligence** — narrower, slower expansion cadence.

Our bet: realtime latency + MCP cloud integrations + native Mac tools + notch UI that
stays out of the way + background agents for the long jobs. None of the others combine all
of that cleanly.

---

## The codebase (orientation)

| Path | Role |
|------|------|
| `leanring-buddy/` | Macky macOS app (legacy folder name — do not rename) |
| `leanring-buddy.xcodeproj/` | Xcode project; product is `Macky.app` |
| `MackyAgentExecutor/` | Sandboxed XPC JS for General Agent |
| `worker/` | Cloudflare Worker |
| `scripts/` | Production release pipeline |
| `AGENTS.md` + nested `*/AGENTS.md` | Agent operating manuals |

Kept from Clicky: NSPanel notch shell, ScreenCaptureKit, CGEvent tap, design tokens.  
Removed: AssemblyAI, ElevenLabs, Claude chain, any assumption of LMCP.

Full agent instructions: start at [`AGENTS.md`](AGENTS.md).
