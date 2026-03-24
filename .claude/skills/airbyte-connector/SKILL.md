---
name: airbyte-connector
description: "Create Airbyte low-code connectors for the Insight platform — full spec-driven workflow: PRD → DESIGN → manifest.yaml. Use when the user wants to create, scaffold, or generate a new Airbyte connector. Handles: API research, PRD generation (requirements, actors, use cases), DESIGN generation (architecture, Bronze table schemas, sequence diagrams, traceability), and manifest.yaml generation (streams, auth, pagination, incremental sync, error handling). Follows Cypilot SDLC conventions with traceability IDs. Can also run in manifest-only mode when PRD/DESIGN already exist."
disable-model-invocation: false
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Task, WebFetch, WebSearch, Agent
---

# Airbyte Connector Skill — Spec-Driven Development

You create Airbyte low-code connectors for the Insight Decision Intelligence Platform following spec-driven development: **PRD → DESIGN → manifest.yaml**.

---

## Context Loading — MANDATORY

Before doing ANY work, read these files:

1. `docs/shared/airbyte-manifest-reference/MANIFEST_REFERENCE.md` — Airbyte manifest syntax
2. `docs/shared/airbyte-manifest-reference/MANIFEST_EXAMPLE.yaml` — complete manifest example
3. `docs/domain/connector/specs/DESIGN.md` — Connector Framework architecture (§3.2 Component Model, §3.7 Database schemas, §Connector Manifest Schema, §New Connector Checklist)
4. `docs/components/connectors/{domain}/README.md` — unified schema for the target domain
5. `docs/components/connectors/{domain}/{source}/specs/` — any existing PRD/DESIGN for this connector

Also read these for conventions:
6. `cypilot/config/rules/conventions.md` — naming, formatting rules
7. `cypilot/config/rules/architecture.md` — Bronze/Silver/Gold layer rules
8. `cypilot/config/rules/patterns.md` — connector documentation patterns

If the user is asking for a connector in a domain that has an existing connector with PRD+DESIGN (e.g., GitHub for git, M365 for collaboration), read that existing spec pair as a **reference model** for structure, depth, and style.

---

## Mode Detection

Detect the mode based on what already exists:

### Full Mode (default) — PRD → DESIGN → manifest.yaml

Use when the connector has NO existing PRD and DESIGN. This is the standard flow.

### Manifest-Only Mode

Use when PRD and DESIGN already exist at `docs/components/connectors/{domain}/{source}/specs/`. Skip to Step 4 (manifest generation), reading the existing specs as input.

### Spec-Only Mode

Use when the user explicitly asks for only specs (PRD, DESIGN) without a manifest. Stop after Step 3.

---

## Step 0: Gather Requirements

Determine the following — ask the user if not provided:

- **Source system** (e.g., YouTrack, Jira, Slack, BambooHR)
- **Source category** (git, task-tracking, collaboration, wiki, support, ai-dev, ai, hr-directory, crm, ui-design, testing)
- **API documentation URL** (fetch and read if provided)
- **Authentication method** (API key, OAuth2, Bearer, Basic, etc.)
- **Target entities** (which data to extract — or "all from DESIGN")

Then **research the API**:
- If the user provides an API docs URL → `WebFetch` it
- If not → `WebSearch` for "{source} API documentation"
- Identify: base URL, auth mechanism, endpoints, pagination style, rate limits, available entities, timestamp fields for incremental sync

Present a brief plan to the user before proceeding:
```
Source: {name}
Category: {category}
Auth: {type}
Streams: {list of Bronze tables}
Pagination: {strategy}
Incremental: {cursor field(s)}
```

---

## Step 1: Generate PRD

Write to: `docs/components/connectors/{domain}/{source}/specs/PRD.md`

### PRD Structure

Follow this exact section structure (matching Cypilot SDLC PRD template and existing connector PRDs):

