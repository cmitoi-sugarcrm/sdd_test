# Feature Specification: SugarCRM Analytics Agent

**Feature Branch**: `001-crm-analytics-agent`  
**Created**: 2026-03-03  
**Status**: Draft  
**Input**: User description: "Build a SugarCRM Analytics Agent that answers natural-language questions about SugarCRM CRM data to help sales leaders and reps understand pipeline health, performance, and changes over time."

## Clarifications

### Session 2026-03-03

- Q: Do sales reps and sales leaders have different data visibility within the agent (e.g., rep sees own records only, leader sees full team)? → A: Data scope is fully governed by the SugarCRM access-control settings; the agent enforces no extra role-based logic.
- Q: What should the agent do when the SugarCRM API is unavailable, times out, or returns a server error mid-query? → A: Return a structured error immediately on any API failure; no retry logic in the agent.
- Q: How is the user-provided SugarCRM authentication token transmitted to the agent? → A: Sent as a request header on every agent API call.
- Q: Which SugarCRM API version and protocol must the agent use? → A: SugarCRM REST API v11 (current stable, OAuth 2.0 bearer token).
- Q: What must the agent log for observability, and what must it never log? → A: Structured logging of request/response lifecycle only — timestamp, request ID, response type, latency, and error category if any. No question text, extracted parameters, or CRM data values.

## User Scenarios & Testing *(mandatory)*

### User Story 1 – Scripted Analytics Question (Priority: P1)

A sales rep or leader types one of the five predefined analytics questions
(or a natural-language variant thereof) into the agent interface. The agent
recognises the intent, retrieves the relevant CRM data using the user's
authentication token, and returns a complete, decision-ready response
consisting of a chart, a narrative summary, and a set of graph insights —
all in a single, consistently structured response that a UI can render
without further transformation.

**Why this priority**: This is the core value proposition. Delivering
structured, insight-rich answers to the five most critical sales questions
immediately demonstrates value and serves as the foundation for all other
capabilities. Without this, the agent provides no differentiated benefit
over a standard CRM dashboard.

**Independent Test**: Can be fully tested by submitting any of the five
scripted questions with a valid authentication token and verifying that the
response contains a non-empty chart payload, a non-empty summary, and at
least one graph insight — without any other agent features being present.

**Scripted questions covered by this story**:
1. "Show me opportunities with no interaction the last 2 weeks."
2. "Break down my Q3 pipeline by stage conversion rates."
3. "Show me win rates by product category."
4. "Who has the shortest average sales cycle in my team?"
5. "What's changed in the pipeline this past week?"

**Acceptance Scenarios**:

1. **Given** a sales rep is authenticated and asks "Show me opportunities with
   no interaction the last 2 weeks," **When** the agent processes the
   question, **Then** the response contains a chart payload with type, labels,
   and series data; a human-readable summary highlighting the most
   significant finding; and at least two graph insights calling out patterns
   or outliers in the data.

2. **Given** a sales leader asks "Who has the shortest average sales cycle in
   my team?" **When** the agent processes the question, **Then** the response
   names the team in scope (resolved from the authenticated user's CRM
   profile), returns a ranked chart or leaderboard payload, a summary
   identifying the top performer and what sets them apart, and graph insights
   noting any outliers or surprising results.

3. **Given** a sales rep asks "What's changed in the pipeline this past week?"
   **When** the agent processes the question, **Then** the response includes a
   chart comparing the pipeline at the start and end of the week, a summary
   describing net change direction and magnitude, and graph insights
   identifying the largest individual changes (deals added, advanced, lost).

4. **Given** a user asks a natural-language variant of a scripted question
   (e.g., "Show opportunities that haven't been touched in two weeks"),
   **When** the agent processes the question, **Then** it classifies the
   intent as scripted and returns the same response structure as the
   canonical scripted question.

5. **Given** a user's SugarCRM data returns zero matching records for a
   scripted question, **When** the agent processes the question, **Then** the
   response contains an empty chart payload, a summary communicating "no data
   found," and no graph insights (or an explicit "no insights available"
   marker) — not an error.

---

### User Story 2 – General CRM Data Question (Priority: P2)

A sales rep or leader asks any CRM-related question that does not match
one of the five scripted intents. The agent recognises the question as
ad-hoc, determines which SugarCRM modules and fields are needed to answer
it, retrieves the data, and returns a structured table — rows, columns,
and basic metadata — that the UI can render directly. No chart, summary,
or insights are produced.

**Why this priority**: General questions extend the agent's usefulness
beyond the five scripted cases without requiring manual report-building.
It is a P2 because the core scripted experience (P1) must work first; the
general path adds breadth but is not essential for an initial MVP.

