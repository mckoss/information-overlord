# information-overlord — Plan

## Status and next steps (as of 2026-10-31)

**Paused for design thinking. No code yet.** Where the project stands:

- Design so far: purpose, method (logging proxy plus traffic analyzer),
  multi-user hosting on Railway, Postgres-only storage, 90-day searchable
  payloads, small-instance scope with bring-your-own Google client.
- Client research (2026-10-24 and 2026-10-30, below): the proxy can be
  *added* to ChatGPT/dots, Muse and Claude, but no client guarantees it's the
  *only* path to Gmail. Agent browsers and shells can go around it. The
  strongest universal control is on Google's side.
- That research moved OAuth (with Client ID Metadata Documents and DCR) from
  stage 4 to stage 2.

When picking this up again:

1. Revisit the open design questions (see **Open questions**), especially
   whether a "trail that may have gaps" (coverage checklist plus Google-side
   backstop) is good enough, or whether the project needs a stronger answer
   to browser bypass.
2. Do the cheap manual experiments in **Drift tests** before writing code.
   They decide which clients are worth supporting first.
3. Then start stage 1 in a worktree.


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
own HTTPS connections to Google. Coverage is complete only when all of these
hold:

1. **The proxy holds the real credentials.** Agents only get revocable proxy
   keys.
2. **Built-in connectors are removed.** Disconnect the account *and*
   uninstall or remove the connector in each client.
3. **Agent browsers and shells can't reach Gmail.** Never sign in to Google
   in an agent's browser, block `mail.google.com` and `accounts.google.com`
   where the client allows it, and keep Google credentials out of agent
   shells.
4. **Google-side backstop.** In the user's Google account (Security →
   third-party access), the proxy's OAuth app is the only one with Gmail
   access. This is the one control that applies to every client at once.

Each user's dashboard has a **coverage checklist** built from the per-client
items in **Client research** below. The agent page shows which items are done
and which known direct paths remain open. The trail is honest about its
gaps.

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

## Client research (2026-10-24, updated 2026-10-30)

