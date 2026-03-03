# Implementation Plan: SugarCRM Analytics Agent

**Branch**: `001-crm-analytics-agent` | **Date**: 2026-03-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-crm-analytics-agent/spec.md`

## Summary

A config-driven, declarative Node.js analytics agent that answers natural-language questions about SugarCRM CRM data. Authenticated users POST a question to a single `/ask` endpoint; the agent classifies the intent (one of 5 scripted questions or general/ad-hoc), retrieves CRM data using the SugarCRM REST API v11 `/bulk` endpoint in a strictly read-only posture, performs local aggregations, and optionally invokes an LLM (OpenAI `gpt-4.1`) to produce a narrative summary and graph insights. Scripted questions return a chart payload + summary + graph insights; non-scripted questions return a table-only payload. An MCP server exposes safe, bounded tools for ad-hoc LLM-driven data retrieval.

## Technical Context

**Language/Version**: Node.js 20 LTS · TypeScript (strict mode)
**Primary Dependencies**: Fastify 4.x · OpenAI SDK v4.x (`gpt-4.1-mini` + `gpt-4.1`) · `@modelcontextprotocol/sdk` v1.27.1 · `zod` v3 · SugarCRM REST API v11
**Storage**: N/A — stateless service; no persistence beyond request lifetime. Session context held in-memory (TTL-bounded).
**Testing**: Jest · contract tests (`tests/contract/`) · integration tests (`tests/integration/`) · unit tests (`tests/unit/`)
**Target Platform**: Node.js 20 LTS server · Linux (Docker-compatible)
**Project Type**: Web service (single-endpoint API) + MCP server
**Performance Goals**: Scripted questions: end-to-end p95 < 3s (10k CRM records); general questions: p95 < 5s (50k records); framework overhead contribution < 5ms per request (Fastify baseline)
**Constraints**: Hard record cap `SUGAR_MAX_RECORDS` (default 50) on all CRM queries; hard LLM payload cap `SUGAR_MAX_LLM_RECORDS` (default 10); max 20 sub-requests per `/bulk` call; max LLM token budgets enforced per call type; no retry on SugarCRM API failures (fail-fast, structured error); all errors return structured `AgentResponse` — never raw exceptions.
**Scale/Scope**: Single-tenant per token; designed for team/org-scale usage (50–500 daily active users). Rate limiting per token.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate | Status | Notes |
|---|---|---|---|
| I. Code Quality Excellence | Linting + static analysis enforced in CI (zero warnings); TypeScript strict; Zod for all public shapes; single-responsibility module layout | ✅ PASS | ESLint + TypeScript strict; Zod schemas are source of truth; modules have clear single responsibilities (api ∕ agent ∕ sugar ∕ mcp ∕ analytics ∕ llm) |
| I. Code Quality Excellence | No dead code; descriptive naming; type annotations on all public interfaces | ✅ PASS | All public interfaces typed via Zod + TypeScript; no `any` without comment |
| II. Test-First Development | TDD Red-Green-Refactor; tests written first and verified to fail | ✅ PASS | quickstart.md documents Red state expectation; Milestone 7 includes full test suites |
| II. Test-First Development | ≥ 80% line coverage; 100% branch coverage on critical paths (auth, data mutation) | ✅ PASS | Auth handling + read-only enforcement are critical paths; 100% branch coverage required on both |
| II. Test-First Development | All three test tiers present: unit / integration / contract | ✅ PASS | `tests/contract/`, `tests/integration/`, `tests/unit/` all in project structure |
| II. Test-First Development | CI gate: full suite must pass before merge | ✅ PASS | Milestone 7 acceptance criterion: all guardrails covered by tests |
| III. User Experience Consistency | All errors are actionable plain-language + recovery path; no raw errors to users | ✅ PASS | FR-017/FR-020: all error responses use `ErrorPayload` with `errorCategory`, `message`, `suggestedActions`. FR-017 explicitly prohibits surfacing raw API errors |
| III. User Experience Consistency | Consistent response schema; single renderer handles all response types | ✅ PASS | Single `AgentResponse` envelope with `responseType` discriminator; contracts defined in `contracts/ask-response.schema.json` |
| III. User Experience Consistency | Loading/empty states handled | ✅ PASS | Empty chart payload + "no data" summary path defined (FR-009); `warnings[]` array on every response |
| IV. Performance Standards | API p95 latency targets met; regression tracked in CI | ✅ PASS | SC-001 (3s scripted), SC-003 (5s general) are measurable; Milestone 7 includes performance benchmarks in CI |
| IV. Performance Standards | Memory bounded; no full table scans on large datasets | ✅ PASS | Hard record caps (SUGAR_MAX_RECORDS, SUGAR_MAX_LLM_RECORDS) prevent unbounded data loads; all queries use fields+limit params |
| IV. Performance Standards | Structured metrics exposed; alerts on threshold breach | ✅ PASS | NFR-003: per-request latency metric exposed; NFR-004: JSON structured logs for tooling |

**Result: All gates PASS. No violations. Proceed to Phase 0.**

## Project Structure

### Documentation (this feature)

```text
specs/001-crm-analytics-agent/
├── plan.md              ← this file
├── research.md          ← Phase 0 output
├── data-model.md        ← Phase 1 output
├── quickstart.md        ← Phase 1 output
├── contracts/
│   ├── README.md
│   ├── ask-request.schema.json
│   └── ask-response.schema.json
└── tasks.md             ← Phase 2 output (/speckit.tasks — NOT created here)
```

### Source Code (repository root)

```text
src/
├── api/
│   ├── server.ts              # Fastify app factory; plugin registration (rate-limit, env, pino)
│   ├── routes/
│   │   └── ask.ts             # POST /ask handler
│   └── schemas/
│       ├── ask-request.ts     # Zod schema → Fastify JSON Schema for request validation
│       └── ask-response.ts    # Zod schema for response serialization
├── agent/
│   ├── engine.ts              # AgentEngine.run() — generic pipeline executor
│   ├── classifier.ts          # gpt-4.1-mini: intent classification + parameter extraction
│   ├── pipeline/
│   │   ├── route.ts           # Step 1: map intent → config
│   │   ├── plan.ts            # Step 2: build QueryPlan (bulk sub-requests + aggregations)
│   │   ├── retrieve.ts        # Step 3: execute /bulk calls via SugarClient
│   │   ├── compute.ts         # Step 4: run local aggregations
│   │   ├── format.ts          # Step 5: shape output per intent response schema
│   │   └── postprocess.ts     # Step 6: LLM summary + insights (scripted only)
│   └── config/
│       ├── scripted-1.yaml    # Zombie deals (no interaction 14 days)
│       ├── scripted-2.yaml    # Q3 pipeline stage conversion
│       ├── scripted-3.yaml    # Win rate by product category
│       ├── scripted-4.yaml    # Shortest sales cycle (team)
│       ├── scripted-5.yaml    # Pipeline changes past week
│       └── general.yaml       # General query constraints + defaults
├── sugar/
│   ├── client.ts              # SugarClient — read-only enforcement (GET only; denylist write verbs + reports endpoints)
│   ├── bulk.ts                # /bulk request builder, double-JSON-serialisation, per-item status check, partial-failure handling
│   └── metadata.ts            # Module schema cache (TTL, field alias resolution, stage synonyms)
├── mcp/
│   ├── server.ts              # McpServer factory + Streamable HTTP transport (enableJsonResponse: true)
│   └── tools/
│       ├── list.ts            # sugar.records.list (limit ≤ 50)
│       ├── get.ts             # sugar.records.get
│       ├── listRelated.ts     # sugar.records.listRelated (limit ≤ 10 per relation)
│       ├── bulkWithRelations.ts # sugar.bulk.getRecordWithRelations + sugar.bulk.listWithRelations
│       └── aggregate.ts       # sugar.analytics.aggregate (server-side; returns top-N / buckets)
├── analytics/
│   ├── zombie-deals.ts        # No-interaction deals: Opportunities × Activities cross-reference
│   ├── stage-conversion.ts    # Q3 stage-to-stage conversion rates
│   ├── win-rate.ts            # Win rate per product category (Opps × RLIs)
│   ├── sales-cycle.ts         # Avg cycle days per rep (date_entered → date_closed)
│   └── pipeline-delta.ts      # Week-over-week pipeline additions / advances / losses
├── llm/
│   ├── classify.ts            # gpt-4.1-mini; structured JSON output; intent + ParameterSet extraction
│   └── generate.ts            # gpt-4.1; summary + GraphInsight[] from aggregated dataset
└── validate/
    └── validator.ts           # Read-only test-data validator (scenario-by-scenario pass/fail)

