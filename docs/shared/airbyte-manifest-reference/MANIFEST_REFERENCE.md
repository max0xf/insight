# Airbyte Low-Code Connector Manifest Reference

> Compiled from official Airbyte documentation (March 2026). Covers CDK v6.44+.

## 1. Manifest Structure

A connector is defined by a single `manifest.yaml`. Top-level keys:

```yaml
version: "6.44.0"          # REQUIRED - CDK version
definitions:                # OPTIONAL - reusable YAML anchors
streams:                    # REQUIRED - list of DeclarativeStream
dynamic_streams:            # OPTIONAL - runtime-generated streams
check:                      # REQUIRED - connection validation
spec:                       # REQUIRED - connector config schema (JSON Schema Draft-07)
concurrency_level:          # OPTIONAL - parallel stream reads
api_budget:                 # OPTIONAL - rate limiting (HTTPAPIBudget)
```

| Key | Required | Purpose |
|-----|----------|---------|
| `version` | Yes | CDK version the manifest targets |
| `definitions` | No | Reusable YAML anchors for DRY configuration |
| `streams` | Yes | List of `DeclarativeStream` objects |
| `dynamic_streams` | No | Streams generated from templates at runtime |
| `check` | Yes | `CheckStream` for connection validation |
| `spec` | Yes | `Spec` with `connection_specification` (JSON Schema Draft-07) |
| `concurrency_level` | No | Controls parallel stream reads |
| `api_budget` | No | Rate limiting policies |

### The `definitions` Section

Uses YAML anchors/aliases and `$ref` for reusability:

```yaml
definitions:
  selector:
    type: RecordSelector
    extractor:
      type: DpathExtractor
      field_path: ["data"]

  requester:
    type: HttpRequester
    url_base: "https://api.example.com/v1"
    http_method: GET
    authenticator:
      type: BearerAuthenticator
      api_token: "{{ config['api_key'] }}"

  retriever:
    type: SimpleRetriever
    record_selector:
      $ref: "#/definitions/selector"
    requester:
      $ref: "#/definitions/requester"
```

### The `spec` Section

JSON Schema Draft-07 with Airbyte-specific annotations:

- `airbyte_secret: true` — masks sensitive fields in UI
- `order: <int>` — controls field display order
- `always_show: true` — prevents optional field collapsing
- `airbyte_hidden: true` — hides field from UI (API-only)
- `multiline: true` — enables multi-line string input
- `pattern_descriptor` — human-readable format hint
- `group` — organizes fields into UI cards

**Forbidden JSON Schema keys:** `not`, `anyOf`, `patternProperties`, `prefixItems`, `allOf`, `if`, `then`, `else`, `dependentSchemas`, `dependentRequired`

The `oneOf` keyword creates dropdown/radio selections. Each option must be an object with a `const` discriminator field.

### The `check` Section

```yaml
check:
  type: CheckStream
  stream_names:
    - "stream_to_test"
```

---

## 2. Streams

Each stream = one API resource (analogous to a table).

```yaml
streams:
  - type: DeclarativeStream
    name: "users"
    primary_key: "id"                    # string, list, or list-of-lists for composite
    schema_loader:
      type: InlineSchemaLoader
      schema:
        $schema: "http://json-schema.org/draft-07/schema#"
        type: object
        properties:
          id:
            type: integer
          name:
            type: string
    retriever:
      type: SimpleRetriever
      requester:
        type: HttpRequester
        url_base: "https://api.example.com"
        path: "/users"
        http_method: GET
      record_selector:
        type: RecordSelector
        extractor:
          type: DpathExtractor
          field_path: ["data", "users"]
      paginator: ...
    incremental_sync: ...
    transformations: ...
```

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `DeclarativeStream` |
| `name` | Yes | Stream identifier |
| `retriever` | Yes | SimpleRetriever / AsyncRetriever / CustomRetriever |
| `primary_key` | No | Unique record identifier(s) |
| `schema_loader` | No | InlineSchemaLoader / JsonFileSchemaLoader / DynamicSchemaLoader |
| `incremental_sync` | No | DatetimeBasedCursor / IncrementingCountCursor |
| `transformations` | No | Record transformations |
| `state_migrations` | No | State format upgraders |

**Schema loading options:**
- **JsonFileSchemaLoader** (default): loads from `schemas/<stream_name>.json`
- **InlineSchemaLoader**: schema defined directly in YAML
- **DynamicSchemaLoader**: schema inferred from API metadata endpoint