```markdown
# PRD — {Source Name} Connector

> Version 1.0 — {Month YYYY}
> Based on: {domain reference} (`docs/components/connectors/{domain}/README.md`)

<!-- toc -->
{auto-generated TOC}
<!-- /toc -->

---

## 1. Overview
### 1.1 Purpose
### 1.2 Background / Problem Statement
### 1.3 Goals (Business Outcomes)
### 1.4 Glossary

## 2. Actors
### 2.1 Human Actors
### 2.2 System Actors

## 3. Operational Concept & Environment
### 3.1 Module-Specific Environment Constraints

## 4. Scope
### 4.1 In Scope
### 4.2 Out of Scope

## 5. Functional Requirements
### 5.x {Feature Area}

## 6. Non-Functional Requirements
### 6.1 NFR Inclusions
### 6.2 NFR Exclusions

## 7. Public Library Interfaces
### 7.1 Public API Surface
### 7.2 External Integration Contracts

## 8. Use Cases

## 9. Acceptance Criteria

## 10. Dependencies

## 11. Assumptions

## 12. Risks

## 13. Open Questions
```

### PRD Conventions

**Traceability IDs** — use this pattern consistently:
- Actors: `cpt-insightspec-actor-{src}-{slug}` (e.g., `cpt-insightspec-actor-yt-platform-engineer`)
- Functional requirements: `cpt-insightspec-fr-{src}-{slug}` (e.g., `cpt-insightspec-fr-yt-collect-issues`)
- Non-functional requirements: `cpt-insightspec-nfr-{src}-{slug}`
- Interfaces: `cpt-insightspec-interface-{src}-{slug}`
- Contracts: `cpt-insightspec-contract-{src}-{slug}`
- Use cases: `cpt-insightspec-usecase-{src}-{slug}`

Where `{src}` is a short abbreviation for the source (e.g., `gh` for GitHub, `yt` for YouTrack, `m365` for Microsoft 365, `bb` for Bitbucket, `jira` for Jira).

**Requirement format**:
```markdown
- [ ] `p{N}` - **ID**: `cpt-insightspec-fr-{src}-{slug}`

The system **MUST** {observable behavior}.

**Rationale**: {why this matters for analytics/business}

**Actors**: `cpt-insightspec-actor-{src}-{slug}`
```

**Priority tiers**:
- `p1` — must-have for initial release
- `p2` — important but not blocking
- `p3` — nice-to-have / future

**Mandatory content per connector PRD**:
1. **At least one FR per Bronze entity** (each API entity to be collected)
2. **Deduplication FR** — how records are deduplicated (primary key definition)
3. **Identity key FR** — which field is the identity anchor for person resolution
4. **Incremental collection FR** — cursor strategy
5. **Fault tolerance FR** — error handling, retry, partial failure behavior
6. **Collection runs FR** — monitoring stream
7. **NFRs**: auth flexibility, rate limit compliance, schema compliance (source-native Bronze), idempotent writes
8. **At least 2 use cases**: initial full sync, incremental sync
9. **Open questions** with IDs: `OQ-{SRC}-{N}: {description}`

---

## Step 2: Review PRD with User

Present a summary of the PRD:
- Number of FRs, NFRs, use cases
- List of Bronze entities identified
- Key constraints and open questions

Ask: "PRD looks good? I'll proceed to DESIGN." — then continue.

---

## Step 3: Generate DESIGN

Write to: `docs/components/connectors/{domain}/{source}/specs/DESIGN.md`

### DESIGN Structure

```markdown
# DESIGN — {Source Name} Connector

- [ ] `p{N}` - **ID**: `cpt-insightspec-design-{src}-connector`

> Version 1.0 — {Month YYYY}
> Based on: {domain reference}, [PRD.md](./PRD.md)

<!-- toc -->
{auto-generated TOC}
<!-- /toc -->

---

## 1. Architecture Overview
### 1.1 Architectural Vision
### 1.2 Architecture Drivers
### 1.3 Architecture Layers

## 2. Principles & Constraints
### 2.1 Design Principles
### 2.2 Constraints

## 3. Technical Architecture
### 3.1 Domain Model
### 3.2 Component Model
### 3.3 API Contracts
### 3.4 Internal Dependencies
### 3.5 External Dependencies
### 3.6 Interactions & Sequences
### 3.7 Database schemas & tables
### 3.8 Deployment Topology

## 4. Additional context

## 5. Traceability

## 6. Non-Applicability Statements
```

