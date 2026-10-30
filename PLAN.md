# Information Overlord — Plan

## Purpose

**An audit trail of AI agents' access to my personal data.**

I use several AI agents: Hermes (self-hosted, persistent), OpenAI dots, Meta
Muse, and Claude Code. Each one connects to my email, calendar, files and
memory in its own way, and each logs that access differently or not at all. I
want to be able to answer:

- Which agents read my inbox this week, and exactly what did they see?
- Who touched this message, event, file or memory, and when?
- Did any agent's access pattern change: more volume, new sources, odd hours?
- What did an agent send, write or delete on my behalf?

## Method: a logging proxy and a traffic analyzer

No agent holds my real credentials. Every agent reaches my data through a
proxy that I run. The proxy passes each request to the real MCP server or
API and records the full request and response. A traffic analyzer turns that
record into an audit trail I can search.

```
                         ┌────────────────────────────────┐
 Hermes ─┐  MCP          │  information-overlord (Railway) │      upstream MCP servers
 Claude ─┤ ─────────────▶│                                │────▶ (Gmail, Calendar, Drive,
 dots   ─┤               │  proxy:  identify agent        │       memory, …)
 Muse   ─┤  HTTP API     │          apply rules           │
 scripts ┘ ─────────────▶│          add real credentials  │────▶ REST APIs
           agent key     │          record full traffic   │      (Google, …)
           only          │                                │
                         │  analyzer: parse resource IDs  │
                         │            search · dashboard  │
                         │            alerts              │
                         └───────┬───────────────┬────────┘
                                 │               │
                        Postgres: events   Object storage: payloads
                        (kept forever)     (encrypted, 90-day rolling)
```

The proxy doesn't implement Gmail or Calendar itself. Adding a data source
means configuring an upstream, not writing tools.

### What makes the trail complete

The proxy only sees traffic routed through it. It can't intercept a vendor's
own HTTPS connections to Google. Coverage is complete only when both of these
hold:

1. **The proxy holds the real credentials.** Agents only get revocable proxy
   keys.
2. **Direct connectors are turned off.** That includes the claude.ai Gmail,
   Calendar and Drive connectors and Muse's built-in Google connector.

The dashboard keeps a **coverage list** (which agents and which sources go
through the proxy, and which known direct paths are still open), so gaps are
visible and not just assumed away.

## Goals

1. **Full-fidelity record.** Every call is recorded: agent, session,
   self-reported skill, upstream, tool or endpoint, full request, full
   response, status, timing. Denied and failed calls are recorded too.
2. **Search by resource.** Parse responses to pull out resource IDs (Gmail
   message or thread, calendar event, Drive file, memory), so I can ask "who
   touched X?" and not just "who called URL Y?".