---

## 3. Authentication

Configured on `requester.authenticator`.

### API Key

```yaml
authenticator:
  type: ApiKeyAuthenticator
  api_token: "{{ config['api_key'] }}"
  inject_into:
    type: RequestOption
    field_name: "X-API-Key"
    inject_into: header          # header | request_parameter | body_data | body_json
```

### Bearer Token

```yaml
authenticator:
  type: BearerAuthenticator
  api_token: "{{ config['api_token'] }}"
```

### Basic HTTP

```yaml
authenticator:
  type: BasicHttpAuthenticator
  username: "{{ config['username'] }}"
  password: "{{ config['password'] }}"
```

### OAuth 2.0

```yaml
authenticator:
  type: OAuthAuthenticator
  client_id: "{{ config['client_id'] }}"
  client_secret: "{{ config['client_secret'] }}"
  refresh_token: "{{ config['refresh_token'] }}"
  token_refresh_endpoint: "https://api.example.com/oauth/token"
  grant_type: "refresh_token"          # or "client_credentials"
  scopes: ["read", "write"]
  access_token_name: "access_token"
  expires_in_name: "expires_in"
  refresh_request_body:
    grant_type: "refresh_token"
```

For single-use refresh tokens:
```yaml
  refresh_token_updater:
    refresh_token_name: "refresh_token"
    refresh_token_config_path: ["credentials", "refresh_token"]
```

### JWT

```yaml
authenticator:
  type: JwtAuthenticator
  secret_key: "{{ config['secret_key'] }}"
  algorithm: "RS256"                   # HS256, RS256, ES256
  token_duration: 3600
  jwt_headers:
    kid: "{{ config['key_id'] }}"
  jwt_payload:
    iss: "{{ config['issuer'] }}"
  base64_encode_secret_key: true
```

### Session Token

```yaml
authenticator:
  type: SessionTokenAuthenticator
  login_requester:
    type: HttpRequester
    url_base: "https://api.example.com"
    path: "/auth/login"
    http_method: POST
    request_body_json:
      username: "{{ config['username'] }}"
      password: "{{ config['password'] }}"
  session_token_path: ["id"]
  expiration_duration: "P14D"
  request_authentication:
    type: ApiKeyAuthenticator
    inject_into:
      type: RequestOption
      field_name: "X-Session-Token"
      inject_into: header
```

### Selective Authenticator

Dynamically chooses auth based on config:

```yaml
authenticator:
  type: SelectiveAuthenticator
  authenticator_selection_path: ["credentials", "auth_type"]
  authenticators:
    api_key:
      type: ApiKeyAuthenticator
      ...
    oauth:
      type: OAuthAuthenticator
      ...
```

---

## 4. Pagination

### Offset Increment

```yaml
paginator:
  type: DefaultPaginator
  page_size_option:
    type: RequestOption
    inject_into: request_parameter
    field_name: "limit"
  page_token_option:
    type: RequestOption
    inject_into: request_parameter
    field_name: "offset"
  pagination_strategy:
    type: OffsetIncrement
    page_size: 100
    inject_on_first_request: true
```

### Page Increment

```yaml
paginator:
  type: DefaultPaginator
  page_size_option:
    type: RequestOption
    inject_into: request_parameter
    field_name: "per_page"
  page_token_option:
    type: RequestOption
    inject_into: request_parameter
    field_name: "page"
  pagination_strategy:
    type: PageIncrement
    page_size: 50
    start_from_page: 1
```

### Cursor Pagination

```yaml
paginator:
  type: DefaultPaginator
  page_token_option:
    type: RequestOption
    inject_into: request_parameter
    field_name: "cursor"
  pagination_strategy:
    type: CursorPagination
    cursor_value: "{{ response['meta']['next_cursor'] }}"
    page_size: 100
    stop_condition: "{{ response['meta']['next_cursor'] is none }}"
```

**For link-header pagination (e.g., GitHub):**
```yaml
paginator:
  type: DefaultPaginator
  page_token_option:
    type: RequestPath                   # replaces entire URL
  pagination_strategy:
    type: CursorPagination
    cursor_value: "{{ headers['link']['next']['url'] }}"
    stop_condition: "{{ 'next' not in headers['link'] }}"
```

**Cursor context variables:** `response`, `headers`, `last_record`, `last_page_size`, `config`, `next_page_token`.

---

