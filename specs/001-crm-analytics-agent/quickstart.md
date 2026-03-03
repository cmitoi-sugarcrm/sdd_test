# Quickstart: SugarCRM Analytics Agent

**Branch**: `001-crm-analytics-agent`
**Stack**: Node.js 20 LTS · TypeScript · Fastify · OpenAI · MCP (`@modelcontextprotocol/sdk`) · SugarCRM REST API v11

---

## Prerequisites

| Tool | Required version | Check |
|---|---|---|
| Node.js | 20.x LTS | `node --version` |
| npm | 10.x+ | `npm --version` |
| OpenAI API key | — | Set in `.env` |
| SugarCRM instance URL | REST API v11 accessible | See _Config_ below |
| SugarCRM OAuth 2.0 bearer token | Per-request user token | Obtained via SugarCRM OAuth flow |

---

## 1. Clone and Install

```bash
git clone <repo-url>
cd <repo-root>
git checkout 001-crm-analytics-agent
npm install
```

---

## 2. Configure Environment

Copy the example env file and fill in your values:

```bash
cp .env.example .env
```

Required variables in `.env`:

```dotenv
# SugarCRM
SUGAR_BASE_URL=https://your-instance.sugarcrm.com
SUGAR_API_VERSION=v11
SUGAR_PLATFORM=analytics_agent          # Avoid session conflicts with Sugar UI
SUGAR_MAX_RECORDS=50                    # Hard cap: records per query
SUGAR_MAX_LLM_RECORDS=10               # Hard cap: records forwarded to LLM
SUGAR_MAX_SUB_REQUESTS_PER_BULK=20     # Safety limit per /bulk call
SUGAR_BULK_TIMEOUT_MS=10000

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_CLASSIFY_MODEL=gpt-4.1-mini
OPENAI_GENERATE_MODEL=gpt-4.1
OPENAI_MAX_CLASSIFY_TOKENS=256
OPENAI_MAX_GENERATE_TOKENS=1024

# MCP Server
MCP_PORT=3001
MCP_HOST=127.0.0.1

# API Server
PORT=3000
HOST=127.0.0.1
LOG_LEVEL=info                          # trace | debug | info | warn | error
NODE_ENV=development
```

---

## 3. Start Development Server

```bash
npm run dev
```

This starts:
- **API server** on `http://127.0.0.1:3000` (Fastify, hot-reload via `tsx watch`)
- **MCP server** on `http://127.0.0.1:3001` (McpServer, Streamable HTTP)

Verify both are up:

```bash
curl -s http://127.0.0.1:3000/health | jq .
# → { "status": "ok", "timestamp": "..." }

curl -s http://127.0.0.1:3001/health | jq .
# → { "status": "ok", "tools": 5 }
```

---

## 4. Make Your First Request

All requests to `POST /ask` require the SugarCRM OAuth bearer token as a header.

### Scripted question (question 1 — zombie deals)

```bash
curl -s -X POST http://127.0.0.1:3000/ask \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-sugar-token>" \
  -d '{ "question": "Show me opportunities with no interaction the last 2 weeks" }' \
  | jq .
```

Expected response shape:

```json
{
  "responseType": "scripted-analytics",
  "requestId": "req_abc123",
  "latencyMs": 1234,
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
      { "type": "outlier", "text": "Acme Renewal has been inactive for 22 days ...", "dataPoint": "22 days" },
      { "type": "notable-shift", "text": "Both deals are past their expected close date ...", "dataPoint": null }
    ],
    "dataTimestamp": "2026-03-03T10:00:00Z"
  },
  "warnings": []
}
```

### General question (table-only)

```bash
curl -s -X POST http://127.0.0.1:3000/ask \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-sugar-token>" \
  -d '{ "question": "List all open deals closing this month" }' \
  | jq .
```

Expected response shape:

```json
{
  "responseType": "general-table",
  "requestId": "req_def456",
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
      "totalCount": 1,
      "returnedCount": 1,
      "pageSize": 50,
      "pageNumber": 1,
      "hasMore": false,
      "appliedFilters": { "status": "open", "timeRange": "current calendar month" },
      "timeRange": "March 2026"
    }
  },
  "warnings": []
}
```

### Authentication error

```bash
curl -s -X POST http://127.0.0.1:3000/ask \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer expired-token" \
  -d '{ "question": "Show me win rates by product category" }' \
  | jq .
```

Expected:

