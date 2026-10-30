# information-overlord

**See exactly what your AI agents access on your behalf.**

AI agents such as Claude, ChatGPT agents, Meta Muse and self-hosted harnesses
like Hermes now read your email, calendar, files and memory. Each one logs
that access differently or not at all, so there's no single place to answer
basic questions:

- Which agents read my inbox this week, and what exactly did they see?
- Who touched this message, event or file, and when?
- Did an agent start reading more than usual, or reach somewhere new?
- What did an agent send, change or delete on my behalf?

information-overlord answers those questions. It's a **logging proxy and
traffic analyzer** that sits between your agents and your data and keeps a
complete, searchable **audit trail** of every access.

## How it works

1. **Sign in with Google** and **connect your data sources**, such as Gmail,
   Google Calendar, Google Drive, or any MCP server. information-overlord
   holds those credentials; your agents never see them.
2. **Create a key for each agent** and point the agent at your
   information-overlord URL instead of connecting it to your data directly.
   Agents can connect over **MCP** (Claude, ChatGPT, Muse, Hermes, …) or call a
   plain **HTTP API** from scripts they write.
3. **Every request is recorded in full:** which agent, when, what it asked
   for, and the complete response it got back. Failed and denied calls are
   recorded too.
4. **Explore the audit trail** in the dashboard:
   - a timeline of all agent activity
   - a page for each agent
   - a "who touched this?" page for any message, event or file
   - full-text search over what agents saw
   - alerts on unusual activity

### What you get

- **Complete record.** The full request and response for every call,
  searchable for 90 days. A permanent, tamper-evident index records who
  accessed what, and when.
- **One view across agents.** It works the same for every agent, including
  hosted ones you can't inspect or modify.
- **Coverage check.** The trail is only complete if agents have no other path
  to your data. The dashboard shows which agents and sources go through
  information-overlord, and reminds you to turn off direct connectors.
- **Optional controls.** Limit each agent to specific sources and tools, and
  require your approval before an agent sends or deletes anything.
- **Your data, your call.** Delete your history or your whole account at any
  time.

## Privacy and trust

information-overlord stores the **full content** of what your agents read (for
90 days) so you can see it later. That means:

- **Whoever runs the server can read your recorded data and holds your
  connected credentials.** Use an instance run by someone you trust, or host
  your own.
- Connected-account credentials are encrypted at rest. Agent keys are random,
  shown once, stored only as hashes, and can be revoked at any time.
- Each user's agents, credentials and audit trail are private to that user.
  Administrators can see who has accounts and how much they store, not the
  content.
- It records access. It can't see what an agent does with your data after
  reading it inside the agent vendor's own systems.

## Hosting an instance

information-overlord is a single web service backed by Postgres, designed to
deploy on [Railway](https://railway.com). Hosted agents (ChatGPT, Muse,
claude.ai) connect from their vendors' clouds, so the server needs a public
HTTPS URL. The host:

- creates a Google OAuth client for sign-in and for connecting Google data,
- decides who may sign in (an email allowlist) and who are administrators,
- publishes the privacy policy and terms pages that Google requires for an
  OAuth app.

Detailed setup instructions will come with the first release.

> **Google verification.** Reading Gmail is a *restricted* Google scope. A
> self-hosted instance for yourself, family or a small group can run as an
> unverified app (users see a warning screen once, up to 100 users). Opening
> an instance to the public requires Google's app verification and security
> assessment.

## Status

**Planning, no code yet.** See [PLAN.md](PLAN.md) for the design, data model
and roadmap.

## Roadmap

1. MCP proxy and recorder; Google sign-in; Gmail connected; first agents.
2. HTTP API proxy for agent scripts; "who touched this?" resource pages.
3. Alerts, 90-day retention, coverage check.
4. OAuth for hosted agents (claude.ai, ChatGPT); per-agent limits and
   approvals.
5. Agent memory as a recorded data source, tracking where each memory came
   from.

## License

[MIT](LICENSE)