## 5. Incremental Sync

### DatetimeBasedCursor

```yaml
incremental_sync:
  type: DatetimeBasedCursor
  cursor_field: "updated_at"
  cursor_datetime_formats:
    - "%Y-%m-%dT%H:%M:%S.%f%z"
    - "%Y-%m-%dT%H:%M:%SZ"
  datetime_format: "%Y-%m-%dT%H:%M:%SZ"
  start_datetime:
    type: MinMaxDatetime
    datetime: "{{ config['start_date'] }}"
    datetime_format: "%Y-%m-%dT%H:%M:%SZ"
    min_datetime: "2020-01-01T00:00:00Z"
  end_datetime:
    type: MinMaxDatetime
    datetime: "{{ now_utc().strftime('%Y-%m-%dT%H:%M:%SZ') }}"
  step: "P30D"                                  # ISO 8601 duration
  cursor_granularity: "PT1S"                    # smallest increment
  lookback_window: "P3D"                        # re-read 3 days back
  start_time_option:
    type: RequestOption
    inject_into: request_parameter
    field_name: "updated_after"
  end_time_option:
    type: RequestOption
    inject_into: request_parameter
    field_name: "updated_before"
  is_data_feed: false                           # true = newest-first APIs
  is_client_side_incremental: false             # true = local filtering only
```

**Key fields:**
- `step` + `cursor_granularity` — splits ranges into windows, prevents overlap
- `lookback_window` — recaptures late-arriving records
- `is_data_feed: true` — stops when cursor exceeds state (newest-first APIs)
- `is_client_side_incremental: true` — filters locally when API can't filter

### IncrementingCountCursor

```yaml
incremental_sync:
  type: IncrementingCountCursor
  cursor_field: "id"
  start_value: 0
```

---

## 6. Request/Response Handling

### HttpRequester

```yaml
requester:
  type: HttpRequester
  url_base: "https://api.example.com/v2"
  path: "/users"                              # supports interpolation
  http_method: GET                            # GET or POST
  authenticator: ...
  request_parameters:
    status: "active"
    fields: "id,name,email"
  request_headers:
    Accept: "application/json"
  request_body_json:                          # for POST
    query: "{{ config['query'] }}"
  request_body_data:                          # for form-encoded POST
    grant_type: "client_credentials"
  error_handler: ...
```

### RecordSelector

```yaml
record_selector:
  type: RecordSelector
  extractor:
    type: DpathExtractor
    field_path: ["data", "results"]
  record_filter:
    type: RecordFilter
    condition: "{{ record['status'] != 'deleted' }}"
  schema_normalization: Default               # Default | None
```

**DpathExtractor `field_path` patterns:**
- `["data"]` — selects `response.data`
- `["data", "*", "record"]` — wildcard through arrays
- `[]` (empty) — entire response as single record

### Decoders

| Decoder | Format |
|---------|--------|
| `JsonDecoder` | JSON (default) |
| `JsonlDecoder` | JSON Lines |
| `CsvDecoder` | CSV |
| `XmlDecoder` | XML |
| `CustomDecoder` | Python class |

---

## 7. Error Handling

Default: retries 429 and 5XX up to 5 times with exponential backoff (5s base).

### DefaultErrorHandler

```yaml
error_handler:
  type: DefaultErrorHandler
  max_retries: 5
  backoff_strategies:
    - type: ExponentialBackoffStrategy
      factor: 5
  response_filters:
    - type: HttpResponseFilter
      http_codes: [429]
      action: RATE_LIMITED
    - type: HttpResponseFilter
      http_codes: [500, 502, 503]
      action: RETRY
    - type: HttpResponseFilter
      http_codes: [403]
      action: FAIL
      failure_type: config_error
      error_message: "Access denied."
```

### CompositeErrorHandler

```yaml
error_handler:
  type: CompositeErrorHandler
  error_handlers:
    - type: DefaultErrorHandler
      response_filters:
        - http_codes: [429]
          action: RATE_LIMITED
      backoff_strategies:
        - type: WaitTimeFromHeader
          header: "Retry-After"
    - type: DefaultErrorHandler
      response_filters:
        - http_codes: [500, 502, 503]
          action: RETRY
      backoff_strategies:
        - type: ExponentialBackoffStrategy
          factor: 5
```

### Backoff Strategies

