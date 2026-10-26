# Information Overlord — Plan

## Problem

I use several AI agents: Hermes (self-hosted, persistent), OpenAI dots, Meta
Muse, and Claude Code. Each one connects to my personal data (email, calendar,
files, memory) in its own way, and each logs that access differently or not at
all. I can't answer basic questions such as:

- Which agents read my inbox this week, and which messages did they read?
- Which skill wrote this memory, and what source data was it based on?
- Which agents have seen that memory since?

## Approach

Audit access where it happens, not inside each skill. No agent holds my real
credentials. Every agent reaches my data through one gateway that I run, and
the gateway records every call. Its log is the same no matter which harness
made the call, including harnesses I didn't write and can't instrument.

```
 Hermes ─┐
 Claude ─┤  MCP (Streamable HTTP)   ┌──────────────────────┐   Google APIs
 dots   ─┼─────────────────────────▶│  information-overlord │──────────────▶ Gmail, Calendar, Drive
 Muse   ─┘  one identity per agent  │  (Railway)            │
                                    │  • auth & policy      │──▶ Postgres: audit log,
                                    │  • audit log          │     memories, approvals
                                    │  • approvals          │
                                    │  • memory w/ provenance│
                                    └──────────────────────┘
```

## Goals

1. **One audit log** for every agent's access to personal data: who, what,
   when, which session or skill, and which resource IDs were touched.
2. **Least privilege.** Each agent has its own identity and a tool allowlist.
   Writes (sending, deleting, labeling) can require my approval.
3. **Memory provenance.** Every memory records where it came from (agent,
   skill, run, source resource IDs), and every memory read is logged.
4. **Works with hosted agents.** The gateway has a public HTTPS endpoint, so
   dots, Muse and claude.ai can reach it from their vendors' clouds.

## Non-goals

- Seeing what an agent does with data *after* it reads it. The log records
  access, not what happens downstream in a vendor's cloud.
- Preventing prompt injection inside content. Approvals on writes are the
  mitigation.
- Multi-user or multi-tenant support. This is for one person.
- Replacing each harness's own tracing. Those can be joined in later (stage 5).

## Client compatibility (researched 2026-10-24, not yet verified)

| Client | Reaches gateway from | Auth it supports | Status |
|---|---|---|---|
| Hermes | My host | Bearer key (any) | Fully under my control |
| Claude Code (local) | My machine | Bearer header or OAuth | Documented |
| claude.ai / Claude cloud | Anthropic cloud | OAuth 2.1 (DCR + PKCE) | Documented |
| OpenAI dots | OpenAI cloud | OAuth only; dev-mode plugin at chatgpt.com/plugins | **Unconfirmed** that dots use dev-mode plugins; Pro may be read-only |
| Meta Muse | Meta cloud | Bearer key via secure credential prompt works best; OAuth unreliable | **Unconfirmed**: added through a chat-built "custom connector", wrapped as a Muse skill |

So the gateway must support **both** per-agent bearer keys **and** OAuth 2.1.

## Architecture

### Stack (proposed)

- **TypeScript on Node** with the official MCP SDK (`@modelcontextprotocol/sdk`),
  Streamable HTTP transport.
- **Postgres** on Railway.
- **Railway** for hosting, deployed from GitHub pushes to `main`.
- Version in `package.json`, shown on the dashboard and returned by a `whoami`
  tool.

### Components

1. **Transport and auth layer.** Resolves each request to an `agent_id`.
   - Bearer keys: random keys whose hashes are stored, one per agent, revocable.
   - OAuth 2.1 (stage 3): either a well-tested library or a hosted identity
     service that supports MCP dynamic client registration (WorkOS AuthKit,
     Stytch, Auth0). Decide when we get to stage 3.
2. **Policy.** Per-agent tool allowlist, plus a per-tool mode: `allow`,
   `approve` (wait for my approval) or `deny`.
3. **Audit middleware.** Wraps every tool call, writes an event before and
   after, and logs denied calls too.
4. **Tool modules**, one per upstream service:
   - `gmail`: search, get message or thread metadata, get body. Later: label,
     draft, send (send requires approval).
   - `calendar`: list and search events. Later: create or respond.
   - `memory`: write, search and read memories with provenance.
   - `meta`: `whoami` (agent identity and gateway version).
5. **Upstream credentials.** One Google OAuth refresh token for my account,
   encrypted at rest with a key held in a Railway variable.
6. **Approvals.** A pending-request queue, a push notification to me, and an
   approve/deny page. The agent's call blocks with a timeout, or returns
   "pending" and the agent polls.
7. **Dashboard.** An authenticated web page: filter the audit log by agent,
   tool, resource and time; open a memory and see its provenance and read
   history.

### Data model (first draft)

