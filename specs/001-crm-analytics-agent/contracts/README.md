# API Contracts: SugarCRM Analytics Agent

**Branch**: `001-crm-analytics-agent` | **Date**: 2026-03-03

This directory contains the public interface contracts for the agent API.

---

## Endpoint: `POST /ask`

The single public endpoint for the analytics agent.

### Authentication

All requests must include a SugarCRM OAuth 2.0 bearer token:

```
Authorization: Bearer <sugar-oauth-token>
```

The token is forwarded to SugarCRM on every data retrieval call. It is never logged, cached, or stored beyond the single request.

Missing or malformed `Authorization` header → HTTP `401` with `responseType: "error"`, `errorCategory: "auth"`.

### Request Headers

| Header | Required | Description |
|---|---|---|
| `Authorization` | Yes | `Bearer <sugar-oauth-token>` |
| `Content-Type` | Yes | `application/json` |
| `X-Request-ID` | No | Client-supplied request ID. If absent, server generates one. |

### Request Body

Schema: [ask-request.schema.json](./ask-request.schema.json)

```json
{
  "question": "<natural-language question>",
  "sessionId": "<uuid — optional, for follow-up context>"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `question` | string | Yes | 1–2000 characters |
| `sessionId` | string (UUID) | No | Must be a valid UUID v4 if provided |

### Response Body

Schema: [ask-response.schema.json](./ask-response.schema.json)

The `responseType` field discriminates the `payload` shape:

| `responseType` | `payload` type | When returned |
|---|---|---|
| `scripted-analytics` | `ScriptedPayload` | Question matched one of the 5 scripted intents |
| `general-table` | `TablePayload` | CRM-relevant question, not scripted |
| `clarification-needed` | `ClarificationPayload` | Ambiguous follow-up with no session context |
| `error` | `ErrorPayload` | Auth failure, API unavailable, unsupported question |

### HTTP Status Codes

| Code | Meaning |
|---|---|
| `200` | Request processed successfully (check `responseType` for outcome) |
| `400` | Request body failed schema validation (missing `question`, bad UUID, etc.) |
| `401` | `Authorization` header missing or malformed |
| `429` | Rate limit exceeded |
| `500` | Unexpected server error (logged internally; user receives error payload) |

> Note: A SugarCRM authentication failure (expired token accepted by the agent but rejected by SugarCRM) returns HTTP `200` with `responseType: "error"` and `errorCategory: "auth"` — not HTTP `401` — because the agent successfully processed the request; the error is at the CRM layer.

### Rate Limiting

Responses include standard rate-limit headers:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 58
X-RateLimit-Reset: 1709460120
```

### Full Request / Response Examples

**Scripted question — zombie deals:**

```http
POST /ask HTTP/1.1
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{
  "question": "Show me opportunities with no interaction the last 2 weeks"
}
```

```json
{
  "responseType": "scripted-analytics",
  "requestId": "550e8400-e29b-41d4-a716-446655440001",
  "latencyMs": 1340,
  "payload": {
    "scriptedIntentId": 1,
    "chart": {
      "type": "horizontal-bar",
      "title": "Opportunities with No Interaction (Last 14 Days)",
      "xAxisLabel": "Days Since Last Activity",
      "yAxisLabel": "Opportunity",
      "labels": ["Acme Renewal", "Beta Corp Expansion"],
      "datasets": [{ "label": "Days Inactive", "data": [22, 18], "color": null }]
    },
    "summary": "2 open opportunities have not been touched in over 2 weeks ...",
    "insights": [
      { "type": "outlier", "text": "Acme Renewal has been silent for 22 days.", "dataPoint": "22 days" },
      { "type": "notable-shift", "text": "Both deals are past their expected close date.", "dataPoint": null }
    ],
    "dataTimestamp": "2026-03-03T10:00:00Z"
  },
  "warnings": []
}
```

**General question — table only:**

```http
POST /ask HTTP/1.1
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{
  "question": "List all open deals closing this month"
}
```

```json
{
  "responseType": "general-table",
  "requestId": "550e8400-e29b-41d4-a716-446655440002",
  "latencyMs": 870,
  "payload": {
    "columns": [
      { "name": "name", "label": "Opportunity Name", "dataType": "string" },
      { "name": "amount", "label": "Amount", "dataType": "number" },
      { "name": "date_closed", "label": "Close Date", "dataType": "date" },
      { "name": "sales_stage", "label": "Stage", "dataType": "string" }
    ],
    "rows": [
      { "name": "Delta Corp Q1", "amount": 45000, "date_closed": "2026-03-28", "sales_stage": "Proposal" }
    ],
    "meta": {
      "totalCount": 1, "returnedCount": 1, "pageSize": 50,
      "pageNumber": 1, "hasMore": false,
      "appliedFilters": { "status": "open", "timeRange": "current calendar month" },
      "timeRange": "March 2026"
    }
  },
  "warnings": []
}
```

**Authentication error:**

```json
{
  "responseType": "error",
  "requestId": "550e8400-e29b-41d4-a716-446655440003",
  "latencyMs": 95,
  "payload": {
    "errorCategory": "auth",
    "message": "Your SugarCRM session has expired. Please sign in again.",
    "suggestedActions": ["Refresh your SugarCRM OAuth token and retry the request."]
  },
  "warnings": []
}
```

**SugarCRM API unavailable:**

```json
{
  "responseType": "error",
  "requestId": "550e8400-e29b-41d4-a716-446655440004",
  "latencyMs": 10050,
  "payload": {
    "errorCategory": "api-unavailable",
    "message": "The CRM system is temporarily unavailable. Please try again in a moment.",
    "suggestedActions": ["Wait a few seconds and retry.", "Contact your CRM administrator if the problem persists."]
  },
  "warnings": []
}
```

---

## MCP Tools Contract

The MCP server exposes the following tools to the LLM via `POST http://127.0.0.1:3001/mcp` (Streamable HTTP transport):

| Tool | Description | Key Constraints |
|---|---|---|
| `sugar.records.list` | List records from an allowlisted module | `limit` ≤ 50; allowlisted modules and fields only |
| `sugar.records.get` | Fetch a single record by ID | Allowlisted modules and fields only |
| `sugar.records.listRelated` | Fetch related records via a relationship | `limit` ≤ 10 per relation |
| `sugar.bulk.getRecordWithRelations` | Fetch a record + up to 4 relations in one `/bulk` call | ≤ 20 sub-requests total |
| `sugar.bulk.listWithRelations` | Fetch a list + relations per record in one `/bulk` call | ≤ `maxBaseRecords` base records; degrades gracefully if relation cap exceeded |

All tools:
- Are annotated `readOnlyHint: true`
- Return `{ isError: true }` for policy violations (not thrown exceptions)
- Include `truncated: boolean` and `meta` in every response

Full tool input/output schemas are defined in `src/mcp/tools/*.ts` and discoverable via the MCP tools/list endpoint.