**Independent Test**: Can be fully tested by submitting a novel CRM
question (e.g., "List all open deals in the enterprise segment") with a
valid authentication token and verifying that the response contains a
table payload with labelled columns and at least one data row (or an
explicit empty-table marker), and that no chart or summary fields are
present.

**Acceptance Scenarios**:

1. **Given** a sales rep asks "List all open deals assigned to me this
   quarter," **When** the agent processes the question, **Then** the response
   contains a table with rows representing individual deals, columns
   representing relevant CRM fields (e.g., name, amount, close date, stage,
   owner), and metadata including the total row count and the time range
   applied.

2. **Given** a sales leader asks about a CRM concept the agent cannot map to
   any known SugarCRM module or field, **When** the agent processes the
   question, **Then** the response communicates clearly that the question
   could not be answered, names the unresolvable concept, and suggests how
   the user might rephrase — without returning a partial or silent failure.

3. **Given** a general question is submitted that returns more than 200
   records, **When** the agent processes the question, **Then** the table
   payload includes only the first page of results and metadata indicates
   total record count and whether more records are available.

---

### User Story 3 – Scope Refinement via Follow-up (Priority: P3)

During a conversation session, a sales rep refines the scope of a
previous answer by asking a follow-up question that changes the time
window, team scope, pipeline stage, or product category — without
repeating the full original question. The agent understands that the new
message modifies the prior query, extracts the updated parameters, re-runs
the query, and returns a response in the same format as the original.

**Why this priority**: Follow-up refinement is a key usability enhancement
that makes the agent feel conversational and efficient. It is a P3 because
the standalone single-question experience (P1, P2) must be solid first;
refinement adds polish but is not blocking.

**Independent Test**: Can be fully tested by submitting a scripted question
followed immediately by "Now show me the same for this month" within the
same session, and verifying that the response reflects the updated time
window without changing any other parameters.

**Acceptance Scenarios**:

1. **Given** a user asked "What's changed in the pipeline this past week?"
   and received a response, **When** they follow up with "Now show me the
   last 30 days," **Then** the agent recognises this as a time-window
   refinement of the previous question and returns the same response
   structure with data scoped to the last 30 days.

2. **Given** a user asked "Who has the shortest average sales cycle in my
   team?" and received a response, **When** they follow up with "Filter by
   the enterprise segment only," **Then** the agent applies the additional
   segment filter and returns an updated response.

3. **Given** a user's follow-up is ambiguous (e.g., "What about last
   month?") and no prior question exists in the session, **When** the agent
   processes the message, **Then** the agent responds with a clarifying
   prompt asking what they would like to see for last month, rather than
   silently failing or guessing.

---

### User Story 4 – Unsupported or Uninterpretable Question (Priority: P4)

A user asks a question the agent cannot map to any CRM concept, scripted
intent, or general query pattern — for example, a non-CRM question or
input that is too ambiguous to resolve. The agent responds gracefully with
a structured error response that communicates the limitation and provides
actionable guidance, without exposing internal errors or crashing.

**Why this priority**: Graceful degradation protects user trust and
prevents frustration. It is P4 because it is a safety net rather than a
core capability.

**Independent Test**: Can be fully tested by submitting a question with no
CRM relevance (e.g., "What is the capital of France?") and verifying that
the response contains an error type, a plain-language message, and at
least one suggestion for a supported question — with no chart, table, or
summary fields present.

**Acceptance Scenarios**:

1. **Given** a user submits a question with no CRM relevance, **When** the
   agent processes it, **Then** the response contains an error payload with a
   clear message stating the question is outside the agent's scope and
   examples of supported question types.

2. **Given** a user submits malformed or empty input, **When** the agent
   processes it, **Then** the response prompts the user to provide a
   complete question.

---

### Edge Cases

- What happens when the SugarCRM authentication token is expired or
  invalid? The agent must return a clear, user-friendly authentication
  error — not a data error or silent empty result.
- What happens when SugarCRM returns partial data (e.g., some records
  missing required fields)? The agent must include the partial result with
  a notice indicating data may be incomplete.
- What happens when two scripted intents could plausibly match the same
  question? The agent must select the highest-confidence match and include
  the confidence decision in the response metadata.
- What happens when the user's team scope (for "my team" questions) cannot
  be resolved from the CRM user profile? The agent must return a
  structured error explaining what data is missing and how to resolve it in
  the CRM.
- What happens when a time window parameter spans a future date range
  (e.g., "next quarter")? The agent must return available data up to the
  current date and note in the summary that future data is not available.
