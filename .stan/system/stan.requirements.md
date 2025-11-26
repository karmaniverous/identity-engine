# Project Prompt — Identity Engine

This file records durable, repo-specific requirements and decisions that guide implementation and reviews.

## Repository signature

- package.json name: @karmaniverous/identity-engine
- stanPath: .stan
- Node: >= 20
- Language: TypeScript (ESM), Vitest for tests, Rollup for builds

## Scope of the first delivery

Feature-complete matching/mastering engine at modest scale (10k–50k records/day), focusing on deterministic logic and APIs. Connectivity to managed services is mocked; the logic, schemas, and contracts are real and testable.

## Architectural constraints (durable)

- API key auth on all inbound endpoints (X-API-Key), validated by an authorizer in later phases.
- Handlers are pure functions created by a factory: makeHandlers(deps) → { ...routes }.
- Each handler has a single parameter named event and must be validated by a Zod schema; the return value must also be validated by a Zod schema.
- External services (DynamoDB, SQS, EventBridge/OpenSearch, clock, id) are injected via deps and are mocked in this phase (in‑memory).
- Idempotency and optimistic concurrency are expressed at the API/contract level and enforced in mocks.
- Late‑arrival window and matching windows are configurable; defaults live in config.
- CLI mirrors the HTTP handlers: each command accepts a JSON event (stdin or --file) and prints JSON result.

## Module surfaces (ports)

- Handlers (application API):
  - ingest(event)
  - candidates(event)
  - link(event)
  - getEntity(event)
  - listEntityRecords(event)
  - createOverride(event)
  - deleteOverride(event)
  - createTenant(event)
  - getTenant(event)
  - updateTenant(event)
  - createApiKey(event)
  - revokeApiKey(event)
  - publishBundle(event)
  - validateBundle(event)
  - activateBundle(event)
  - simulate(event)
  - shadowRun(event)
  - health(event)

- Deps (injected; all mocked for now):
  - store: EntitiesStore (entity, record link, version, overrides, outbox)
  - blockIndex: BlockIndexStore
  - outbox: Outbox (collects events for assertions)
  - auth: ApiKeyRepo (lookup/authorize tenant by key)
  - clock: Clock (nowISO)
  - id: Id (ulid/uuid)

## Endpoint map (HTTP contract; mirrors handlers)

- POST /v1/ingest → ingest(event)
- GET /v1/candidates → candidates(event)
- POST /v1/link → link(event)
- GET /v1/entities/{entityId} → getEntity(event)
- GET /v1/entities/{entityId}/records → listEntityRecords(event)
- POST /v1/overrides → createOverride(event)
- DELETE /v1/overrides → deleteOverride(event)
- POST /v1/tenants → createTenant(event)
- GET /v1/tenants/{tenantId} → getTenant(event)
- PUT /v1/tenants/{tenantId} → updateTenant(event)
- POST /v1/tenants/{tenantId}/keys → createApiKey(event)
- DELETE /v1/tenants/{tenantId}/keys/{keyId} → revokeApiKey(event)
- POST /v1/bundles → publishBundle(event)
- POST /v1/bundles/{bundleId}/validate → validateBundle(event)
- POST /v1/bundles/{bundleId}/activate → activateBundle(event)
- POST /v1/simulate → simulate(event)
- POST /v1/shadow-run → shadowRun(event)
- GET /healthz → health(event)

All endpoints require X-API-Key; the authorizer resolves tenantId and injects it as part of the event in later phases. For this phase, tenantId is provided directly in the event.

## Zod event/response contracts (names only; to be implemented under src/api/schemas)

- IngestEventSchema: { tenantId, sourceId, idempotencyKey, eventTimeISO, rawRef?: string, record: unknown }
- IngestResultSchema: { accepted: boolean, recordId: string }

- CandidatesEventSchema: { tenantId, recordId, windowDays?: number, perKeyCap?: number, globalCap?: number }
- CandidatesResultSchema: { candidates: Array<{ recordId: string; entityId?: string; score?: number; features?: Record<string, unknown> }> }

- LinkEventSchema: { tenantId, recordId, target: { entityId?: string; createNew?: boolean }, ruleId: string, ruleTrace?: unknown, bundleId: string, sdkVersion: string }
- LinkResultSchema: { entityId: string; entityVersion: number; changedFields: string[] }

- GetEntityEventSchema: { tenantId, entityId: string }
- GetEntityResultSchema: { entity: unknown | null }

