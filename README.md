# information-overlord

A personal information gateway for AI agents.

Every agent I use (Hermes, Claude Code, OpenAI dots, Meta Muse) reaches my
email, calendar and memory through this one MCP server instead of holding my
credentials itself. The gateway authenticates each agent, enforces per-agent
policy, asks me before sensitive writes, and keeps an append-only audit log of
every access. That gives me one place to answer "which agent read what, and
when?"

It also stores agent memory **with provenance**: each memory records which
agent and skill wrote it and what source data it came from, and every read is
logged.

## Status

Planning. See [PLAN.md](PLAN.md) for the design, data model and staged
roadmap. No code yet.

## How it will work

- **Transport:** MCP over Streamable HTTP, hosted on Railway at a public HTTPS
  URL so cloud-hosted agents can reach it.
- **Auth:** a bearer key per agent (Hermes, Claude Code, Muse) and OAuth 2.1
  (claude.ai, dots).
- **Upstream:** a single Google OAuth grant for my account, stored encrypted;
  agents never see it.
- **Audit:** every tool call, including denied ones, is written to an
  insert-only Postgres table. It stores resource IDs and hashes, not content.
- **Dashboard:** browse the log by agent, tool and resource; trace any memory
  back to its sources.

## Roadmap

1. Gmail read tools, bearer auth, audit log; Hermes and Claude Code connected.
2. Muse connected; approval queue for writes.
3. OAuth 2.1; claude.ai and dots connected.
4. Memory with provenance.
5. Optional: join in OpenTelemetry traces from the agents.

## License

[MIT](LICENSE)
