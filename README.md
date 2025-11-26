# Identity Engine

Status: design/spec phase. This repo currently contains project scaffolding, [requirements](.stan/system/stan.requirements.md), and this conceptual README. No installation or usage is expected at this time.

Identity Engine is a deterministic identity‑resolution and entity‑mastering engine written in TypeScript. It ingests records, proposes match candidates, links records into entities with override support, and enforces idempotency and windowing policies. All handlers are pure functions created by a factory makeHandlers(deps) and validated with Zod (inputs and outputs). A CLI will mirror the future HTTP endpoints by accepting an event JSON and returning a validated JSON response.

Concept at a glance

- Deterministic matching/mastering with explicit overrides
- Zod‑validated contracts (events and results)
- Services‑first architecture; thin adapters (CLI/HTTP later)
- In‑memory mocks for deps (store, block index, outbox, API key repo, clock, id)
- Runtime bundle flow: publish → validate → activate; plus simulate and shadow‑run
- Tenants and API keys; health endpoint

Planned surfaces (first delivery)

- Handlers: ingest, candidates, link, getEntity, listEntityRecords, overrides (create/delete), tenants (create/get/update), api keys (create/revoke), bundles (publish/validate/activate), simulate, shadowRun, health
- CLI: one command per handler; reads an event JSON (stdin or --file), prints JSON; validates both request and response via the same Zod schemas

---

Built for you with ❤️ on Bali! Find more great tools & templates on [my GitHub Profile](https://github.com/karmaniverous).
