# information-overlord — Plan

## Purpose

**An audit trail of AI agents' access to people's personal data.**

People now use several AI agents: hosted ones (Claude, ChatGPT agents such as
dots, Meta Muse) and self-hosted harnesses (Hermes, Claude Code). Each one
connects to email, calendar, files and memory in its own way, and each logs
that access differently or not at all. A user should be able to answer:

- Which agents read my inbox this week, and exactly what did they see?
- Who touched this message, event, file or memory, and when?
- Did any agent's access pattern change: more volume, new sources, odd hours?
- What did an agent send, write or delete on my behalf?

## Method: a logging proxy and a traffic analyzer

No agent holds the user's real credentials. Every agent reaches the user's
data through the proxy. The proxy passes each request to the real MCP server
or API and records the full request and response. A traffic analyzer turns
that record into an audit trail the user can search.

```
                         ┌────────────────────────────────┐
 Hermes ─┐  MCP          │  information-overlord (Railway) │      upstream MCP servers
 Claude ─┤ ─────────────▶│                                │────▶ (Gmail, Calendar, Drive,
 dots   ─┤               │  proxy:  identify user + agent │       memory, …)
 Muse   ─┤  HTTP API     │          apply rules           │
 scripts ┘ ─────────────▶│          add user's credentials│────▶ REST APIs
           agent key     │          record full traffic   │      (Google, …)
           only          │                                │
                         │  analyzer: parse resource IDs  │
                         │            search · dashboard  │
                         │            alerts              │
                         └───────────────┬────────────────┘
                                         │
                        Postgres: users, events & resource index (kept forever)
                                  payloads (90-day rolling, full-text searchable)
```

The proxy doesn't implement Gmail or Calendar itself. Adding a data source
means configuring an upstream, not writing tools.

### Multi-user hosting

One deployment serves many users (the same pattern as the author's
ai-quotas app). The intended scale is **self-hosted and small trusted-group
instances**: one person, a family, a team. Under 100 users, the host's Google
OAuth app can stay unverified (see Gotchas):

- Users **sign in with Google**. The host controls who can sign in with an
  email allowlist (`["*"]` for open sign-up).
- Each user connects **their own** data sources, creates **their own** agent
  keys, and sees **only their own** audit trail. Every query is scoped by
  `user_id`.
- **Admins** can see users, their connections and stored volume on `/admin`,
  but never their credentials or recorded payloads.
- Users can delete their history or their whole account.
- Public home, `/privacy` and `/terms` pages, as Google requires for a
  published OAuth app.

**Trust model:** payloads are stored as plain text so they can be searched.
Whoever operates the instance (anyone with database access) can read every
user's recorded data and use their connected credentials. The README says so
plainly. Users should use an instance run by someone they trust, or host
their own.

### What makes a user's trail complete

The proxy only sees traffic routed through it. It can't intercept a vendor's
own HTTPS connections to Google. Coverage is complete only when both of these
hold:

1. **The proxy holds the real credentials.** Agents only get revocable proxy
   keys.
2. **Direct connectors are turned off.** That includes the claude.ai Gmail,
   Calendar and Drive connectors and Muse's built-in Google connector.

Each user's dashboard has a **coverage checklist**: which agents and sources go
through the proxy, and which known direct paths the user should turn off.

## Goals

1. **Full-fidelity record.** Every call is recorded: user, agent, session,
   self-reported skill, upstream, tool or endpoint, full request, full
   response, status, timing. Denied and failed calls are recorded too.
2. **Search by resource.** Parse responses to pull out resource IDs (Gmail
   message or thread, calendar event, Drive file, memory), so a user can ask
   "who touched X?" and not just "who called URL Y?".
3. **Tamper evidence.** The log can only be added to, and each user's events
   are hash-chained to the one before, so history can't be quietly rewritten.
4. **Works with hosted agents.** A public HTTPS endpoint, so dots, Muse and
   claude.ai can reach it from their vendors' clouds.