### DESIGN Conventions

**Traceability IDs**:
- Principles: `cpt-insightspec-principle-{src}-{slug}`
- Constraints: `cpt-insightspec-constraint-{src}-{slug}`
- Components: `cpt-insightspec-component-{src}-{slug}`
- Interfaces: `cpt-insightspec-interface-{src}-{slug}`
- Sequences: `cpt-insightspec-seq-{src}-{slug}`
- Database tables: `cpt-insightspec-dbtable-{src}-{slug}`
- Technologies: `cpt-insightspec-tech-{src}-{slug}`
- Topology: `cpt-insightspec-topology-{src}-{slug}`

**Architecture Drivers table** — map EVERY PRD FR to a design response:
```markdown
| Requirement | Design Response |
|-------------|-----------------|
| `cpt-insightspec-fr-{src}-{slug}` | Stream `{stream_name}` → `{endpoint}` |
```

**NFR Allocation table** — map every PRD NFR:
```markdown
| NFR ID | NFR Summary | Allocated To | Design Response | Verification Approach |
```

**§1.1 Architectural Vision** — MUST state:
- This is an Airbyte declarative manifest connector (YAML, no code)
- Which API version and endpoints are used
- Authentication method
- Pagination strategy
- Incremental sync approach
- How many streams and what they map to

**§1.3 Architecture Layers** — use this standard table:

| Layer | Responsibility | Technology |
|-------|---------------|------------|
| Source API | {Source} API endpoints | REST / JSON (or GraphQL, etc.) |
| Authentication | {Auth type} | {Token endpoint or mechanism} |
| Connector | Stream definitions, pagination, incremental sync | Airbyte declarative manifest (YAML) |
| Execution | Container runtime | Airbyte Declarative Connector framework |
| Bronze | Raw data storage with source-native schema | Destination connector (ClickHouse) |

**§2.1 Design Principles** — ALWAYS include:
1. One Stream per Endpoint (or entity group)
2. Source-Native Schema (Bronze = raw API field names, no transformation)

**§3.2 Component Model** — for declarative connectors, the manifest IS the component:
```markdown
#### {Source} Connector Manifest

- [ ] `p2` - **ID**: `cpt-insightspec-component-{src}-manifest`

##### Why this component exists
Defines the complete {source} connector as a YAML declarative manifest...

##### Responsibility scope
Defines all N streams with: {endpoint details}, {auth type}, {pagination}, {incremental sync}, {transformations}, inline JSON schemas.

##### Responsibility boundaries
Does not handle orchestration, scheduling, or state storage. Does not perform Silver/Gold transformations.
```

**§3.3 API Contracts** — document:
- All endpoints with their HTTP method, path, parameters
- Request/response format
- Auth details (token endpoint, scopes, header format)
- Rate limits
- Source config schema (JSON with field descriptions)

**§3.6 Interactions & Sequences** — include at least one Mermaid sequence diagram showing:
```mermaid
sequenceDiagram
    participant Orch as Orchestrator
    participant Src as Source Container
    participant API as {Source} API
    participant Dest as Destination
    ...
```

**§3.7 Database schemas & tables** — this is the CRITICAL section. For EACH Bronze table:

```markdown
#### Table: `{source}_{entity}`

| Column | Type | Description |
|--------|------|-------------|
| {pk_field} | String | PK: {description} |
| {identity_field} | String | Identity key — {description} |
| {cursor_field} | String/DateTime | Cursor field for incremental sync |
| ... | ... | ... |
```

