# information-overlord

**An audit trail of AI agents' access to your personal data.**

AI agents (Hermes, Claude Code, OpenAI dots, Meta Muse, …) now read your
email, calendar, files and memory on your behalf, and each one logs that
differently or not at all. information-overlord is a **logging proxy and
traffic analyzer** that every agent goes through to reach your data, so
there's one complete, searchable record of what each agent accessed.

## How it works

- **Proxy.** Agents connect to information-overlord, over MCP or a plain HTTP
  API for agent-written scripts, using a revocable per-agent key. The proxy
  holds the real credentials and forwards each request to the upstream MCP
  server or API (Gmail, Calendar, Drive, memory, …).
- **Recorder.** Every request and response is recorded in full: agent,
  session, upstream, operation, payloads, timing. Denied and failed calls are
  recorded too. Event metadata is append-only, hash-chained, and kept
  forever. Full payloads are encrypted and kept on a rolling **90-day**
  window.
- **Analyzer.** Parsers pull out resource IDs, so you can ask "who touched
  this message?", see each agent's access over time, get alerts on unusual
  patterns, and view exactly what an agent saw.
- **Coverage.** The trail is only complete if agents have no other path to
  your data. The dashboard tracks which agents and sources go through the
  proxy and which direct connectors are still open.
- **Optional controls.** Per-agent allowlists and approval of sensitive
  writes, enforced at the same point.

It runs as a single service with Postgres and object storage. It's designed
for Railway, so cloud-hosted agents can reach it at a public HTTPS URL.

## Status

Planning. See [PLAN.md](PLAN.md) for the design, data model and staged
roadmap. No code yet.

## Roadmap

1. MCP proxy and recorder with full payloads; Hermes and Claude Code connected.
2. HTTP API proxy for scripts; resource parsers and "who touched X".
3. Analyzer: alerts, 90-day retention, coverage list; Muse connected.
4. OAuth 2.1 (claude.ai, dots), allowlists and approvals.
5. Memory as an upstream, recording where each memory came from.
6. Optional: join in OpenTelemetry traces from the agents.

## License

[MIT](LICENSE)
