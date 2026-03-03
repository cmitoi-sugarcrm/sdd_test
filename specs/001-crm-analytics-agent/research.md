# Research: SugarCRM Analytics Agent

**Branch**: `001-crm-analytics-agent` | **Date**: 2026-03-03
**Phase**: 0 — resolves all NEEDS CLARIFICATION items from Technical Context

---

## LLM Provider

**Decision**: OpenAI, tiered model strategy.
- Classification & parameter extraction: `gpt-4.1-mini`
- Narrative summary and graph-insights generation: `gpt-4.1`

**Rationale**:
- `response_format: { type: "json_schema", json_schema: { strict: true } }` enforces valid JSON output at the API level — critical for parameter extraction (time windows, team scope, pipeline stages). Anthropic's equivalent is tool-use shaping, which is less strict.
- Native MCP support in OpenAI's Responses API: `tools: [{ type: "mcp", server_url: "..." }]`. No external orchestration layer needed.
- Prompt caching: cached input tokens cost $0.50/1M (vs $2.00/1M uncached on `gpt-4.1`) — highly effective since every request repeats the same system prompt and CRM schema definitions.
- `gpt-4.1-mini` costs $0.40 in / $1.60 out per 1M tokens — well-suited for high-frequency classification calls.
- `gpt-4.1` costs $2.00 in / $8.00 out per 1M (with caching applied to system prompt).

**Alternatives considered**:
- Anthropic Claude 3.7 Sonnet: strong narrative quality but 2.5× more expensive for generation; JSON schema enforcement weaker (requires tool-call shaping); MCP available but beta. Viable as fallback; rejected as primary.
- Azure OpenAI: surfaces the same models but adds deployment management overhead, regional availability lag for new models, and Azure margin. Recommended only if hard compliance requirements (VNet, data residency) apply. SDK is identical to direct OpenAI — switchable without code changes.

**Node.js SDK**: `npm install openai` (v4.96+ as of early 2026).

---

## HTTP Framework

**Decision**: Fastify.

**Rationale**:
- 2–4× faster throughput than Express (Fastify ~75k req/s vs Express ~15–20k req/s on simple JSON routes). Lower per-request overhead reduces framework contribution to p95 latency.
- Schema-based serialization via `fast-json-stringify` (compiled from route `response` schema) — 2–5× faster than `JSON.stringify` for known-shape responses. Every `/ask` response has a fixed schema.
- First-class TypeScript generics for request body, headers, and reply — no manual `req.body as T` casts needed.
- Native `ajv` JSON Schema validation per route replaces a separate validation library.
- `pino` logger bundled; every request gets a `reqId` automatically, with built-in redaction rules — directly satisfies the spec's logging requirement (timestamp, request ID, response type, latency, error category; no CRM data).

**Alternatives considered**:
- Express v4/v5: mature ecosystem; types community-maintained; no built-in schema validation or structured logging. Valid choice but requires assembling 5–6 packages to reach Fastify's baseline. Rejected for new greenfield work.
- Hono: excellent TypeScript DX, edge-compatible; thinner rate-limiting and structured-logging ecosystem on Node.js. Worth revisiting if edge deployment needed.
- NestJS: wraps Fastify/Express with DI framework; justified at 50+ endpoints; overkill for a single-endpoint service.

**Package stack**:
```
fastify                   # framework core
@fastify/rate-limit       # rate limiting (Redis-backed for multi-instance)
@fastify/env              # typed env-var validation at startup
pino                      # logging (bundled; configure redaction here)
pino-pretty               # dev-only pretty-print
zod                       # contract models (shared with MCP)
zod-to-json-schema        # compile Zod schemas → Fastify JSON Schema for routes
```

**Gotcha**: Fastify's plugin scoping uses a DAG model (`fastify-plugin` / `fp()`). Shared decorators/hooks must be wrapped in `fp()` to be visible across scopes. Budget 30–60 min on project setup to read the plugin guide.

---

## MCP Server

**Decision**: `@modelcontextprotocol/sdk` v1.27.1, `McpServer` high-level API, Streamable HTTP transport with `enableJsonResponse: true`.

**Rationale**:
- `McpServer.registerTool(name, config, handler)` with Zod input/output schemas converts automatically to JSON Schema for the protocol — no duplicate schema definitions.
- `annotations: { readOnlyHint: true }` on every tool signals to the LLM that tools have no side effects.
- `outputSchema` + `structuredContent` in handler return enables validated structured output per tool call.
- Streamable HTTP (`NodeStreamableHTTPServerTransport`) supports multiple clients, scales behind a load balancer, and is stateless-capable via `sessionIdGenerator: undefined`. HTTP+SSE (legacy) deprecated since protocol rev 2025-03-26.
- `enableJsonResponse: true` removes SSE overhead for pure request/response patterns (all CRM queries are synchronous).

