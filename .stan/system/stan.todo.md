# Development Plan

When updated: 2025-09-02T16:45:00Z

## Next up

- Add zod dependency and commit baseline schema module.
- Define Zod event/response schemas under src/api/schemas for:
  - ingest, candidates, link, getEntity, listEntityRecords
  - overrides (create/delete), tenants (create/get/update), api keys (create/revoke)
  - bundles (publish/validate/activate), simulate, shadowRun, health
- Define deps interfaces under src/api/deps (Store, BlockIndex, Outbox, ApiKeyRepo, Clock, Id).
- Implement in‑memory mocks under src/mocks with deterministic behavior.
- Implement handler factory src/api/handlers/makeHandlers.ts:
  - Construct closures over deps; each handler takes a single event parameter and validates with Zod.
  - Validate return payloads with Zod before returning.
- Wire minimal CLI (src/cli/ideng) using Commander:
  - One command per handler; read JSON event from stdin or --file; print JSON.
  - Validate inputs with the same Zod schemas.
- Tests (Vitest):
  - Unit tests for each handler (happy/error paths).
  - CLI tests invoking commands with sample events.
  - Determinism tests (fixed clock/id).
- Docs:
  - TypeDoc for public types; README snippets for CLI usage.

## Completed (recent)

- Repository context verified; created stan.project.md with endpoint map, constraints, and contracts.