- ListEntityRecordsEventSchema: { tenantId, entityId: string, limit?: number, cursor?: string }
- ListEntityRecordsResultSchema: { items: Array<{ recordId: string; sourceId: string; eventTimeISO: string }>; nextCursor?: string }

- CreateOverrideEventSchema: { tenantId, kind: 'must_link'|'cannot_link', a: { type: 'record'|'entity', id: string }, b: { type: 'record'|'entity', id: string }, reason?: string, user?: string }
- CreateOverrideResultSchema: { ok: true, overrideId: string }

- DeleteOverrideEventSchema: { tenantId, overrideId: string }
- DeleteOverrideResultSchema: { ok: true }

- CreateTenantEventSchema: { tenantId: string, name?: string, settings?: unknown }
- CreateTenantResultSchema: { ok: true }

- GetTenantEventSchema: { tenantId: string }
- GetTenantResultSchema: { tenant: { tenantId: string; name?: string; settings?: unknown } | null }

- UpdateTenantEventSchema: { tenantId: string, name?: string, settings?: unknown }
- UpdateTenantResultSchema: { ok: true }

- CreateApiKeyEventSchema: { tenantId: string, role: 'admin'|'ingest'|'read', scopes?: string[] }
- CreateApiKeyResultSchema: { keyId: string, secretPreview: string }

- RevokeApiKeyEventSchema: { tenantId: string, keyId: string }
- RevokeApiKeyResultSchema: { ok: true }

- PublishBundleEventSchema: { tenantId: string, bundleId: string, sdkVersion: string, artifactUri: string }
- PublishBundleResultSchema: { ok: true }

- ValidateBundleEventSchema: { tenantId: string, bundleId: string }
- ValidateBundleResultSchema: { ok: true, report: unknown }

- ActivateBundleEventSchema: { tenantId: string, bundleId: string }
- ActivateBundleResultSchema: { ok: true }

- SimulateEventSchema: { tenantId: string, moduleId: string, input: unknown }
- SimulateResultSchema: { ok: true, decision: unknown }

- ShadowRunEventSchema: { tenantId: string, bundleId: string, fromISO: string, toISO: string, sample?: number }
- ShadowRunResultSchema: { ok: true, metrics: unknown }

- HealthEventSchema: { }
- HealthResultSchema: { ok: true, nowISO: string }

## Testing policy (durable)

- Unit tests for every handler happy-path and key error paths (validation, overrides conflict, idempotent duplicate).
- Contract tests: Zod schemas reject bad inputs, coerce/transform as specified.
- CLI tests: invoke commands with sample events via stdin and --file; verify exit codes and JSON output.
- Mocks must support deterministic time (Clock.nowISO injected), ULID/UUID generation (seeded), and in-memory stores with TTL where needed.

## CLI surface (Commander)

- ideng ingest --file evt.json
- ideng candidates --record-id R123 --tenant T1
- ideng link --file evt.json
- ideng entity get --entity-id E123 --tenant T1
- ideng entity records --entity-id E123 --tenant T1
- ideng overrides add --file evt.json; ideng overrides rm --override-id O1 --tenant T1
- ideng tenants create --tenant T1; ideng tenants get --tenant T1; ideng tenants update --tenant T1 --file settings.json
- ideng keys create --tenant T1 --role ingest; ideng keys revoke --tenant T1 --key-id K1
- ideng bundles publish/validate/activate --file evt.json
- ideng simulate --file evt.json
- ideng shadow-run --file evt.json

Commands read an event JSON object (stdin or --file) that must conform to the corresponding Zod event schema; output validates against the response schema.

## File layout (to be implemented)

- src/api/schemas/*.ts (Zod event/response schemas, re‑exported via index)
- src/api/handlers/*.ts (pure handler factories; export makeHandlers)
- src/api/deps/*.ts (interface types; mocks in src/mocks)
- src/mocks/* (in‑memory stores: EntitiesStore, BlockIndexStore, Outbox, ApiKeyRepo; clock, id)
- src/cli/ideng/* (Commander commands; each wraps a handler)
- test/* (unit + CLI tests)

## Non‑goals in this phase

- No real AWS connectivity (DynamoDB/SQS/etc.). No OpenSearch. No EventBridge.
- No HTTP framework wiring; only pure functions and CLI.
- No operator UI.

## Acceptance bar for “production‑ready MVP”

- All handlers implemented with Zod input/output schemas.
- 90%+ branch coverage on handlers; CLI happy‑path tests pass.
- Mocks demonstrate idempotency, overrides precedence, window checks, and deterministic mastering for the chosen domain (Transactions).
- Endpoint contracts are stable and documented here and via TypeDoc.