**Tool call limits**: no built-in session-level counter in the SDK. Enforce max tool calls at the LLM orchestrator layer (agent loop counter; pass `tool_choice: "none"` after N rounds). Per-tool hard caps enforced in handler via Zod schema `.max()` constraints (visible to LLM as JSON Schema) plus runtime guard.

**Safety pattern**: return `{ isError: true, content: [...] }` for policy violations — not thrown exceptions. Lets the LLM reason about the failure.

**Package**: `npm install @modelcontextprotocol/sdk zod` — MCP SDK ships Zod as a peer dep.

---

## SugarCRM REST API v11 `/bulk` Endpoint

**Decision**: Use `POST /rest/v11/bulk` as the default multi-call mechanism. Apply ≤ 20 sub-requests per call based on community practice (no documented hard limit; bounded by web server request timeout).

**Key facts**:
| Fact | Detail |
|---|---|
| URL | `POST https://<instance>/rest/v11/bulk` |
| Outer method | Always POST — even for read-only sub-requests |
| Sub-request `url` format | Version-relative: `/v11/Opportunities/<id>?fields=...` |
| Sub-request `data` field | **JSON-encoded string** (double-serialised): `"data": "{\"key\":\"val\"}"` |
| Auth | Top-level `Authorization: Bearer <token>` inherited by all sub-requests |
| Execution order | Sequential, index-aligned |
| Partial failures | Non-fatal — outer HTTP always `200 OK`; check each item's `status` field |
| Query parameters | Append to sub-request `url` (e.g., `?fields=id,name&max_num=50`) |

**Gotchas**:
- `data` field must be a JSON **string**, not an object — `JSON.stringify(payload)` required on inner body before embedding.
- `url` is version-relative, not host-relative (`/v11/...`, not `/rest/v11/...`).
- Outer HTTP response is always `200 OK` even on total failure — per-item `status` must be checked.
- No transaction semantics — no rollback across sub-requests.
- Use a dedicated `platform` query param on OAuth token acquisition (e.g., `platform=analytics_agent`) to avoid conflicting with logged-in Sugar UI sessions.

**Node.js bulk safety limits** (all configurable):
| Limit | Default (configurable) | Purpose |
|---|---|---|
| `maxBaseRecords` | 50 | Max records in a list sub-request |
| `maxRelationRecordsPerBase` | 10 | Max related records per relation sub-request |
| `maxRelationsPerRequest` | 4 | Max relation sub-requests per base record |
| `maxSubRequestsPerBulk` | 20 | Total sub-requests per `/bulk` call |
| `bulkTimeoutMs` | 10000 | Hard timeout for entire bulk call |

**Partial failure handling**: inspect per-item `status`; treat non-200 items as soft errors; continue with available data and attach a `warnings[]` array to the response.

---

## Node.js Version

**Decision**: Node.js 20 LTS (20.x).

**Rationale**: Current Active LTS line (LTS ends April 2026; Node 22 enters Active LTS April 2025, so Node 22 is also available). Node 20 is the safe production default with broad hosting support. Upgrade to Node 22 is straightforward when available on target platform.

---

## Declarative Agent Engine Pattern

**Decision**: Config-driven pipeline with a fixed execution protocol: `route → plan → retrieve → compute → format → postprocess`.

**Pattern**:
- Each intent is a YAML/JSON config object defining: data requirements (modules, fields, filters), transformation steps (group-by, top-N, delta), output schema mapping, and LLM instructions (system prompt fragment, token budget).
- A single generic `AgentEngine.run(intent, parameters, token)` function iterates the pipeline steps. No hand-coded handler per intent.
- Adding a new scripted intent = adding a config block. No new orchestration code.

**Scripted intents bypass MCP tools**: predefined data-fetch logic is executed directly by the `SugarClient` in the aggregate computation layer. MCP tools are used primarily for general (ad-hoc) questions where the LLM drives the data retrieval strategy.

**LLM invoked only for**:
1. Intent classification + parameter extraction (all question types; `gpt-4.1-mini`)
2. Summary + graph insights generation (scripted intents only; `gpt-4.1`, input = aggregated dataset, never raw records)

**LLM NOT invoked for**:
- Final table formatting for general questions (pure Node.js transformation)
- Scripted data retrieval (deterministic, config-driven)