| Strategy | Key Field |
|----------|-----------|
| `ConstantBackoffStrategy` | `backoff_time_in_seconds` |
| `ExponentialBackoffStrategy` | `factor` (default 5) |
| `WaitTimeFromHeader` | `header` (e.g., `Retry-After`) |
| `WaitUntilTimeFromHeader` | `header`, `regex`, `min_wait` |

### HttpResponseFilter Actions

| Action | Effect |
|--------|--------|
| `SUCCESS` | Treat as successful |
| `RETRY` | Retry with backoff |
| `FAIL` | Stop sync |
| `IGNORE` | Skip, continue |
| `RATE_LIMITED` | Rate-limit-specific retry |

### Rate Limiting (API Budget)

```yaml
api_budget:
  type: HTTPAPIBudget
  policies:
    - type: FixedWindowCallRatePolicy
      period: PT1M
      call_limit: 60
    - type: MovingWindowCallRatePolicy
      rates:
        - type: Rate
          limit: 1000
          interval: PT1H
```

---

## 8. Transformations

### AddFields

```yaml
transformations:
  - type: AddFields
    fields:
      - path: ["source_name"]
        value: "my_api"
      - path: ["fetched_at"]
        value: "{{ now_utc().strftime('%Y-%m-%dT%H:%M:%SZ') }}"
      - path: ["parent_id"]
        value: "{{ stream_partition['parent_id'] }}"
      - path: ["hashed_email"]
        value: "{{ record['email'] | hash('md5') }}"
    condition: "{{ record['type'] == 'active' }}"
```

### RemoveFields

```yaml
transformations:
  - type: RemoveFields
    field_pointers:
      - ["internal_field"]
      - ["nested", "secret"]
      - ["**", "name"]              # glob: removes 'name' at any depth
```

### Other Transformations

| Type | Purpose |
|------|---------|
| `FlattenFields` | Flatten nested records (`flatten_lists: true/false`) |
| `KeysToLower` | Lowercase all keys |
| `KeysToSnakeCase` | Convert keys to snake_case |
| `KeysReplace` | Pattern replacement in keys (`old`/`new`) |
| `DpathFlattenFields` | Flatten specific nested field |
| `RecordFilter` | Filter records by condition |

---

## 9. Interpolation / Jinja Templates

String values in `{{ ... }}` are Jinja2 templates.

### Context Variables

| Variable | Available In | Description |
|----------|-------------|-------------|
| `config` | Everywhere | User-provided config |
| `parameters` | Everywhere | `$parameters` from YAML |
| `record` | Selectors, transforms, filters | Current record |
| `response` | Selectors, pagination, errors | Raw response body |
| `headers` | Pagination, errors | Response headers |
| `stream_partition` | Requests, transforms | Current partition values |
| `stream_slice` | Requests | Partition + interval context |
| `stream_interval` | Incremental contexts | `start_time` / `end_time` |
| `next_page_token` | Pagination | Current cursor value |
| `last_record` | Pagination stop | Last record from page |
| `last_page_size` | Pagination stop | Records on current page |

### Built-in Functions

- `now_utc()` — current UTC datetime
- `max(a, b)` / `min(a, b)`
- `day_delta(n)` — datetime N days from now
- `timestamp(dt)` — Unix timestamp
- `format_datetime(dt, fmt)`

### Jinja Filters

- `| hash('md5')` / `| hash('sha256')`
- `| urlencode` / `| urldecode`
- `| b64encode` / `| b64decode`
- Standard Jinja2: `capitalize`, `upper`, `lower`, `default`, `tojson`, `length`, etc.

---

## 10. Substreams / Parent-Child Relationships

```yaml
streams:
  - type: DeclarativeStream
    name: "repositories"
    retriever:
      type: SimpleRetriever
      requester:
        type: HttpRequester
        path: "/repositories"
      record_selector:
        type: RecordSelector
        extractor:
          type: DpathExtractor
          field_path: ["data"]

  - type: DeclarativeStream
    name: "commits"
    retriever:
      type: SimpleRetriever
      requester:
        type: HttpRequester
        path: "/repositories/{{ stream_partition['repository_id'] }}/commits"
      record_selector:
        type: RecordSelector
        extractor:
          type: DpathExtractor
          field_path: ["data"]
      partition_router:
        type: SubstreamPartitionRouter
        parent_stream_configs:
          - type: ParentStreamConfig
            stream:
              $ref: "#/definitions/repositories_stream"
            parent_key: "id"
            partition_field: "repository_id"
            incremental_dependency: true
    transformations:
      - type: AddFields
        fields:
          - path: ["repository_id"]
            value: "{{ stream_partition['repository_id'] }}"
```

