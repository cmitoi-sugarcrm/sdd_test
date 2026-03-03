# Data Model: SugarCRM Analytics Agent

**Branch**: `001-crm-analytics-agent` | **Date**: 2026-03-03
**Source**: spec.md entities + research.md decisions

All entities below are runtime data structures (TypeScript interfaces). None represent database tables — the service is stateless and stores nothing.

---

## 1. Request / Input

### `AgentRequest`
The incoming payload POSTed to `POST /ask`.

| Field | Type | Required | Description |
|---|---|---|---|
| `question` | `string` | Yes | Raw natural-language question from the user |
| `sessionId` | `string` (UUID) | No | Opaque session identifier for follow-up context. Omit for stateless requests |

---

### `ParsedQuestion`
The enriched question after intent classification and parameter extraction (LLM step 1).

| Field | Type | Required | Description |
|---|---|---|---|
| `raw` | `string` | Yes | Original question text |
| `intent` | `IntentType` | Yes | Classification result |
| `scriptedIntentId` | `1\|2\|3\|4\|5 \| null` | Yes | Set when `intent === 'scripted'`; null otherwise |
| `parameters` | `ParameterSet` | Yes | Extracted query parameters |
| `confidence` | `number` (0–1) | Yes | Classifier confidence score |
| `sessionId` | `string \| null` | Yes | Passed through from request |

#### `IntentType` (enum)
```
'scripted' | 'general' | 'clarification-needed' | 'unsupported'
```

---

### `ParameterSet`
Structured parameters extracted from the question.

| Field | Type | Required | Description |
|---|---|---|---|
| `timeWindow` | `TimeWindow \| null` | Yes | Resolved date range |
| `teamScope` | `TeamScope \| null` | Yes | Resolved team or user |
| `stageFilter` | `string[] \| null` | Yes | Pipeline stage names |
| `productCategory` | `string \| null` | Yes | Product category name |
| `changeDirection` | `'added'\|'advanced'\|'lost'\|'all' \| null` | Yes | Pipeline change filter |

#### `TimeWindow`
| Field | Type | Description |
|---|---|---|
| `startDate` | `string` (ISO 8601 date) | Inclusive start |
| `endDate` | `string` (ISO 8601 date) | Inclusive end |
| `label` | `string` | Human-readable label, e.g. "Q3 2025", "last 14 days" |

#### `TeamScope`
| Field | Type | Description |
|---|---|---|
| `type` | `'user'\|'team'` | Scope kind |
| `id` | `string` | SugarCRM user or team ID (resolved from token at query time) |
| `label` | `string` | Display name |

---

## 2. Response / Output

### `AgentResponse`
The unified response envelope returned by `POST /ask`.

| Field | Type | Required | Description |
|---|---|---|---|
| `responseType` | `ResponseType` | Yes | Discriminator for the response shape |
| `requestId` | `string` (UUID) | Yes | Server-generated ID for observability |
| `latencyMs` | `number` | Yes | End-to-end processing latency |
| `payload` | `ScriptedPayload \| TablePayload \| ClarificationPayload \| ErrorPayload` | Yes | Typed by `responseType` |
| `warnings` | `string[]` | Yes | Non-fatal notices (e.g., partial data, truncation). Empty array if none |

#### `ResponseType` (enum)
```
'scripted-analytics' | 'general-table' | 'clarification-needed' | 'error'
```

---

### `ScriptedPayload`
Returned when `responseType === 'scripted-analytics'`.

| Field | Type | Required | Description |
|---|---|---|---|
| `scriptedIntentId` | `1\|2\|3\|4\|5` | Yes | Which of the 5 scripted questions was matched |
| `chart` | `ChartPayload` | Yes | Renderable chart data |
| `summary` | `string` | Yes | LLM-generated narrative highlight (plain language) |
| `insights` | `GraphInsight[]` | Yes | LLM-generated insight bullets (≥ 2 when data present; empty array when no data) |
| `dataTimestamp` | `string` (ISO 8601) | Yes | Time the CRM data was fetched |

---

