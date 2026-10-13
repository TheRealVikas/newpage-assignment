# Meridian Claude + Gemini Usage Connector

Assignment ID: `Fullstack-ClaudeGemini-UsageConnector-v1`.

This MVP ingests supplied Claude Enterprise and Vertex AI Gemini exports into SQLite, serves scoped usage APIs and a small admin page, and exposes the same governed information through a four-tool local MCP server.

## Quick setup

Prerequisite: Node.js 22+ and npm. Run `npm install`, then `npm run setup`, then `npm run dev`. Open `http://localhost:3000`. The initial setup ingests deterministic fixtures and renders generated manifests. Other useful commands are `npm test`, `npm run lint`, `npm run dev:mcp`, `npm run provision`, and `npm run demo`.

For MCP, copy `.env.example` to `.env` and select a fixture identity for `MERIDIAN_CALLER_USER_ID`. The value is a `user_id` from `starter-kit/data/directory/users.json`, not an external credential. The included default is Priya Nair, the Clinical Operations team lead. The server now loads `.env` automatically; alternatively set `MERIDIAN_CALLER_USER_ID` and `MERIDIAN_ENV` in PowerShell **before** `npm run dev:mcp`.

## Structure and architecture

`packages/core` contains directory lookup, policy loading, periods, money conversion, SQLite queries, and capability defaults. `scripts/ingest.ts` maps fixtures. `apps/api` owns REST and serves `apps/web`; `apps/mcp` is a thin stdio JSON-RPC MCP adapter over the same core service. `generated/` is provisioning output; packaging examples live in `claude-desktop/` and `gemini-extension/`.

```text
Claude JSON + Gemini JSONL -> ingestion -> SQLite
                                      |        |
                              REST/admin UI   MCP stdio -> Claude/Gemini host
                                      ^
                         configuration-driven policy + capabilities
```

## Data model and ingestion

SQLite tables are `usage_records` (one deterministic SHA-256 source fact), `capabilities`, plus lookup indexes. Re-running ingestion uses `INSERT OR REPLACE`, so it is idempotent. Money is integer USD nanos (100,000,000 per dollar); JSON outputs convert only at the boundary.

Claude records use `actor.user_id` and email against the directory, `rbac_group_id` against groups, date from `starting_at`, input as uncached + cache read + both cache creation fields, source `total_tokens`, and requests/web-search counts. Cost rows are keyed by user/date/product/model and aggregate both cost types; decimal cents are converted exactly to nanos.

Gemini records use `labels.owner` for directory user, `project.id` for group, labels.model (falling back to SKU text), date from `usage_start_time`, SKU input/output direction, and token `usage.amount`. Requests are null. Every attribution or expected unit/SKU is validated and fails ingestion rather than disappearing.

Periods are UTC and deterministic relative to the supplied 2026-09-28 exclusive endpoint: last 7 is 2026-09-21..28, last 30 is 2026-08-29..28. ISO weeks are Monday inclusive / next Monday exclusive; months are first-of-month inclusive / next month exclusive.

## Authorization and capabilities

Identity comes only from `x-user-id` (REST) or `MERIDIAN_CALLER_USER_ID` (MCP), then resolves in the directory; roles supplied by clients are ignored. The YAML environment policy is loaded in one place. Dev lets all roles read all groups; staging/prod permit leads only their own group and admins all groups; capability writes are lead/admin in dev/staging and admin only in prod. Denials are structured `{error:"permission_denied", reason, required_scope}` and are emitted before data aggregation.

`usage_summary` and `top_users` require both `usage:read` and that `usage-connector` is enabled for the requested group. `capability_status` remains available to authorized readers and `set_capability` to authorized writers so an admin can recover from a disabled connector. Changes persist with updater and timestamp. Defaults apply only when no stored value exists.

## API and MCP

REST endpoints: `/api/health`, `/api/me`, `/api/groups`, `/api/usage/summary`, `/api/usage/top-users`, `/api/usage/weekly`, `/api/capabilities`, and `PATCH /api/capabilities/:group/:capability`. Query usage with `group`, `period`, and optionally `limit`.

The MCP server exposes exactly `usage_summary`, `top_users`, `capability_status`, and `set_capability`; its input validation follows the supplied group/capability/period contract and it independently checks authorization. It reuses core aggregation rather than duplicating business logic.

## Provisioning, packaging, and CI

`npm run provision` reads all three supplied YAML files, validates roles/scopes/groups/capabilities/host, and renders `generated/<env>/permissions-manifest.json` and `host-registration.json`. `npm run validate:provisioning` validates without writes. CI installs, type-checks, tests, renders, and validates.

See [Claude setup](claude-desktop/README.md) and [Gemini artifact](gemini-extension/README.md). They are reproducible local stdio configurations; no external registration or host execution was claimed.

## Tests, standards, and trade-offs

Tests cover exact cent-to-nano conversion, fixture user/group attribution, all requested period boundaries, authorization cases, a clean SQLite ingestion/aggregation run, and a second ingestion that proves idempotency. TypeScript strict mode, validation, deterministic fixtures, idempotent ingestion, centralized policy, structured errors, audit fields, and no committed credentials are followed. This MVP intentionally skips a framework/ORM, migrations beyond self-creating schema, production authentication, rate limiting, and rich UI polish to keep the critical path small and inspectable.

AI coding assistance was used to draft implementation and documentation. The generated result was reviewed for central policy, integer money, deterministic periods, and fixture-derived identifiers.

## Known limitations and next steps

The local MCP adapter implements stdio JSON-RPC directly rather than importing the SDK, external desktop/CLI hosts were not run here, and the UI is intentionally plain. With more time, add SDK transport conformance tests, migration tooling, integration tests against a running API, and a charting UI.

## In the real enterprise version

Replace simulated identity with SSO/OIDC/SAML plus SCIM, secrets management, organization connector registry, centralized least-privilege policy, and durable audit logs. Ingest real Claude APIs and Vertex/BigQuery billing through scheduled/event-driven jobs into a warehouse/OLAP model with reconciliation and retention controls. Add observability, rate limits, caching, connector lifecycle/versioning, production infrastructure, encryption, compliance controls, multi-tenant isolation, and failure handling.

Read first: this README, `packages/core/src/index.ts`, `scripts/ingest.ts`, `apps/api/src/server.ts`, and `docs/submission-verification.md`.
