# AGENTS.md — worker (Cloudflare Worker proxy)

Operating manual for the Cloudflare Worker. Root [`AGENTS.md`](../AGENTS.md) and
[`MACKY.md`](../MACKY.md) still apply. Swift client URLs:
`leanring-buddy/Networking/WorkerEndpoints.swift`. App guide:
[`../leanring-buddy/AGENTS.md`](../leanring-buddy/AGENTS.md).

---

## 1. Purpose

The Worker (`realtime-proxy`) is the **network boundary** between the macOS app and
external services. It keeps every secret out of the app bundle. The app never talks to
Azure or Composio with a vendor key.

`/realtime` is a **byte-forwarding WebSocket proxy**: app ↔ Worker ↔ Azure realtime.
Bytes flow without per-message compute. That is deliberate — a persistent socket doing
real work would hit Cloudflare CPU-time limits; pure proxying does not.

Default public origin (also `PUBLIC_BASE_URL` / Swift `WorkerEndpoints.baseHost`):
`https://realtime-proxy.winky-secrets.workers.dev`.

---

## 2. Routes (`src/index.ts`)

| Method & path | What it does |
|---------------|--------------|
| `GET /realtime` | WebSocket upgrade. Proxies bytes to Azure Foundry realtime (`…/openai/v1/realtime?model=gpt-realtime-2.1`) using `AZURE_OPENAI_API_KEY`. Both sockets torn down as a pair. |
| `GET /agent-config` | `Authorization: Bearer <sessionToken>`. Development-only General Agent protocol-v1 capability document + kill switch. Advertises `sol-medium`, operations `general` \| `skill-draft`, optional native web search, five fixed local tool names. |
| `POST /agent-response` | Bearer auth. Strict protocol-v1 request → stateless Azure Responses (`gpt-5.6-sol`, medium reasoning, `store: false`, encrypted reasoning continuation, five strict function schemas, optional web search). Azure SSE normalized to Macky protocol-v1; raw provider events/errors never reach the app. |
| `GET /composio-config` | Bearer auth. Composio Tool Router session for `composioUserId` → `{ url, key }` for realtime MCP. No toolkit allowlist (search catalog); `manage_connections` on with `enable_wait_for_connections: false`. |
| `POST /composio-connect` | Bearer auth. Body/query `{ toolkit }`. Hosted connect `link` with `callback_url` → `/auth/connected`. Returns `{ toolkit, redirect_url }`. |
| `GET /composio-connections` | Bearer auth. ACTIVE connected accounts → `{ connected: ["gmail", …] }`. |
| `POST /spotify-play` | Bearer auth. Body `{ query, uri? }`. Server-side SPOTIFY_SEARCH → START_RESUME (or TRANSFER) via Composio REST execute — not MCP. Powers native `play_spotify_track`. May return `{ needs_device: true, … }`. |
| `GET /dictation/realtime` | Bearer auth + WebSocket. App sends `dictation.start` (surface kind, mode, ≤100 keyterms); Worker opens `gpt-realtime-2.1-mini` text-only 24 kHz session (no tools/tracing). Accepts bounded `dictation.audio` + one `dictation.commit`. Never logs transcripts, AX metadata, or keyterms. |
| `POST /auth/magic-link` | `{ email }` → one-time token in `AUTH_TOKENS` (15 min TTL) + Resend email to https `/auth/open`. |
| `POST /auth/verify` | Consume token; provision Composio user (best-effort); create `SESSIONS` with `composioUserId = email`; return `{ sessionToken, composioUserId }`. **Replaces** prior anonymous identity (no connection migration). |
| `POST /auth/anonymous` | Mints `composioUserId = "anon-<uuid>"` + `SESSIONS`; same response shape as verify. First-run / no-session bootstrap. |
| `GET /auth/open?token=…` | HTML bridge → `Macky://auth?token=…`. |
| `GET /auth/connected?toolkit=…` | HTML bridge → `Macky://connected?toolkit=…`. |

Legacy Durable Object classes may still exist in `wrangler.toml` for compatibility; they are
**not** part of the product router above. Do not build new features on them.

---

## 3. Files

- `src/index.ts` — all route handlers + exported validation/payload helpers for tests.
- `test/agent-validation.test.ts` — General Agent contract + Azure payload tests.
- `test/dictation-validation.test.ts` — dictation WebSocket / session config tests.
- `wrangler.toml` — name `realtime-proxy`, entry, compatibility date, KV bindings,
  `MAGIC_LINK_FROM`, `PUBLIC_BASE_URL`.
- `.wrangler/` — local Wrangler cache. Do not commit secrets from local state.

**No `package.json`.** Do not add an npm manifest unless explicitly asked (see §6).

---

## 4. Bindings, Secrets & Vars

**Secrets** (`npx wrangler secret put <NAME>` — never in source):