### `ChartPayload`
Self-describing chart data renderable by any standard charting library.

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `ChartType` | Yes | Chart rendering type |
| `title` | `string` | Yes | Chart title |
| `xAxisLabel` | `string \| null` | Yes | X-axis label (null for pie/donut) |
| `yAxisLabel` | `string \| null` | Yes | Y-axis label (null for pie/donut) |
| `labels` | `string[]` | Yes | Category labels (aligned by index with series data) |
| `datasets` | `ChartDataset[]` | Yes | One or more data series |

#### `ChartType` (enum)
```
'bar' | 'horizontal-bar' | 'line' | 'pie' | 'donut' | 'scatter'
```

#### `ChartDataset`
| Field | Type | Required | Description |
|---|---|---|---|
| `label` | `string` | Yes | Series name |
| `data` | `number[]` | Yes | Values aligned with `ChartPayload.labels` |
| `color` | `string \| null` | No | Hex color hint for the renderer |

---

### `GraphInsight`
A single noteworthy observation derived from interpreting the data.

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | `InsightType` | Yes | Pattern category |
| `text` | `string` | Yes | Plain-language description |
| `dataPoint` | `string \| null` | Yes | The specific value or entity referenced |

#### `InsightType` (enum)
```
'outlier' | 'top-performer' | 'bottom-performer' | 'largest-change' | 'notable-shift' | 'no-data'
```

---

### `TablePayload`
Returned when `responseType === 'general-table'`.

| Field | Type | Required | Description |
|---|---|---|---|
| `columns` | `TableColumn[]` | Yes | Column definitions |
| `rows` | `Record<string, unknown>[]` | Yes | Data rows (keyed by column `name`) |
| `meta` | `TableMeta` | Yes | Pagination and filter metadata |

#### `TableColumn`
| Field | Type | Description |
|---|---|---|
| `name` | `string` | Machine key (matches row object keys) |
| `label` | `string` | Human-readable column header |
| `dataType` | `'string'\|'number'\|'date'\|'boolean'` | Display type hint |

#### `TableMeta`
| Field | Type | Description |
|---|---|---|
| `totalCount` | `number` | Total matching records in CRM (before pagination) |
| `returnedCount` | `number` | Records in this response |
| `pageSize` | `number` | Page size applied (default 50) |
| `pageNumber` | `number` | Current page (1-based) |
| `hasMore` | `boolean` | Whether more pages exist |
| `appliedFilters` | `Record<string, string>` | Filters applied (display only, no CRM field names) |
| `timeRange` | `string \| null` | Human-readable time range applied |

---

### `ClarificationPayload`
Returned when `responseType === 'clarification-needed'`.

| Field | Type | Required | Description |
|---|---|---|---|
| `message` | `string` | Yes | Question to the user asking for clarification |
| `ambiguousContext` | `string` | Yes | What was ambiguous (no CRM data values) |
| `examples` | `string[]` | Yes | 1–3 example phrasings the user could try |

---

### `ErrorPayload`
Returned when `responseType === 'error'`.

| Field | Type | Required | Description |
|---|---|---|---|
| `errorCategory` | `ErrorCategory` | Yes | Machine-readable error kind |
| `message` | `string` | Yes | User-readable plain-language description |
| `suggestedActions` | `string[]` | Yes | At least one actionable suggestion |

#### `ErrorCategory` (enum)
```
'auth' | 'no-data' | 'unsupported-question' | 'partial-data' | 'api-unavailable'
```

---

## 3. Internal Agent Structures

### `IntentConfig`
Config block per intent (scripted or general constraints) loaded at startup.

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique intent identifier (e.g., `"scripted-1"`, `"general"`) |
| `type` | `'scripted'\|'general'` | Intent kind |
| `matchPatterns` | `string[]` | Semantic match patterns (scripted only) |
| `allowedModules` | `string[]` | SugarCRM modules this intent may read |
| `allowedFields` | `Record<string, string[]>` | Per-module field allowlist |
| `allowedRelationships` | `string[]` | Relationship names the intent may traverse |
| `maxRecords` | `number` | Hard cap on records retrieved |
| `maxLlmRecords` | `number` | Hard cap on records forwarded to LLM |
| `pipeline` | `PipelineStep[]` | Ordered pipeline steps |
| `responseSchema` | `object` | JSON Schema for the output payload |

---

### `PipelineStep`
A single step in the declarative execution pipeline.