3. **Tamper evidence.** The log can only be added to, and each event is
   hash-chained to the one before it, so an agent (or anyone who compromises
   the proxy's app role) can't quietly rewrite history.
4. **Works with hosted agents.** The proxy has a public HTTPS endpoint, so
   dots, Muse and claude.ai can reach it from their vendors' clouds.
5. **Supports both styles of agent.** MCP tool calls, *and* agent-written
   scripts calling a plain HTTP API. Scripts make fewer model round trips and
   send less data through the model, and the audit trail is the same for both.
6. **Optional controls.** Per-agent allowlists and approval of sensitive
   writes. These are secondary to the audit trail but come cheap once all
   traffic passes through one place.

## Non-goals

- Seeing what an agent does with data *after* it reads it inside a vendor's
  cloud. The trail records access.
- Preventing prompt injection inside content. Approvals on writes are the
  mitigation.
- Multi-user or multi-tenant support. This is for one person.
- Transparent interception of traffic the agents aren't configured to send to
  the proxy.

## Client compatibility (researched 2026-10-24, not yet verified)

| Client | Reaches proxy from | Auth it supports | Status |
|---|---|---|---|
| Hermes | My host | Bearer key | Fully under my control |
| Claude Code (local) | My machine | Bearer header or OAuth; can also write scripts against the HTTP API | Documented |
| claude.ai / Claude cloud | Anthropic cloud | OAuth 2.1 (DCR + PKCE) | Documented |
| OpenAI dots | OpenAI cloud | OAuth only; dev-mode plugin at chatgpt.com/plugins | **Unconfirmed** that dots use dev-mode plugins; Pro may be read-only |
| Meta Muse | Meta cloud | Bearer key via secure credential prompt works best; OAuth unreliable | **Unconfirmed**: added through a chat-built "custom connector" |

So the proxy must accept **both** per-agent bearer keys **and** OAuth 2.1.

## Architecture

### Stack (proposed)

- **TypeScript on Node** with the official MCP SDK (`@modelcontextprotocol/sdk`)
  for both the server side (facing agents) and the client side (facing
  upstreams). Streamable HTTP transport.
- **Postgres** on Railway for event metadata, resource index and config.
- **S3-compatible object storage** (a Railway bucket) for payloads,
  compressed and encrypted per object.
- **Railway** hosting, deployed from GitHub pushes to `main`.
- Version in `package.json`, shown in the dashboard header and returned by a
  `whoami` tool.

### Components

1. **Agent-facing front doors** (both share auth, rules and recording):
   - **MCP endpoint:** `/mcp`. The tools are the union of the upstream MCP
     servers' tools, with namespaced names (`gmail.search`, `memory.read`).
     Tool descriptions are passed through and kept short, to limit context
     cost.
   - **HTTP API proxy:** `/api/<upstream>/...`, e.g.
     `/api/google/gmail/v1/users/me/messages`. It forwards with the real
     credentials added. `/api/openapi.json` lists what's available, so
     agents can write scripts against it.
2. **Identity.** Each request resolves to an `agent_id`.
   - Bearer keys: random, stored hashed, one per agent, revocable.
   - OAuth 2.1 for dots and claude.ai: a well-tested library or a hosted
     identity service that supports MCP dynamic client registration (WorkOS
     AuthKit, Stytch, Auth0). Decide in stage 4.
   - Optional `X-Agent-Session` and `X-Agent-Skill` headers, or matching MCP
     metadata, recorded as **self-reported**. Hermes skills are expected to
     set them.
3. **Upstream registry.** Configuration for each upstream: kind (MCP or
   HTTP), URL or command, how its credentials are added, and which parser to
   use for its traffic.
4. **Credential vault.** Upstream secrets (the Google refresh token, API
   keys), encrypted at rest with a key held in a Railway variable, refreshed
   automatically. Agents never see them.
5. **Recorder.** Writes the event row before forwarding (so failed and
   timed-out calls are still captured), then completes it with the response.
   Payloads go to object storage, gzipped and encrypted, addressed by
   event ID.
6. **Parsers.** Small, per-upstream modules that turn request and response
   into resource references (`gmail:msg/<id>`, `gmail:thread/<id>`,
   `gcal:event/<id>`, `drive:file/<id>`, `memory:<id>`) and an access kind
   (`list`, `read-metadata`, `read-content`, `write`, `delete`, `send`).
   Unknown traffic is still recorded, just not indexed by resource.
7. **Rules.** Per-agent allowlist by upstream and tool or endpoint, plus a
   mode for each: `allow` (default), `approve`, or `deny`. An approval returns
   "pending" immediately and doesn't block, so scripts don't hang. The agent
   retries with the approval ID.
8. **Analyzer and dashboard.** An authenticated web UI:
   - Timeline filtered by agent, upstream, tool, resource and time.
   - A page per resource: every access to one message, event or file.
   - A page per agent: volume over time, the sources it touches, the tools it
     uses.
   - The full request and response for any event still inside the payload
     window.
   - The coverage list (see above).
   - Search over payload content within the window.
9. **Alerts.** Simple rules over the event stream: volume spikes compared with
   an agent's baseline, first access to a new upstream or tool, any `send` or
   `delete`, access at unusual hours. Delivered by push notification.
10. **Retention job.** Each day, delete payload objects older than **90 days**
    and mark their events `payload_pruned`. Event metadata, resource
    references and payload hashes are kept forever.

### Data model (first draft)

```sql
agents(
  id text primary key,            -- 'hermes', 'claude-code', 'dots', 'muse'
  display_name text,
  auth_kind text,                 -- 'bearer' | 'oauth'
  key_hash text,
  created_at timestamptz, revoked_at timestamptz
)

upstreams(
  id text primary key,            -- 'gmail', 'gcal', 'google-api', 'memory'
  kind text,                      -- 'mcp' | 'http'
  target text,                    -- URL or launch command
  credential_ref text,
  parser text
)

events(                           -- append-only: the app role has INSERT/SELECT only;
  id bigserial primary key,       --   completion goes to event_results
  at timestamptz not null default now(),
  agent_id text not null,
  session_id text,                -- self-reported
  skill text,                     -- self-reported
  front_door text,                -- 'mcp' | 'http'
  upstream_id text,
  operation text,                 -- tool name, or 'GET /gmail/v1/…'
  decision text,                  -- 'allowed' | 'denied' | 'pending' | 'approved' | 'rejected'
  request_hash text,
  prev_hash text,                 -- hash chain for tamper evidence
  chain_hash text
)

event_results(
  event_id bigint primary key,
  completed_at timestamptz,
  status text,                    -- 'ok' | 'error' | 'timeout'
  error text,
  duration_ms int,
  response_hash text,
  response_bytes int,
  payload_key text,               -- object-storage key; null once pruned
  payload_pruned_at timestamptz
)

resource_refs(                    -- the "who touched X" index
  event_id bigint,
  resource text,                  -- 'gmail:msg/18c…'
  access text                     -- 'list' | 'read-metadata' | 'read-content' | 'write' | 'delete' | 'send'
)

approvals(
  id bigserial primary key,
  event_id bigint,
  requested_at timestamptz, decided_at timestamptz,
  decision text, decided_via text
)

alerts(id bigserial primary key, at timestamptz, rule text, agent_id text, event_ids bigint[], acknowledged_at timestamptz)
```

### Payload storage

- **Content:** the full request and response bodies plus headers, with
  credential headers removed.
- **Encryption:** each payload is gzipped, then encrypted with AES-GCM under a
  data key held in a Railway variable. This protects against a leaked bucket
  or database backup, not against a compromised proxy process. A later
  option: encrypt payloads to a public key whose private key lives only in
  the browser I use for the dashboard.
- **Retention:** 90 days rolling, pruned daily. Hashes stay forever, so a
  payload I've exported can still be verified against the trail later.
- **Size check:** occasionally an agent will read thousands of messages. Track
  bytes per day on the dashboard so storage cost stays visible.

### Data sent through MCP vs. scripts

MCP tool calls send every intermediate result through the model, and each
call is a model round trip. Agents that can run code are better off writing a
script against the HTTP API. The script runs once, filters on the proxy side,
and only its output reaches the model. That's cheaper, faster, and **sends
less of my data to the vendor's model**. The audit trail is identical either
way, because every call still crosses the proxy. Tool design should follow
from this: search returns IDs and snippets, and full content is a separate,
explicit read.

## Stages

### Stage 1: MCP proxy and recorder (target: a weekend)
- Repo scaffold, `package.json` at `0.1.0`, Railway service, Postgres and a
  bucket.
- Bearer-key agents; a CLI to issue and revoke keys.
- An MCP proxy in front of **one existing Gmail MCP server**, with the Google
  credentials held by the proxy.
- The recorder, with full payloads, the hash chain, and the append-only role.
- A minimal dashboard: event timeline plus event detail with payload.
- Connect **Hermes** and **local Claude Code**.
- **Done when:** every Gmail access by either agent shows up with its full
  request and response.

### Stage 2: HTTP API proxy and parsers
- `/api/google/...` passthrough with token injection, and `openapi.json`.
- Parsers for Gmail, Calendar and Drive, filling `resource_refs`.
- Resource page ("who touched X") and agent page.
- **Done when:** a Claude Code script reading Gmail through the HTTP API and
  an MCP tool call for the same message both show up on that message's page.

### Stage 3: Analyzer, retention and coverage
- Retention job (90 days), storage gauge.
- Alert rules and push notifications.
- Coverage list.
- Connect **Muse** with a bearer key (verify the custom-connector flow).
- **Done when:** an unusual spike in reads triggers an alert, and payloads
  older than 90 days are gone while their events remain.

### Stage 4: OAuth 2.1 and rules
- Protected-resource and authorization-server metadata, dynamic client
  registration, PKCE, refresh.
- Connect **claude.ai** and **dots**. Verify the plan limits and dev-mode
  plugin support on dots.
- Allowlists and approvals (`pending`, then retry with the approval ID).
- Turn off direct Google connectors in every client.
- **Done when:** all four agents go through the proxy and the coverage list
  shows no known direct paths.

### Stage 5: Memory as an upstream
- A memory MCP server that records where each memory came from (writing
  agent and skill, source resource IDs, derived-from memories). The proxy
  supplies "who read it".
- Move Hermes's memory (or a slice of it) behind the proxy.

### Stage 6: Join in harness traces (optional)
- Collect OpenTelemetry from Claude Code (`CLAUDE_CODE_ENABLE_TELEMETRY`) and
  Hermes, keyed by session ID, to show *why* next to *what*.

## Gotchas to remember

- **Google OAuth app mode.** Gmail read is a restricted scope. In *testing*
  mode refresh tokens expire after 7 days. Run the app in *production,
  unverified* (personal use; one warning screen). Check Google's current rules
  before relying on this.
- **Opaque hosted clients.** Record full HTTP exchanges on the auth
  endpoints too. It's the only way to debug dots and Muse when they fail to
  connect.
- **High-value target.** The proxy holds my Google credentials *and* 90 days
  of everything agents read from my inbox, on a public URL. Encrypt stored
  tokens and payloads, keep upstream scopes narrow, make agent keys easy to
  rotate, require a separate strong login (passkey) for the dashboard, and
  never return payloads through the agent-facing endpoints.
- **Streaming and large responses.** Record payloads by streaming to object
  storage, not by holding them in memory. Cap payload size per event and
  mark the event when a payload is truncated.

## Open questions

1. Does the Hermes skills repo already have a manifest or permissions format
   that agent allowlists should reuse?
2. Which existing Gmail MCP server to put behind the proxy in stage 1, or
   should stage 1 go straight to the HTTP proxy for Google?
3. Dashboard login: a passkey, or Google sign-in limited to my account?
4. OAuth in stage 4: write it with a library, or use a hosted identity
   service?
5. Should payload encryption move to a browser-held key (so a compromised
   proxy can't read history)?

## Alternatives considered

Hosted gateways (Zapier MCP, Composio, Arcade, Pipedream Connect) and
open-source ones (Obot, Metorial, IBM ContextForge, small Go gateways). Their
focus is *access management*. Logging is a secondary feature, usually
metadata-only, and the hosted ones put a third party in the data path and keep
the record themselves. This project's focus is *the audit trail itself*: full
payloads, search by resource, tamper evidence, and analysis, under my own
control. Obot is the fallback if the proxy layer turns out to be too much
work.