- What happens when the SugarCRM API is unreachable or returns a server
  error (timeout, 5xx)? The agent must immediately return a structured
  error response with error category `api-unavailable` and a user-readable
  message — no retries are attempted and no partial data is returned.

## Requirements *(mandatory)*

### Functional Requirements

**Intent Detection & Routing**

- **FR-001**: The agent MUST classify every user question into exactly one
  of two intents: scripted or general, using semantic understanding of the
  question text.
- **FR-002**: The agent MUST extract and apply the following parameter
  types when present in the question: time window (relative: "last N days/
  weeks," "this quarter," "Q[N]"; absolute: specific month or quarter),
  team scope ("my team," a named team or individual), pipeline stage,
  product category, and pipeline change direction.
- **FR-003**: The agent MUST handle natural-language variants of the five
  scripted questions and still classify them as scripted intent — not
  general.
- **FR-004**: The agent MUST maintain question context within a session to
  support follow-up scope refinements without requiring the user to repeat
  the full original question.

**Scripted Analytics Responses**

- **FR-005**: For each of the five scripted questions, the agent MUST
  return a response containing all three of: a chart payload (chart type,
  labels, series data), a narrative summary, and a set of graph insights.
- **FR-006**: Chart payloads MUST be structured so that a UI can render
  them without transformation (self-describing: type declared explicitly,
  labels and series aligned by index).
- **FR-007**: The narrative summary MUST highlight the most significant
  finding in the data in plain language — not merely repeat the data
  values.
- **FR-008**: Graph insights MUST identify at least two noteworthy
  patterns: largest changes, best/worst performers, outliers, or notable
  shifts — derived by interpreting the data, not listing it.
- **FR-009**: If a scripted query returns zero records, the agent MUST
  return a valid empty-state response (empty chart payload + "no data"
  summary) rather than an error.

**General CRM Data Responses**

- **FR-010**: For all non-scripted questions the agent CAN meaningfully
  answer, it MUST return a table payload: rows, columns with labels, and
  metadata (total record count, applied filters, time range).
- **FR-011**: The table response MUST NOT include a chart, narrative
  summary, or graph insights.
- **FR-012**: When a general query returns more records than the defined
  page size, the agent MUST return the first page and include pagination
  metadata indicating total count and whether more records exist.

**SugarCRM Awareness & Data Access**

- **FR-013**: The agent MUST have awareness of standard SugarCRM modules
  (Opportunities, Accounts, Contacts, Leads, Calls, Meetings, Tasks,
  Users, Teams, Products/ProductCatalog), their key fields, and their
  relationships. All data access MUST use the SugarCRM REST API v11.
- **FR-014**: The agent MUST determine autonomously which modules and
  fields are required to answer a given question — without the user naming
  them explicitly.
- **FR-015**: The agent MUST retrieve data from SugarCRM using a
  user-provided authentication token that represents the user's identity
  and access scope. The token MUST be transmitted as a request header
  (e.g., `Authorization` or `X-SugarCRM-Token`) on every outbound
  SugarCRM API call as an OAuth 2.0 Bearer token (`Authorization: Bearer
  <token>`). The token MUST NOT be logged, cached, or persisted beyond
  the lifetime of the single request being processed.
- **FR-016**: "My team" and similar possessive team references MUST be
  resolved using the data visible to the authenticated user as determined
  by SugarCRM's access-control settings. The agent MUST NOT apply any
  additional role-based filtering beyond what the token's CRM access scope
  already enforces.
- **FR-017**: The agent MUST handle authentication errors (expired token,
  insufficient permissions) and return a structured, user-friendly error
  response that does not expose internal system details. The agent MUST
  also handle SugarCRM API failures (network timeout, 503, or any 5xx
  response) by immediately returning a structured error with category
  `api-unavailable` — no retry attempts are made. The raw API error MUST
  NOT be surfaced to the user.

**Response Structure & Consistency**

- **FR-018**: All agent responses MUST conform to a consistent, documented
  response schema so that a single UI renderer can handle all response
  types (scripted, general, error).
- **FR-019**: Responses MUST include a response type field that
  distinguishes: scripted-analytics, general-table, clarification-needed,
  and error.
- **FR-020**: Error responses MUST include a user-readable message, an
  error category (auth, no-data, unsupported-question, partial-data,
  api-unavailable), and at least one suggested next action.

### Non-Functional Requirements

**Observability & Logging**

- **NFR-001**: The agent MUST emit a structured log event for every
  request containing exactly: timestamp, request ID, response type
  (scripted-analytics | general-table | clarification-needed | error),
  end-to-end latency (ms), and error category (if applicable).
- **NFR-002**: The agent MUST NOT log question text, extracted parameter
  values, CRM field values, user identifiers, or authentication tokens.
- **NFR-003**: The agent MUST expose per-request latency as a metric so
  that alerts can fire when p95 latency exceeds the targets defined in
  Success Criteria (SC-001, SC-003).
- **NFR-004**: Log entries MUST be machine-parseable (structured/JSON
  format) to support aggregation and alerting in standard observability
  tooling.

### Key Entities The user's natural-language input. Attributes: raw text,
  detected intent (scripted | general), extracted parameters (time window,
  team scope, stage, product category), session context reference.
- **Intent**: Classification of a question. Values: scripted (with
  matched scripted question ID 1–5) or general.
- **Parameter Set**: The structured parameters extracted from a question.
  Attributes: time window (start date, end date, label), team scope
  (user ID or team ID), stage filter, product category filter.
- **Chart Payload**: The renderable chart output for scripted responses.
  Attributes: chart type (bar, line, pie, etc.), axis labels, series name
  and data values, color series (optional).
- **Table Payload**: The renderable table output for general responses.
  Attributes: column definitions (name, label, data type), rows (list of
  field-value maps), metadata (total count, page size, page number, applied
  filters).
- **Summary**: The narrative interpretation for scripted responses.
  Attributes: text (plain language), data highlight (key metric surfaced).
- **Graph Insight**: A single noteworthy observation for scripted
  responses. Attributes: type (outlier | top-performer | largest-change |
  notable-shift), description (plain language), referenced data point.
- **SugarCRM Metadata**: Agent's internal knowledge of the CRM structure.
  Attributes: module name, available fields (name, type, label),
  relationships to other modules, filterable fields.
- **Session Context**: The record of prior questions and responses within
  a single user session, used to resolve follow-up references.
- **Authentication Token**: The user-provided OAuth 2.0 bearer token for
  accessing SugarCRM REST API v11. Transmitted as an `Authorization:
  Bearer <token>` header on every outbound API call. Scoped to the
  authenticated user's CRM visibility. Never logged, cached, or stored
  beyond the single request lifetime.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All five scripted analytics questions return a complete
  response (chart + summary + at least two graph insights) in under
  3 seconds for datasets of up to 10,000 CRM records.
- **SC-002**: At least 90% of scripted-intent questions (variants, not
  just exact phrasing) are correctly classified as scripted in
  user-acceptance testing.
- **SC-003**: General CRM data questions return a table response in under
  5 seconds for datasets of up to 50,000 records with standard filtering.
- **SC-004**: Follow-up scope refinements are correctly interpreted and
  applied in at least 85% of tested cases without the user repeating the
  full original question.
- **SC-005**: 100% of parameter extractions for time window (relative and
  named quarter) and team scope ("my team") succeed when these parameters
  are explicitly stated in the question.
- **SC-006**: Authentication errors are surfaced to the user as
  human-readable messages in 100% of cases — no raw technical errors
  are ever shown.
- **SC-007**: Sales reps can go from question to decision-ready insight
  for any of the five scripted questions within 60 seconds of typing —
  including reading the summary and graph insights.
- **SC-008**: The rate at which users manually build CRM reports for the
  five scripted question categories decreases by at least 70% within
  30 days of agent availability, as measured by report creation events
  in the CRM audit log.

## Assumptions

The following reasonable defaults have been applied where the feature
description left details unspecified:

- **Authentication scope**: The user-provided token grants the same data
  visibility as the user has in the SugarCRM UI. The agent does not have
  elevated or admin-level access. Data scope — including what constitutes
  "my team" — is entirely determined by SugarCRM's access-control
  settings for the authenticated user. The agent enforces no additional
  role-based distinctions between sales reps and sales leaders.
- **Session context window**: Follow-up refinement context is maintained
  for the duration of a single user session. Sessions do not persist
  across browser refreshes or re-logins unless session restoration is
  explicitly supported in a future feature.
- **Scripted question matching**: The agent uses semantic / intent-based
  classification to match variants of the five scripted questions — not
  exact string matching. Fuzzy matching allows natural phrasing differences
  (word order, synonyms, abbreviations).
- **Default time window**: When a question provides no time parameter,
  the agent defaults to the current calendar quarter.
- **Pagination page size**: General table responses default to 50 rows
  per page.
- **No data persistence**: The agent does not store or cache CRM data
  beyond what is needed to generate a single response. All data is fetched
  fresh per question.
- **Five scripted questions are fixed**: The list of scripted questions is
  predefined and does not change at runtime. Adding new scripted questions
  is a future feature enhancement, not in scope here.
- **SugarCRM API version**: The agent targets SugarCRM REST API v11.
  Support for other versions (v10, custom) is out of scope and would
  require a separate feature.