**ParentStreamConfig fields:**
- `stream` — reference to parent
- `parent_key` — field to extract from parent records
- `partition_field` — name in `stream_partition['<name>']`
- `request_option` — optional injection into request
- `incremental_dependency` — parent reads incrementally based on child state

---

## 11. Partition Routers

### ListPartitionRouter

```yaml
partition_router:
  type: ListPartitionRouter
  values: "{{ config['account_ids'] }}"    # or static list
  cursor_field: "account_id"
  request_option:
    type: RequestOption
    field_name: "account_id"
    inject_into: request_parameter
```

### Multiple Routers (Cartesian Product)

```yaml
partition_router:
  - type: ListPartitionRouter
    values: ["us", "eu"]
    cursor_field: "region"
  - type: ListPartitionRouter
    values: ["desktop", "mobile"]
    cursor_field: "platform"
```

### GroupingPartitionRouter

Batches partitions for multi-value filters:

```yaml
partition_router:
  type: GroupingPartitionRouter
  group_size: 10
  underlying_partition_router:
    type: ListPartitionRouter
    values: "{{ config['ids'] }}"
    cursor_field: "id"
  deduplicate: true
```

---

## 12. Dynamic Streams (Templates)

```yaml
dynamic_streams:
  - type: DynamicDeclarativeStream
    stream_template:
      type: DeclarativeStream
      name: "{{ components_values.name }}"
      retriever:
        type: SimpleRetriever
        requester:
          type: HttpRequester
          path: "/{{ components_values.endpoint }}"
        record_selector:
          type: RecordSelector
          extractor:
            type: DpathExtractor
            field_path: ["data"]
    stream_template_config: ...
```

---

## 13. Custom Components

When built-in components are insufficient, implement Python `@dataclass` classes.

### Requirements

1. Must be a `@dataclass`
2. Must implement the target interface (e.g., `PaginationStrategy`, `RecordTransformation`)
3. Must include `parameters: InitVar[Mapping[str, Any]]`

### YAML Registration

```yaml
transformations:
  - type: CustomRecordTransformation
    class_name: "source_myapi.transformations.MyTransformation"
    my_param: "value"
```

### Available Custom Types

`CustomAuthenticator`, `CustomPaginationStrategy`, `CustomRecordExtractor`, `CustomRecordTransformation`, `CustomRecordFilter`, `CustomBackoffStrategy`, `CustomErrorHandler`, `CustomDecoder`, `CustomPartitionRouter`, `CustomRetriever`

**Security:** Custom components require `AIRBYTE_ENABLE_UNSAFE_CODE=true`.

---

## 14. Best Practices

### Manifest Organization
- Use `definitions` extensively to avoid duplication
- Reference via `$ref: "#/definitions/<name>"`
- Use `$parameters` for passing values to child components

### Pagination
- Use the largest `page_size` the API supports
- Always set a `stop_condition` for cursor pagination
- Test with small page sizes first

### Incremental Sync
- Always configure `lookback_window` for late-arriving data
- Use `step` + `cursor_granularity` for large date ranges
- `is_client_side_incremental: true` only when API can't filter server-side
- `is_data_feed: true` for newest-first APIs

### Error Handling
- Use `CompositeErrorHandler` for production connectors
- Handle 429 with `WaitTimeFromHeader` reading `Retry-After`
- Classify errors: `config_error` for auth, `transient_error` for retryable

### Schema
- Prefer `InlineSchemaLoader` for manifest-only connectors
- Always declare `primary_key` for deduplication

### Substreams
- Add parent ID to child records via `AddFields`
- Use `incremental_dependency: true` to avoid full parent re-reads
- Per-partition state by default; falls back to global after 10,000 partitions

### Transformations
- Keep minimal — complex logic belongs in the warehouse
- Use `RemoveFields` for PII
- Use `AddFields` with `hash('md5')` for pseudonymization

---

## 15. Complete Example

See `MANIFEST_EXAMPLE.yaml` for a non-trivial multi-stream connector with:
- Shared definitions (requester, selector, paginator, cursor)
- Parent/child streams (projects → tasks → comments)
- POST-based analytics stream with ListPartitionRouter
- Incremental sync with lookback
- Composite error handling with rate-limit support
- Inline schemas