tests/
├── contract/                  # /ask request+response schema contract tests; MCP tool schema contracts
├── integration/               # SugarClient bulk endpoint; full pipeline end-to-end with fixtures
└── unit/                      # AgentEngine steps; analytics functions; classifier; formatter; error paths
```

**Structure Decision**: Single-project layout (Option 1), no frontend. The MCP server and API server are co-located in the same Node.js process (separate Fastify + McpServer instances on different ports), sharing the `SugarClient` and `analytics/` modules. This avoids network overhead between components and simplifies deployment for an initial release.

## Complexity Tracking

> No constitution violations. This section intentionally empty.

## Implementation Milestones (Delivery Order)

The milestones below directly correspond to the user's architecture plan. Each milestone is independently verifiable.

| Milestone | Focus | Key Acceptance Criterion |
|---|---|---|
| M1 — Foundations | Skeleton, config loader, read-only enforcement, structured errors, record limits, logging | Service refuses any non-GET Sugar attempt; truncation metadata returned |
| M2 — Metadata Awareness | `/bulk` client, module schema cache, field aliases, stage synonyms, query plan builder | Config aliases resolve; query plans never include disallowed fields/modules |
| M3 — Declarative Engine | Generic pipeline executor; handler registry from config; no custom code per intent | New intent requires config change only |
| M4 — Scripted Intents | All 5 scripted questions: data fetch + local aggregation + LLM summary/insights | Consistent schema returned; insights derived from dataset not raw records |
| M5 — General Questions | Ad-hoc table-only pipeline; LLM for intent parsing only; no summary in response | Non-scripted questions return table-only; no LLM call for formatting |
| M6 — MCP Server | 5 safe tools; module/field/relationship allowlists; limit enforcement; server-side aggregate | LLM can only retrieve bounded allowlisted data; exceeding limits rejected |
| M7 — Hardening | Rate limiting, circuit breaker (LLM), metadata caching, full test suites, perf benchmarks | Stable under load; all guardrails covered by tests |
| M8 — Test Data Validator | Read-only validator utility for 5 analytics scenarios + env-configurable field aliases | Validator reports pass/fail with actionable reasons per scenario |