Follow these rules from `cypilot/config/rules/patterns.md`:
- Exactly 3 columns: `| Column | Type | Description |`
- Use canonical types: String, Number, Boolean, DateTime, Array, String (JSON)
- Include `metadata` field (`String (JSON)`) for primary entity tables where API response contains extra data
- Identity fields documented with "— identity key" suffix
- Cursor fields documented with "— cursor for incremental sync" suffix
- ALWAYS end with `collection_runs` table + note: "Monitoring table — not an analytics source."

**§3.8 Deployment Topology** — use this pattern:
```
Connection: {source}-{instance_name}
├── Source image: airbyte/source-declarative-manifest
├── Manifest: src/connectors/{source}/manifest.yaml
├── Source config: {list of config fields}
├── Configured catalog: {N} streams
├── Destination image: airbyte/destination-clickhouse (or other)
├── Destination config: {host, port, database, schema, credentials}
└── State: per-stream cursor positions
```

**§4 Additional Context** — include:
- Identity Resolution Strategy (which field → person_id)
- Silver / Gold Mappings table (Bronze table → Silver target → Status)
- Source-specific considerations

**§5 Traceability** — link to PRD, domain README, ADR directory

---

## Step 4: Generate manifest.yaml

Write to: `src/connectors/{source}/manifest.yaml`

### Manifest Rules

**Read the full reference at** `docs/shared/airbyte-manifest-reference/MANIFEST_REFERENCE.md` before generating.

#### Top-level structure

```yaml
version: "6.44.0"

definitions:
  # Shared components (DRY via $ref)

streams:
  - $ref: "#/definitions/{stream}_stream"

check:
  type: CheckStream
  stream_names:
    - "{lightest_stream}"

spec:
  type: Spec
  connection_specification:
    $schema: "http://json-schema.org/draft-07/schema#"
    type: object
    required: [...]
    properties: ...
```

#### Naming (Insight conventions)

- Stream names: `{source}_{entity}` (e.g., `youtrack_issues`, `bamboohr_employees`)
- Primary keys: source system's natural key
- Cursor fields: prefer `updated_at`/`updated`/modification timestamp
- Data source field: `_airbyte_data_source` = `"insight_{source}"`

#### Authentication mapping

| Source Auth | Airbyte Type |
|-------------|-------------|
| API token / PAT | `BearerAuthenticator` or `ApiKeyAuthenticator` |
| OAuth 2.0 | `OAuthAuthenticator` |
| Basic auth | `BasicHttpAuthenticator` |
| Session/cookie | `SessionTokenAuthenticator` |
| Multiple options | `SelectiveAuthenticator` |

#### Mandatory patterns

1. **Error handling** — ALWAYS `CompositeErrorHandler` with 429 (rate limit) + 5XX (retry):
   ```yaml
   error_handler:
     type: CompositeErrorHandler
     error_handlers:
       - type: DefaultErrorHandler
         response_filters:
           - type: HttpResponseFilter
             http_codes: [429]
             action: RATE_LIMITED
         backoff_strategies:
           - type: WaitTimeFromHeader
             header: "Retry-After"
       - type: DefaultErrorHandler
         response_filters:
           - type: HttpResponseFilter
             http_codes: [500, 502, 503, 504]
             action: RETRY
         backoff_strategies:
           - type: ExponentialBackoffStrategy
             factor: 5
         max_retries: 5
   ```

2. **Metadata transformations** — on EVERY stream:
   ```yaml
   transformations:
     - type: AddFields
       fields:
         - path: ["_airbyte_data_source"]
           value: "insight_{source}"
         - path: ["_airbyte_collected_at"]
           value: "{{ now_utc().strftime('%Y-%m-%dT%H:%M:%SZ') }}"
   ```

3. **Incremental sync** — prefer `DatetimeBasedCursor` with:
   - `lookback_window`: ≥ `P1D`
   - `step`: P7D–P30D
   - `cursor_granularity`: match source precision