Sources were mostly official vendor docs (learn.chatgpt.com,
developers.openai.com, code.claude.com, support.claude.com, meta.com/help,
Meta's Muse security post), plus third-party guides where official docs say
nothing. **[unverified]** marks claims that need a hands-on test.

### Summary

| | ChatGPT / dots | Meta Muse | Claude (claude.ai, Cowork, Claude Code) | Hermes |
|---|---|---|---|---|
| **How it reaches Gmail by default** | Gmail plugin (Google sign-in); or its cloud browser, which it picks on its own | Built-in Google connector (Google sign-in, runs in Meta's sandboxed workers) | Anthropic's Gmail connector, also loaded into Claude Code and Cowork; Claude in Chrome | Whatever its skills use |
| **Can the default be turned off?** | Yes: uninstall the plugin **and** disconnect the account; admins can disable it (Business/Enterprise) | Yes, in Settings → Connectors. No permanent block; Muse may offer to reconnect | Yes: disconnect on claude.ai; admins can block it; `disableClaudeAiConnectors` in Claude Code | Yes, the user controls it |
| **How the proxy is added** | chatgpt.com/plugins → + → Add custom MCP server (web only) | Ask Muse in chat to create a custom connector; it writes an MCP client and saves it as a skill. A Settings form is claimed but **[unverified]** | Custom connector on claude.ai once and it syncs to Claude Code, Cowork and desktop; or `claude mcp add --transport http` locally | MCP config |
| **Auth it supports** | **OAuth only**: static client, Client ID Metadata Documents, or DCR. No static bearer token | Bearer key via secure credential prompt | OAuth: Client ID Metadata Documents (recommended), DCR or static client. Bearer headers only in a limited beta. Claude Code: bearer header or OAuth | Bearer key |
| **Calls come from** | OpenAI cloud (public URL, or OpenAI Secure MCP Tunnel) | Meta cloud, via its Sentinel forward proxy (public URL) | Anthropic cloud, even for Desktop and Cowork (public URL, or Anthropic MCP tunnels); local Claude Code calls directly | User's host |
| **Plan limits** | Dots: Pro 100/200/500, Business Premium, Enterprise. Custom MCP: full on Business/Enterprise; Pro read-only **[unverified]**; Plus unreliable | US, adults only; no documented limit on custom connectors | All plans; Free limited to one custom connector; on Team/Enterprise only Owners add connectors | None |
| **Main ways around the proxy** | Cloud browser logging into gmail.com, with sessions that persist and saved passwords; a connected local computer | VM browser with saved passwords; Muse's own email address (forwarded mail); its own code with an app password | Claude in Chrome or Cowork's browser using the user's Google session; shell running curl with the user's credentials; another Gmail MCP server | Its skills' code |
| **Closing those** | Per-site block in cloud-browser settings (whether it covers dots is **[unverified]**); admins can turn off the browser and computer use entirely; dots' Custom Rules are advice the dot tries to follow, *not* enforced | Weak: no documented per-domain block. Sentinel requires approval before data leaves once a task has read user data | Chrome site block list (Team/Enterprise; also covers Cowork's browser); sandbox network allowlist; deny rules (`mcp__claude_ai_Gmail__*`, Google API domains); `allowedMcpServers` by URL | Full control |
| **Feasibility** | **Medium** (high on Business, low on Plus) | **Medium** (Muse Code: **high**) | **Medium-high** (Claude Code), **medium** (claude.ai, Cowork) | **High** |

### Per-client notes

**ChatGPT / dots**
- The current docs call agent mode **ChatGPT Work** and connectors
  **Plugins**. Dots use the plugins installed for the account; there's no
  separate MCP URL for a dot.
- Write tools ask for confirmation by default. Mark read tools with
  `readOnlyHint` so they aren't treated as writes.
- **[unverified]** Whether custom MCP plugins are actually used in Work cloud
  tasks and by dots (older docs said agent mode didn't use custom apps).

**Meta Muse**
- No Meta page mentions MCP. Third-party guides say Muse writes a client with
  the official MCP SDK over streamable HTTP and saves it as a skill.
- Every call still reaches the proxy. A skill-name header (for example
  `X-Muse-Skill`) can be requested from Muse but is only a self-reported
  hint.
- Muse can receive email at its own address. That path never touches the
  proxy, so the checklist should say not to use it.
- **Muse Code**, per third-party docs: `~/.config/muse/settings.json` with an
  `mcp_servers` block (`streamable_http` with `url` and `headers`), and
  `muse mcp login <server>` for OAuth. That's a cleaner fit, because the
  config is the user's and not agent-written code.

**Claude**
- Claude Code loads claude.ai connectors only when signed in with a claude.ai
  subscription. Gmail tools appear as `mcp__claude_ai_Gmail__<tool>`.
- Ways to turn them off in Claude Code:
  - `"disableClaudeAiConnectors": true`, or `ENABLE_CLAUDEAI_MCP_SERVERS=false`.
  - A deny rule on `mcp__claude_ai_Gmail__*`.
  - Managed `deniedMcpServers` (match by `serverUrl`).
- **Catch:** `disableClaudeAiConnectors` also hides the user's own claude.ai
  custom connector, so add the proxy locally with `claude mcp add` too.
- None of those Claude Code settings affect the desktop app's local or SSH
  sessions. There, only disconnecting on claude.ai or an org-level `blocked`
  setting works.
- Extra audit signals: PreToolUse/PostToolUse hooks (matcher
  `mcp__<server>__.*`) and OpenTelemetry (`OTEL_LOG_TOOL_DETAILS=1`). Neither
  covers claude.ai chat or Cowork's browser.

### What this means for the design

- **OAuth is required early.** ChatGPT accepts only OAuth, and on claude.ai
  OAuth is the dependable option. Implement **Client ID Metadata Documents and
  DCR** (both vendors support them), with Google sign-in as the login step.
  Keep bearer keys for Hermes, Muse and local Claude Code.
- **One claude.ai custom connector covers Claude Code, Cowork and desktop,**
  which makes OAuth worth a lot.
- **Streamable HTTP at `/mcp`**, with tools annotated (`readOnlyHint`).
- The coverage checklist needs **per-client items** (see the table) and the
  **Google-side backstop**.

### Drift tests (manual, before or alongside stage 1)

For each client: remove the built-in Gmail connector, connect only the proxy
(or a test MCP server with one harmless `whoami` tool that logs every
request), then ask the agent to "check my email". Record whether it:

1. calls the proxy,
2. falls back to its browser,
3. offers to reconnect the built-in connector, or
4. refuses.

Also verify the **[unverified]** items: dots and Work actually call custom
plugins; ChatGPT Pro's write limits; whether ChatGPT's per-site browser
blocks apply to dots; Muse's exact connector flow; whether claude.ai
Request headers are available to the account.

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
  loading, Postgres migrations, Railway service.
- GitHub Actions CI (check, tests, build, gitleaks) from day one, with the
  Railway deploy gated on it (`checkSuites: true`).
- Google sign-in, allowlist, sessions; home, privacy and terms pages.
- Google connection (Gmail read-only); credential encryption and refresh.
- Agents page: create and revoke bearer keys.
- `/mcp` proxy exposing Gmail tools (wrap an existing Gmail MCP server or call
  the Gmail API directly; decide at the start of the stage).
- The recorder, with full payloads and the per-user hash chain; timeline and
  event detail pages.
- Connect **Hermes** and **local Claude Code** (bearer).
- **Done when:** two users can sign in, each connects Gmail, and each sees only
  their own agents' Gmail access with full requests and responses.

### Stage 2: OAuth 2.1 for hosted clients
- Protected-resource and authorization-server metadata, **Client ID Metadata
  Documents** and **dynamic client registration**, PKCE, refresh, with Google
  sign-in as the login step. The issued token is bound to the user and a named
  agent.
- Connect **claude.ai** as a custom connector (which also covers Claude Code,
  Cowork and desktop) and **ChatGPT/dots** as a custom plugin.
- Connect **Muse** with a bearer key.
- Record full HTTP exchanges on the OAuth endpoints for debugging.
- **Done when:** claude.ai, ChatGPT and Muse calls show up under their own
  agent identities.

### Stage 3: HTTP API proxy and parsers
- `/api/google/...` passthrough and `openapi.json`.
- Parsers for Gmail, Calendar and Drive, filling `resource_refs`.
- Resource page ("who touched X") and agent page; full-text search.
- **Done when:** a script reading Gmail through the HTTP API and an MCP tool
  call for the same message both show up on that message's page.

### Stage 4: Analyzer, retention, coverage, rules and admin
- Retention job, storage view in admin, account and history deletion.
- Coverage checklist with per-client items and the Google-side backstop.
- Alert rules and notifications.
- Allowlists and approvals.
- Custom MCP upstreams.

### Stage 5: Memory as an upstream
- A memory MCP server that records where each memory came from (writing
  agent and skill, source resource IDs, derived-from memories). The proxy
  supplies "who read it".

### Stage 6: Join in harness traces (optional)
- Collect OpenTelemetry from Claude Code (`CLAUDE_CODE_ENABLE_TELEMETRY`,
  `OTEL_LOG_TOOL_DETAILS=1`) and Hermes, keyed by session ID, to show *why*
  next to *what*.

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
3. OAuth in stage 2: write it with a library, or use a hosted identity
   service? It must support Client ID Metadata Documents and DCR.
4. Per-user storage quotas: needed from the start, or only once there's a
   public instance?
5. Should approvals and alerts notify by email, web push, or both?
6. **Is a trail with possible gaps good enough?** Agent browsers (ChatGPT
   Work/dots, Muse, Claude in Chrome/Cowork) can reach gmail.com directly if
   ever signed in. Options: accept it and rely on the checklist and the
   Google-side backstop; detect it (for example, watch Google account
   security activity for new agent sign-ins); or scope the project's promise
   to "every access through the proxy" and say so plainly.
7. Which client to support first after Hermes and Claude Code? The drift
   tests should decide.

## Alternatives considered

Hosted gateways (Zapier MCP, Composio, Arcade, Pipedream Connect) and
open-source ones (Obot, Metorial, IBM ContextForge, small Go gateways). Their
focus is *access management*. Logging is a secondary feature, usually
metadata-only, and the hosted ones keep the record themselves. This project's
focus is *the audit trail itself*: full payloads, search by resource, tamper
evidence and analysis, on an instance the user or someone they trust
controls.