5. **Supports both styles of agent.** MCP tool calls, *and* agent-written
   scripts calling a plain HTTP API (fewer model round trips, less data sent
   through the vendor's model). The audit trail is the same for both.
6. **Multi-user with strict isolation** between users.
7. **Optional controls.** Per-agent allowlists and approval of sensitive
   writes.

## Non-goals

- Seeing what an agent does with data *after* it reads it inside a vendor's
  cloud. The trail records access.
- Preventing prompt injection inside content. Approvals on writes are the
  mitigation.
- Protecting users from the instance operator. That's handled by the trust
  model, not by encryption.
- Transparent interception of traffic the agents aren't configured to send to
  the proxy.
- **Running a large public instance.** Holding many strangers' full inbox
  contents is a serious responsibility, and Gmail's restricted scopes would
  require Google verification plus an annual security assessment. Nothing
  prevents someone from doing it, but the project won't be designed around
  it.

## Client compatibility (researched 2026-10-24, not yet verified)

| Client | Reaches proxy from | Auth it supports | Status |
|---|---|---|---|
| Hermes | User's host | Bearer key | Fully under the user's control |
| Claude Code (local) | User's machine | Bearer header or OAuth; can also write scripts against the HTTP API | Documented |
| claude.ai / Claude cloud | Anthropic cloud | OAuth 2.1 (DCR + PKCE) | Documented |
| OpenAI dots | OpenAI cloud | OAuth only; dev-mode plugin at chatgpt.com/plugins | **Unconfirmed** that dots use dev-mode plugins; Pro may be read-only |
| Meta Muse | Meta cloud | Bearer key via secure credential prompt works best; OAuth unreliable | **Unconfirmed**: added through a chat-built "custom connector" |

So the proxy must accept **both** per-agent bearer keys **and** OAuth 2.1.
In the OAuth flow, users authenticate with the same Google sign-in, and the
issued token is bound to that user and a named agent.

## Architecture

### Stack (proposed, matching ai-quotas)

- **SvelteKit (adapter-node) and TypeScript** for the dashboard and the HTTP
  endpoints. The official MCP SDK (`@modelcontextprotocol/sdk`) handles the
  agent-facing `/mcp` endpoint (Streamable HTTP) and the client connections to
  upstreams.
- **Postgres** for everything: users, config, events, resource index, and
  payloads (in a table partitioned by day). If volume ever outgrows it,
  payloads can move to object storage later.
- **Railway** hosting, deployed from GitHub pushes to `main`.
- **Configuration in one JSON document**, as in ai-quotas: `config.json`
  locally, the `CONFIG_JSON` variable in production, pushed with
  `npm run config:push`. Keys include `databaseUrl`, `origin`,
  `sessionSecret`, `tokenEncryptionKey`, `google.clientId/clientSecret`,
  `allowedEmails`, `admins`, `contactEmail`, `payloadRetentionDays`
  (default 90).
- gitleaks in CI; no secrets in the repo.
- Version in `package.json`, shown in the site header and in `/healthz`, and
  returned by a `whoami` tool.

### Components

1. **Accounts.** Google sign-in using only the `openid email profile` scopes,
   gated by `allowedEmails`. Session cookies are random tokens, stored hashed.
2. **Connections (upstreams per user).**
   - Built-in **Google** connection. It's a separate consent step from sign-in
     that asks for the Gmail, Calendar and Drive scopes the user chooses
     (read-only by default). By default it uses the host's OAuth client.
   - **Bring your own Google client** (advanced). A user can paste the
     client ID and secret of an OAuth client from their own Google Cloud
     project. The connection then authorizes against that client, so the
     user's consent never counts against the host's 100-user unverified
     limit and no host verification is needed. The connection page links a
     step-by-step setup guide (create project, enable APIs, consent screen in
     production mode, web client with redirect `<origin>/connect/google/callback`).
   - **Custom MCP upstreams.** The user adds an MCP server URL and its
     credentials (bearer key or OAuth).
   - Credentials are encrypted at rest (AES-256-GCM, `tokenEncryptionKey`).
     Refreshes happen under a row lock so two instances never both spend a
     rotating refresh token (the same pattern as ai-quotas).
3. **Agent-facing front doors** (both share auth, rules and recording):
   - **MCP endpoint:** `/mcp`. The tools are the union of *that user's*
     upstream tools, with namespaced names (`gmail.search`, `memory.read`).
     Tool descriptions are kept short to limit context cost.
   - **HTTP API proxy:** `/api/<upstream>/...`, e.g.
     `/api/google/gmail/v1/users/me/messages`. It forwards with the user's
     credentials added. `/api/openapi.json` lists what's available so agents
     can write scripts.
4. **Agent identity.** Each request resolves to `(user_id, agent_id)`.
   - Bearer keys: 256-bit random, shown once, stored hashed, revocable.
   - OAuth 2.1 for dots and claude.ai: the user signs in with Google during
     the flow and picks or creates the agent the token belongs to.
   - Optional `X-Agent-Session` and `X-Agent-Skill` headers, or matching MCP
     metadata, recorded as **self-reported**.
5. **Recorder.** Writes the event row before forwarding (so failed and
   timed-out calls are still captured), then completes it with the response
   and payload. Credential headers are removed from recorded payloads.
6. **Parsers.** Small, per-upstream modules that turn request and response
   into resource references (`gmail:msg/<id>`, `gcal:event/<id>`,
   `drive:file/<id>`, `memory:<id>`) and an access kind (`list`,
   `read-metadata`, `read-content`, `write`, `delete`, `send`). Unknown
   traffic is still recorded, just not indexed by resource.
7. **Rules.** Per-agent allowlist by upstream and tool or endpoint, plus a
   mode for each: `allow` (default), `approve`, or `deny`. An approval returns
   "pending" immediately and doesn't block. The agent retries with the
   approval ID.
8. **Dashboard** (per user):
   - Timeline filtered by agent, upstream, tool, resource and time.
   - A page per resource ("who touched this?").
   - A page per agent: volume over time, sources and tools used.
   - The full request and response for any event inside the retention
     window, plus full-text search over payloads.
   - The coverage checklist.
   - Pages to manage connections, agents and keys, rules and approvals.
   - Account page: delete history, delete account.
9. **Alerts.** Per-user rules: volume spikes compared with an agent's
   baseline, first access to a new upstream or tool, any `send` or `delete`,
   access at unusual hours. Delivered by email or push notification.
10. **Admin** (`/admin`): users, connections, agent counts and storage per
    user. No payloads or credentials.
11. **Retention job.** Each day, drop the payload partition that has passed
    `payloadRetentionDays` (instant, no table bloat) and mark its events
    `payload_pruned`. Event metadata, resource references and hashes are kept
    until the user deletes them.

### Data model (first draft)

Every table except `users` carries `user_id`, and every query filters on it.

```sql
users(id uuid pk, email text unique, name text, created_at, last_seen_at, deleted_at)
sessions(token_hash text pk, user_id, created_at, expires_at)

connections(                      -- a user's upstreams
  id uuid pk, user_id,
  kind text,                      -- 'google' | 'mcp'
  label text,                     -- 'gmail', 'work-notion', …
  target text,                    -- MCP URL (for 'mcp')
  scopes text[],
  credentials_enc bytea,          -- AES-256-GCM
  oauth_client_enc bytea,         -- optional bring-your-own Google client ID + secret
  parser text,
  created_at, revoked_at
)

agents(
  id uuid pk, user_id,
  name text,                      -- 'hermes', 'claude-code', 'dots', 'muse'
  auth_kind text,                 -- 'bearer' | 'oauth'
  key_hash text,
  created_at, revoked_at
)

events(                           -- append-only: the app role has INSERT/SELECT
  id bigserial pk,                --   (plus DELETE only via account/history deletion)
  user_id, agent_id,
  at timestamptz default now(),
  session_id text, skill text,    -- self-reported
  front_door text,                -- 'mcp' | 'http'
  connection_id uuid,
  operation text,                 -- tool name, or 'GET /gmail/v1/…'
  decision text,                  -- 'allowed' | 'denied' | 'pending' | 'approved' | 'rejected'
  request_hash text,
  prev_hash text, chain_hash text -- per-user hash chain
)

event_results(
  event_id pk, user_id,
  completed_at, status text, error text, duration_ms int,
  response_hash text, response_bytes int,
  payload_pruned_at timestamptz
)

payloads(                         -- PARTITION BY RANGE (day)
  event_id, user_id, day date,
  request jsonb, response jsonb, truncated boolean,
  search tsvector
)

resource_refs(event_id, user_id, resource text, access text)   -- "who touched X"
rules(id, user_id, agent_id, connection_id, operation_pattern text, mode text)
approvals(id, user_id, event_id, requested_at, decided_at, decision text, decided_via text)
alerts(id, user_id, at, rule text, agent_id, event_ids bigint[], acknowledged_at)
```

User deletion: a user may delete their own events. That breaks the hash
chain, so deletion records a chain checkpoint. Tamper evidence protects
against the *app or an agent* rewriting history, not against the user
deleting their own data.

### Payload storage

- **Where:** Postgres, in a table partitioned by day. Postgres compresses
  large values automatically.
- **No application-level encryption** (decided 2026-10-30), so payloads are
  searchable. Protection comes from Railway's database security, a database
  that isn't publicly exposed, per-user query scoping, and the trust model
  above.
- **Retention:** `payloadRetentionDays`, default 90, pruned daily. Hashes stay,
  so an exported payload can still be verified later.
- **Size:** a rough per-user volume is a few thousand calls a day at about
  20 KB each: about 50 MB a day, or about 5 GB per heavy user at 90 days. Admin
  shows storage per user. A per-user quota is a later option.

### Data sent through MCP vs. scripts

MCP tool calls send every intermediate result through the model, and each
call is a model round trip. Agents that can run code are better off writing a
script against the HTTP API. The script runs once, filters on the proxy side,
and only its output reaches the model. That's cheaper, faster, and sends less
of the user's data to the vendor's model. The audit trail is identical either
way. Tool design should follow from this: search returns IDs and snippets,
and full content is a separate, explicit read.

## Stages

### Stage 1: Accounts, Gmail and the recorder
- SvelteKit scaffold modeled on ai-quotas: `package.json` at `0.1.0`, config
  loading, Postgres migrations, Railway service, gitleaks CI.
- Google sign-in, allowlist, sessions; home, privacy and terms pages.
- Google connection (Gmail read-only); credential encryption and refresh.
- Agents page: create and revoke bearer keys.
- `/mcp` proxy exposing Gmail tools (wrap an existing Gmail MCP server or call
  the Gmail API directly; decide at the start of the stage).
- The recorder, with full payloads and the per-user hash chain; timeline and
  event detail pages.
- Connect **Hermes** and **local Claude Code**.
- **Done when:** two users can sign in, each connects Gmail, and each sees only
  their own agents' Gmail access with full requests and responses.

### Stage 2: HTTP API proxy and parsers
- `/api/google/...` passthrough and `openapi.json`.
- Parsers for Gmail, Calendar and Drive, filling `resource_refs`.
- Resource page ("who touched X") and agent page; full-text search.
- **Done when:** a script reading Gmail through the HTTP API and an MCP tool
  call for the same message both show up on that message's page.

### Stage 3: Analyzer, retention, coverage and admin
- Retention job, storage view in admin, account and history deletion.
- Alert rules and notifications.
- Coverage checklist.
- Custom MCP upstreams.
- Connect **Muse** with a bearer key (verify the custom-connector flow).

### Stage 4: OAuth 2.1 and rules
- Protected-resource and authorization-server metadata, dynamic client
  registration, PKCE, refresh, with Google sign-in as the login step.
- Connect **claude.ai** and **dots** (verify plan limits and dev-mode
  plugins).
- Allowlists and approvals.

### Stage 5: Memory as an upstream
- A memory MCP server that records where each memory came from (writing
  agent and skill, source resource IDs, derived-from memories). The proxy
  supplies "who read it".

### Stage 6: Join in harness traces (optional)
- Collect OpenTelemetry from Claude Code (`CLAUDE_CODE_ENABLE_TELEMETRY`) and
  Hermes, keyed by session ID, to show *why* next to *what*.

## Gotchas to remember

- **Google verification for restricted scopes.** Gmail read is a *restricted*
  scope.
  - An unverified app works for up to 100 users and shows a warning screen.
  - In *testing* mode, refresh tokens expire after 7 days, so run the app in
    *production, unverified*.
  - A public instance needs Google's app verification plus an annual
    third-party security assessment (CASA), which has real cost.
  - Ways to stay out of review: keep an instance under 100 users;
    *bring your own client* for users beyond that (or anyone who prefers it);
    or, on a Google Workspace domain, mark the app *internal*, which needs no
    verification for that domain's users.
  - Changing the *sign-in* method wouldn't help. Sign-in uses only basic
    scopes. Review is triggered by Gmail data access, whatever the sign-in
    method.
  - Gmail over IMAP with app passwords would avoid OAuth but covers mail only
    and is being steadily restricted by Google. Don't build on it.
  - Check Google's current rules before relying on any of this.
- **Keep sign-in and data scopes separate.** Sign-in asks only for identity.
  Data scopes come from a separate "connect Google" step, so users who only
  connect other upstreams never see restricted-scope warnings.
- **Opaque hosted clients.** Record full HTTP exchanges on the OAuth
  endpoints too. It's the only way to debug dots and Muse when they fail to
  connect.
- **High-value target.** Users' Google credentials *and* 90 days of what
  agents read, on a public URL. Encrypt credentials, keep the database off the
  public network, keep upstream scopes narrow, make keys easy to rotate, never
  return payloads through agent-facing endpoints, and test isolation between
  users explicitly.
- **Large responses.** Cap payload size per event (e.g. 5 MB), mark truncated
  payloads, and don't buffer huge streams in memory.

## Open questions

1. Stage 1: wrap an existing Gmail MCP server, or implement Gmail tools over
   the Gmail API directly? With multiple users, the proxy has to pass each
   user's token to a shared upstream server, which favors calling the API
   directly for built-in Google sources.
2. Does the Hermes skills repo have a manifest or permissions format that
   agent allowlists should reuse?
3. OAuth in stage 4: write it with a library, or use a hosted identity
   service?
4. Per-user storage quotas: needed from the start, or only once there's a
   public instance?
5. Should approvals and alerts notify by email, web push, or both?

## Alternatives considered

Hosted gateways (Zapier MCP, Composio, Arcade, Pipedream Connect) and
open-source ones (Obot, Metorial, IBM ContextForge, small Go gateways). Their
focus is *access management*. Logging is a secondary feature, usually
metadata-only, and the hosted ones keep the record themselves. This project's
focus is *the audit trail itself*: full payloads, search by resource, tamper
evidence and analysis, on an instance the user or someone they trust
controls.