```json
{
  "responseType": "error",
  "requestId": "req_ghi789",
  "latencyMs": 210,
  "payload": {
    "errorCategory": "auth",
    "message": "Your SugarCRM session has expired. Please sign in again to get a new token.",
    "suggestedActions": ["Refresh your SugarCRM OAuth token and retry the request."]
  },
  "warnings": []
}
```

---

## 5. Run Tests

```bash
# Unit tests (fast — no external dependencies)
npm run test:unit

# Integration tests (requires .env with valid SUGAR_BASE_URL and token)
npm run test:integration

# Contract tests (validates /ask response schema and MCP tool schemas)
npm run test:contract

# All tests
npm test
```

Test-first note: tests are written **before** implementation. Run `npm test` immediately after checkout — they should all **fail** until the corresponding implementation is in place. This is the expected Red state before Green.

---

## 6. Validate Test Data (Optional)

Before running analytics scenarios, verify your SugarCRM instance has data that meets the requirements for each scripted question:

```bash
npm run validate-data -- --token <your-sugar-token>
```

The validator is read-only (no writes). Output example:

```
✓ Scripted-1 (Zombie Deals): 3 qualifying opportunities found
✓ Scripted-2 (Stage Conversion): Q3 pipeline data present
✗ Scripted-3 (Win Rate by Product): Only 2 RLIs per product found (need ≥ 5)
  → Action: Ensure at least 5 Revenue Line Items per product in Closed Won/Lost opps
✓ Scripted-4 (Sales Cycle): 4 reps with ≥ 10 closed opps found
✓ Scripted-5 (Pipeline Changes): Last-week delta data calculable
```

---

## 7. Project Structure Reference

```text
src/
├── api/                  # Fastify server — routes, schemas, middleware
│   ├── server.ts         # Fastify app factory + plugin registration
│   ├── routes/ask.ts     # POST /ask handler
│   └── schemas/          # Zod schemas compiled → Fastify JSON Schema
├── agent/                # Declarative agent engine
│   ├── engine.ts         # AgentEngine.run() — generic pipeline executor
│   ├── classifier.ts     # Intent classification + parameter extraction (LLM)
│   ├── pipeline/         # Pipeline step implementations
│   │   ├── route.ts
│   │   ├── plan.ts
│   │   ├── retrieve.ts
│   │   ├── compute.ts
│   │   ├── format.ts
│   │   └── postprocess.ts
│   └── config/           # Intent config files (YAML/JSON)
│       ├── scripted-1.yaml
│       ├── scripted-2.yaml
│       ├── scripted-3.yaml
│       ├── scripted-4.yaml
│       ├── scripted-5.yaml
│       └── general.yaml
├── sugar/                # SugarCRM REST API v11 client
│   ├── client.ts         # SugarClient — read-only enforcement + bulk-first
│   ├── bulk.ts           # /bulk request builder + response parser
│   └── metadata.ts       # Module schema cache (TTL-based)
├── mcp/                  # MCP server
│   ├── server.ts         # McpServer factory + Streamable HTTP transport
│   └── tools/            # Tool handlers
│       ├── list.ts
│       ├── get.ts
│       ├── listRelated.ts
│       ├── bulkWithRelations.ts
│       └── aggregate.ts
├── analytics/            # Local aggregation + computation
│   ├── zombie-deals.ts
│   ├── stage-conversion.ts
│   ├── win-rate.ts
│   ├── sales-cycle.ts
│   └── pipeline-delta.ts
├── llm/                  # OpenAI integration
│   ├── classify.ts       # gpt-4.1-mini classification + param extraction
│   └── generate.ts       # gpt-4.1 summary + insights generation
└── validate/             # Test data validator utility (read-only)
    └── validator.ts

tests/
├── contract/             # /ask endpoint schema + MCP tool schema contracts
├── integration/          # SugarClient, bulk endpoint, full pipeline
└── unit/                 # Engine, analytics, classifier, formatter
```

---

## 8. Key npm Scripts

| Script | Purpose |
|---|---|
| `npm run dev` | Start API + MCP servers with hot-reload |
| `npm run build` | Compile TypeScript to `dist/` |
| `npm start` | Run compiled output (production) |
| `npm test` | All tests |
| `npm run test:unit` | Unit tests only |
| `npm run test:integration` | Integration tests only |
| `npm run test:contract` | Contract tests only |
| `npm run lint` | ESLint + Prettier check |
| `npm run lint:fix` | Auto-fix lint issues |
| `npm run validate-data` | Run test-data validator |
| `npm run typecheck` | TypeScript compile check (no emit) |