| Field | Type | Description |
|---|---|---|
| `name` | `'route'\|'plan'\|'retrieve'\|'compute'\|'format'\|'postprocess'` | Step identifier |
| `config` | `object` | Step-specific configuration (transformation instructions, aggregation type, etc.) |

---

### `QueryPlan`
Built by the `plan` step; describes what data to fetch.

| Field | Type | Description |
|---|---|---|
| `subRequests` | `BulkSubRequest[]` | List of `/bulk` sub-requests to execute |
| `aggregations` | `AggregationSpec[]` | Computation instructions to apply after retrieval |

#### `BulkSubRequest`
| Field | Type | Description |
|---|---|---|
| `url` | `string` | Version-relative SugarCRM URL (e.g., `/v11/Opportunities?fields=...`) |
| `method` | `'GET'` | Always GET (read-only) |
| `role` | `string` | Logical label (e.g., `"base-list"`, `"related-accounts"`) used to index results |

#### `AggregationSpec`
| Field | Type | Description |
|---|---|---|
| `type` | `'group-by'\|'top-n'\|'delta'\|'win-rate'\|'avg-duration'` | Aggregation kind |
| `config` | `object` | Type-specific parameters (group field, N, date fields, etc.) |

---

### `SugarCRMMetadata`
Cached module schema loaded from `/rest/v11/<Module>` metadata endpoint.

| Field | Type | Description |
|---|---|---|
| `module` | `string` | Module name |
| `fields` | `FieldMeta[]` | Available fields |
| `relationships` | `RelationshipMeta[]` | Available relationships |
| `cachedAt` | `string` (ISO 8601) | Cache timestamp (TTL enforced at runtime) |

#### `FieldMeta`
| Field | Type | Description |
|---|---|---|
| `name` | `string` | API field name |
| `label` | `string` | Human-readable label |
| `type` | `string` | Sugar field type (varchar, currency, relate, etc.) |
| `alias` | `string \| null` | Configured alias (e.g., `last_interaction_date`) |

---

### `SessionContext`
Maintained per `sessionId` for the duration of a user session.

| Field | Type | Description |
|---|---|---|
| `sessionId` | `string` | Opaque session identifier |
| `history` | `SessionTurn[]` | Ordered list of prior question/response pairs |
| `createdAt` | `string` (ISO 8601) | Session creation time |
| `lastActivityAt` | `string` (ISO 8601) | Last question time (used for TTL) |

#### `SessionTurn`
| Field | Type | Description |
|---|---|---|
| `question` | `string` | Raw question text (retained for context resolution only) |
| `parsedParameters` | `ParameterSet` | Parameters resolved for this turn |
| `responseType` | `ResponseType` | What kind of response was returned |

---

## 4. Scripted Intent Reference

| ID | Canonical Question | Key Modules | Key Metric |
|---|---|---|---|
| 1 | "Show me opportunities with no interaction the last 2 weeks" | Opportunities, Calls, Meetings, Tasks, Emails | Days since last activity |
| 2 | "Break down my Q3 pipeline by stage conversion rates" | Opportunities | Stage conversion % |
| 3 | "Show me win rates by product category" | Opportunities, RevenueLineItems | Win rate % per product |
| 4 | "Who has the shortest average sales cycle in my team?" | Opportunities, Users | Avg cycle days per rep |
| 5 | "What's changed in the pipeline this past week?" | Opportunities | Delta: added / advanced / lost |

---

## 5. Validation Rules

- `AgentRequest.question` — non-empty string, max 2000 characters
- `ParameterSet.timeWindow.startDate` — must not be in the future
- `ParameterSet.timeWindow.endDate` — must be ≥ `startDate`
- `ChartPayload.labels` — must have the same length as every `ChartDataset.data` array
- `TablePayload.rows` — every row object must have a key for every `TableColumn.name` (missing values allowed as `null`)
- `ErrorPayload.suggestedActions` — minimum 1 item
- `GraphInsight[]` — minimum 2 items when `ScriptedPayload.chart.datasets[n].data` is non-empty
- `BulkSubRequest.method` — must always be `'GET'`; any other value is a hard error at query plan time
- Any module not present in the intent's `allowedModules` list — rejected at query plan time with `error-category: unsupported-question`
