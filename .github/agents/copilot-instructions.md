# spec_driven_test Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-03-03

## Active Technologies

- **Runtime**: Node.js 20 LTS · TypeScript (strict)
- **HTTP Framework**: Fastify 4.x (`@fastify/rate-limit`, `@fastify/env`, `pino`)
- **LLM**: OpenAI `gpt-4.1-mini` (classification) · `gpt-4.1` (generation) via `openai` npm SDK v4.x
- **MCP**: `@modelcontextprotocol/sdk` v1.27.1 · Streamable HTTP transport · `McpServer` high-level API
- **CRM Integration**: SugarCRM REST API v11 · `/bulk` POST endpoint · OAuth 2.0 Bearer token (per-request header)
- **Validation**: `zod` v3 · `zod-to-json-schema` for Fastify route schemas
- **Testing**: Jest · contract tests · integration tests · unit tests

## Project Structure

```text
src/
├── api/                  # Fastify server — routes, schemas, middleware
│   ├── server.ts
│   ├── routes/ask.ts
│   └── schemas/
├── agent/                # Declarative agent engine
│   ├── engine.ts
│   ├── classifier.ts
│   ├── pipeline/         # route, plan, retrieve, compute, format, postprocess
│   └── config/           # Intent config files (YAML/JSON)
├── sugar/                # SugarCRM REST API v11 client (read-only enforced)
│   ├── client.ts
│   ├── bulk.ts
│   └── metadata.ts
├── mcp/                  # MCP server (Streamable HTTP)
│   ├── server.ts
│   └── tools/
├── analytics/            # Local aggregation computations
├── llm/                  # OpenAI classify.ts + generate.ts
└── validate/             # Test data validator (read-only)

tests/
├── contract/
├── integration/
└── unit/
```

## Commands

```bash
npm run dev           # Start API + MCP servers (hot-reload)
npm run build         # Compile TypeScript
npm start             # Run compiled output
npm test              # All tests
npm run test:unit
npm run test:integration
npm run test:contract
npm run lint
npm run lint:fix
npm run typecheck
npm run validate-data -- --token <sugar-token>
```

## Code Style

- TypeScript strict mode (`"strict": true` in tsconfig)
- Zod schemas are the single source of truth for all data shapes — compile to JSON Schema for Fastify routes and MCP tool schemas
- All SugarCRM calls go through `SugarClient` — no direct `fetch` to Sugar outside that module
- MCP tool handlers: return `{ isError: true, content: [...] }` for policy violations — never throw
- No `any` types without an inline `// reason:` comment
- Pino logger — never log question text, parameter values, CRM field values, or tokens
- SugarCRM `/bulk` `data` field must be `JSON.stringify(payload)` — a string, not an object

## Recent Changes

- `001-crm-analytics-agent` (2026-03-03): Initial feature — SugarCRM Analytics Agent. Added Fastify API layer, declarative agent engine, OpenAI gpt-4.1 tiered LLM integration, MCP server with 5 bulk-enabled tools, SugarCRM REST API v11 bulk-first read-only client, 5 scripted intents + general table mode, test-data validator utility.

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