- `AZURE_OPENAI_API_KEY` — `/realtime`, `/dictation/realtime`, `/agent-response`
- `COMPOSIO_API_KEY` — Composio routes + user provisioning
- `RESEND_API_KEY` — magic-link email

**KV**

- `AUTH_TOKENS` — pending magic-link tokens (`token → email`), TTL via `expirationTtl`
- `SESSIONS` — `sessionToken → { composioUserId, kind, email? }` JSON, no TTL. Created only
  by `/auth/anonymous` and `/auth/verify`. Every Composio-facing route resolves identity
  via `Authorization: Bearer` + `resolveSession()` — never a fixed shared user id.

**Vars (`wrangler.toml`)**

- `MAGIC_LINK_FROM` — default `onboarding@resend.dev` (Resend signup-address only until a
  verified domain is configured)
- `PUBLIC_BASE_URL` — public Worker origin for email links

---

## 5. Invariants (do not break)

- Keep `/realtime` a pure byte-forwarding proxy — no payload transform/logging/compute.
- Validate JSON route methods and shapes; reflect only validated values.
- Magic-link tokens are **single-use** and expire (15 min).
- Resolve `composioUserId` only from `SESSIONS` — never trust client-supplied identity.
- `/auth/anonymous` and `/auth/verify` are the only `SESSIONS` creators; same response
  shape. Email login does not merge anonymous Composio connections.
- No durable Worker state except KV. No module-level session stores.
- Composio provisioning is best-effort — must not block login/bootstrap.
- Dictation stays a separate authenticated socket; Worker owns Azure session config; no
  client event may enable tools or audio out.
- General Agent API stays development-oriented and **stateless**: no KV/DO/quota/trial
  task store. `/agent-config` kill switch must match `/agent-response` enforcement.
- Agent requests: only `protocol_version: 1`, `agent: "general"`, bounded `input`,
  optional `operation`, `web_search`, allow-listed `continuation_items`, `tool_outputs`.
  Reject unknown fields and mismatched/duplicate `call_id`s.
- Worker owns Azure model, medium reasoning, safety instructions, `store: false`,
  streaming, encrypted reasoning inclusion, disabled parallel tool calls, and strict
  schemas for `read_attachment`, `run_javascript`, `create_artifact`, `ask_question`,
  `final_result`. Never accept caller tool defs, remote MCP, or provider overrides.
- Preserve continuation order; insert each `function_call_output` immediately after its
  `function_call`. `final_result` is the only completion signal.
- SSE normalizer emits only protocol-v1 Macky events. Do not log prompts, responses, tool
  data, Azure error bodies, or tokens.

---

## 6. Ask Before

- Adding/renaming routes (including new General Agent routes beyond the two above)
- Changing Azure realtime URL, `model` query param, or auth header
- Changing Composio session payload / returned `{ url, key }` shape
- Changing `SESSIONS` creation/resolution or `{ sessionToken, composioUserId }` shape
- Changing magic-link content or `/auth/open` / `/auth/connected` bridges
- Adding npm dependencies or a package manifest
- Replacing KV with another persistence mechanism
- Multi-tenant hardening (rate limits, stronger session binding) — prerequisite before
  opening the Worker to many independent users

---

## 7. Validation

- Static first: read `src/index.ts`, `rg` route names in the Swift app, confirm
  `WorkerEndpoints` agreement.
- Local: `npx wrangler dev` (secrets required for real Azure/Composio).
- Deploy only when asked: `npx wrangler deploy`.
- Tests:
  - `node --experimental-transform-types --test test/dictation-validation.test.ts`
  - `node --experimental-transform-types --test test/agent-validation.test.ts`

---

## User Instructions

### One-time setup

1. `npx wrangler login`
2. Create KV namespaces and paste ids into `wrangler.toml`:
   ```bash
   npx wrangler kv namespace create AUTH_TOKENS
   npx wrangler kv namespace create AUTH_TOKENS --preview
   npx wrangler kv namespace create SESSIONS
   npx wrangler kv namespace create SESSIONS --preview
   ```
3. Resend account + API key.
4. Composio project API key; create auth configs for each
   `ConnectorRegistry` slug (`gmail`, `slack`, `googlecalendar`, `notion`, `github`,
   `linear`, `spotify`) or `/composio-connect` 404s.
5. Secrets:
   ```bash
   npx wrangler secret put AZURE_OPENAI_API_KEY
   npx wrangler secret put COMPOSIO_API_KEY
   npx wrangler secret put RESEND_API_KEY
   ```

### Develop & deploy

- Local: `npx wrangler dev` (from `worker/`)
- Deploy: `npx wrangler deploy`
- Logs: `npx wrangler tail`

### Auth smoke test

`POST /auth/magic-link` with `{ "email": "you@example.com" }` → open link →
`Macky://auth?token=…` → app `POST /auth/verify`.

### Pointing the app at your Worker

Set `WorkerEndpoints.baseHost` in the Swift app to your Worker host (no scheme). That is
the only Swift host constant.