```sql
agents(
  id text primary key,          -- 'hermes', 'claude-code', 'dots', 'muse'
  display_name text,
  auth_kind text,               -- 'bearer' | 'oauth'
  key_hash text,                -- bearer only
  created_at timestamptz, revoked_at timestamptz
)

audit_events(                   -- append-only: the app role gets INSERT/SELECT only
  id bigserial primary key,
  at timestamptz not null default now(),
  agent_id text not null,
  session_id text,              -- from the client if it sends one
  skill text,                   -- self-reported by the agent (optional tool arg)
  tool text not null,
  decision text not null,       -- 'allowed' | 'denied' | 'approved' | 'rejected'
  args jsonb,                   -- argument values hidden unless allowlisted
  resource_ids text[],          -- e.g. gmail:msg/18c…, memory:42
  result_hash text,
  error text,
  duration_ms int
)

memories(
  id bigserial primary key,
  created_at timestamptz,
  created_by_agent text,
  created_by_skill text,
  run_id text,
  content text,
  tags text[],
  source_ids text[],            -- resource IDs this memory was based on
  derived_from bigint[],        -- other memory IDs (summaries of summaries)
  deleted_at timestamptz
)

memory_reads(memory_id bigint, audit_event_id bigint)

approvals(
  id bigserial primary key,
  audit_event_id bigint,
  requested_at timestamptz, decided_at timestamptz,
  decision text, decided_via text
)
```

Logging rules: store resource IDs and hashes by default, **not message
bodies or memory content**. The log should not become a second copy of my
inbox.

### Self-reported context

The gateway can't verify which *skill* inside an agent made a call. Tools
accept an optional `skill` / `purpose` argument, and Hermes skills are
expected to fill it in. It's recorded as self-reported. The agent's identity
is always authenticated.

## Stages

### Stage 1: Gmail read and audit log (target: a weekend)
- Repo scaffold, `package.json` at `0.1.0`, Railway service and Postgres.
- Bearer-key auth; CLI script to issue and revoke agent keys.
- Gmail read tools; Google OAuth bootstrap script for my account.
- Append-only `audit_events`; minimal dashboard listing events.
- Connect **Hermes** and **local Claude Code**.
- **Done when:** I can open the dashboard and see every Gmail message an agent
  read, by agent, with timestamps.

### Stage 2: Hosted bearer client and approvals
- Connect **Muse** with a bearer key (verify the custom-connector flow).
- Policy table and per-tool modes; approval queue with push notification.
- First write tool: `gmail.create_draft` (allowed); `gmail.send` (approval
  required).
- **Done when:** Muse can read mail through the gateway, and a send request
  waits for my approval.

### Stage 3: OAuth 2.1
- Protected-resource and authorization-server metadata, dynamic client
  registration, PKCE, refresh.
- Connect **claude.ai** and **dots**. Verify the plan limits and dev-mode
  plugin support on dots.
- **Done when:** both connect and their calls show up under their own agent
  identity.

### Stage 4: Memory with provenance
- `memory.write / search / read` tools, with `source_ids`, `derived_from` and
  read tracking.
- Dashboard memory view: provenance chain and read history.
- Migrate Hermes's memory (or a slice of it) to the gateway.
- **Done when:** for any memory I can see where it came from and which agents
  have read it.

### Stage 5: Join in harness traces (optional)
- Collect OpenTelemetry from Claude Code (`CLAUDE_CODE_ENABLE_TELEMETRY`) and
  Hermes, keyed by session ID, so gateway events can be joined with the
  agent's reasoning trace.

## Gotchas to remember

- **Google OAuth app mode.** Gmail read is a restricted scope. In *testing*
  mode refresh tokens expire after 7 days. Run the app in *production,
  unverified* (personal use; one warning screen). Check Google's current rules
  before relying on this.
- **Opaque hosted clients.** Log full request and response headers (with
  secrets hidden) on the auth endpoints. It's the only way to debug dots and
  Muse when they fail to connect.
- **Turn off direct connectors.** The claude.ai Gmail, Calendar and Drive
  connectors go straight to Google and bypass the gateway. Turn them off once
  the gateway covers the same tools.
- **High-value target.** Encrypt stored tokens, keep scopes narrow, make agent
  keys easy to rotate, and require a separate login for the dashboard.

## Open questions

1. Does the Hermes skills repo already have a manifest or permissions format
   that tool allowlists should reuse?
2. Should approvals block the call (with a timeout) or return "pending"? It
   depends on what each client tolerates.
3. How should I log in to the dashboard: Google sign-in limited to my account,
   or a passkey?
4. OAuth in stage 3: write it with a library, or use a hosted identity service?
5. How long to keep audit events: forever, or roll up after N months?

## Alternatives considered

Hosted gateways (Zapier MCP, Composio, Arcade, Pipedream Connect) and
open-source ones (Obot, Metorial, IBM ContextForge, small Go gateways). I
chose to build my own because the hosted ones put a third party in the data
path and keep the log themselves, and none of them handle memory provenance.
Obot is the fallback if building the gateway layer turns out to be too much
work.