4. **Pagination** — largest `page_size` API supports; explicit `stop_condition` for cursor pagination

5. **Inline schemas** — `InlineSchemaLoader` for every stream, complete JSON Schema matching the DESIGN §3.7 table definitions:
   - Nullable fields: `{"type": ["null", "string"]}`
   - Timestamps: `{"type": ["null", "string"], "format": "date-time"}`
   - Arrays: `{"type": ["null", "array"], "items": {...}}`

6. **Parent-child streams** — `SubstreamPartitionRouter` + `AddFields` for parent ID + `incremental_dependency: true`

7. **Rate limiting** — if API documents limits:
   ```yaml
   api_budget:
     type: HTTPAPIBudget
     policies:
       - type: FixedWindowCallRatePolicy
         period: "{window}"
         call_limit: {limit}
   ```

#### Schema consistency

The inline schemas in `manifest.yaml` MUST match the table definitions in DESIGN §3.7 exactly:
- Same field names
- Compatible types (DESIGN `String` → schema `"string"`, DESIGN `Number` → schema `"number"` or `"integer"`, etc.)
- Same primary keys

---

## Step 5: Validation

Run through this checklist and report results:

### Spec Validation
- [ ] PRD has at least one FR per Bronze entity
- [ ] PRD has deduplication, identity, incremental, fault tolerance, and collection_runs FRs
- [ ] PRD has ≥ 2 use cases (full sync + incremental)
- [ ] PRD has open questions with IDs
- [ ] DESIGN §1.2 maps every PRD FR to a design response
- [ ] DESIGN §1.2 NFR table maps every PRD NFR
- [ ] DESIGN §3.7 has a table definition for every Bronze entity + collection_runs
- [ ] DESIGN §3.7 tables follow 3-column format with identity/cursor annotations
- [ ] DESIGN §3.6 has at least one sequence diagram
- [ ] DESIGN §5 links to PRD
- [ ] All traceability IDs follow `cpt-insightspec-{kind}-{src}-{slug}` pattern

### Manifest Validation
- [ ] All `{{ config['...'] }}` references match `spec.connection_specification.properties`
- [ ] All `$ref` references point to existing definitions
- [ ] All streams have `primary_key` set
- [ ] All streams with incremental sync have `cursor_field` matching a real response field
- [ ] `check.stream_names` references an existing, lightweight stream
- [ ] Error handler covers 429 and 5XX
- [ ] Pagination matches the API's actual pagination mechanism
- [ ] Every stream has a complete inline schema
- [ ] Stream names follow `{source}_{entity}` convention
- [ ] `_airbyte_data_source` and `_airbyte_collected_at` transformations on every stream
- [ ] Inline schemas match DESIGN §3.7 table definitions

---

## What NOT To Do

- Do NOT generate Silver/Gold layer mappings (human-authored semantic decisions)
- Do NOT generate `unifier_mapping.yaml` (separate concern)
- Do NOT add content/PII fields (message bodies, email text) unless explicitly requested
- Do NOT use `JsonFileSchemaLoader` — always inline schemas
- Do NOT hardcode credentials — always `{{ config['...'] }}`
- Do NOT skip error handling or pagination
- Do NOT use custom Python components unless the API genuinely cannot be handled declaratively
- Do NOT invent API endpoints — verify against real API documentation
- Do NOT generate Bronze table names that violate `{source}_{entity}` convention

---

## Output Summary

When complete, present:

```
## Connector: {Source Name}

### Files Created
- docs/components/connectors/{domain}/{source}/specs/PRD.md
- docs/components/connectors/{domain}/{source}/specs/DESIGN.md
- src/connectors/{source}/manifest.yaml

### Streams
{table of stream name → API endpoint → primary key → cursor field}

### Validation
{checklist results}

### Next Steps
- [ ] Review PRD open questions
- [ ] Verify API field names against live endpoint
- [ ] Create Silver unifier_mapping.yaml (human-authored)
- [ ] Register connector in orchestrator
```
